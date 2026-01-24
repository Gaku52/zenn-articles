---
title: "実践ケーススタディ（後編） - パフォーマンス最適化とデプロイ"
---

# 実践ケーススタディ（後編） - パフォーマンス最適化とデプロイ

前編で構築したブログAPIシステムに、キャッシング、全文検索、レート制限などの高度な機能を追加します。

## コメント機能

### コメントルート

```typescript
// src/routes/comments.ts
import express from 'express';
import { z } from 'zod';
import { prisma } from '../lib/prisma';
import { authenticate } from '../middleware/auth';

const router = express.Router();

const createCommentSchema = z.object({
  content: z.string().min(1).max(1000),
});

// コメント作成
router.post('/posts/:postId/comments', authenticate, async (req, res) => {
  try {
    const { content } = createCommentSchema.parse(req.body);

    const comment = await prisma.comment.create({
      data: {
        content,
        postId: req.params.postId,
        authorId: req.user!.userId,
      },
      include: {
        author: {
          select: { id: true, name: true },
        },
      },
    });

    res.status(201).json(comment);
  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({ error: error.errors });
    }
    res.status(500).json({ error: 'Internal server error' });
  }
});

export default router;
```

## いいね機能

### いいねルート

```typescript
// src/routes/likes.ts
import express from 'express';
import { prisma } from '../lib/prisma';
import { authenticate } from '../middleware/auth';

const router = express.Router();

// いいねをトグル
router.post('/posts/:postId/likes', authenticate, async (req, res) => {
  try {
    const { postId } = req.params;
    const userId = req.user!.userId;

    // 既存のいいねをチェック
    const existingLike = await prisma.like.findUnique({
      where: {
        postId_userId: {
          postId,
          userId,
        },
      },
    });

    if (existingLike) {
      // いいねを削除
      await prisma.like.delete({
        where: { id: existingLike.id },
      });

      res.json({ liked: false });
    } else {
      // いいねを追加
      await prisma.like.create({
        data: {
          postId,
          userId,
        },
      });

      res.json({ liked: true });
    }
  } catch (error) {
    res.status(500).json({ error: 'Internal server error' });
  }
});

export default router;
```

## Redis キャッシング

### Redis クライアント

```typescript
// src/lib/redis.ts
import { createClient } from 'redis';

export const redis = createClient({
  url: process.env.REDIS_URL || 'redis://localhost:6379',
});

redis.on('error', (err) => console.error('Redis Error:', err));
redis.on('connect', () => console.log('Redis connected'));

export async function connectRedis() {
  await redis.connect();
}
```

### キャッシュミドルウェア

```typescript
// src/middleware/cache.ts
import { Request, Response, NextFunction } from 'express';
import { redis } from '../lib/redis';

export function cacheMiddleware(ttl: number = 300) {
  return async (req: Request, res: Response, next: NextFunction) => {
    // GETリクエストのみキャッシュ
    if (req.method !== 'GET') {
      return next();
    }

    const key = `cache:${req.originalUrl}`;

    try {
      const cached = await redis.get(key);

      if (cached) {
        return res.json(JSON.parse(cached));
      }

      // レスポンスをキャッシュ
      const originalSend = res.json.bind(res);
      res.json = function (data: any) {
        redis.setEx(key, ttl, JSON.stringify(data));
        return originalSend(data);
      };

      next();
    } catch (error) {
      next();
    }
  };
}
```

### キャッシュ適用

```typescript
// src/routes/posts.ts
import { cacheMiddleware } from '../middleware/cache';

// 記事一覧（5分キャッシュ）
router.get('/', cacheMiddleware(300), async (req, res) => {
  // ... 既存のコード
});

// 記事詳細（10分キャッシュ）
router.get('/:id', cacheMiddleware(600), async (req, res) => {
  // ... 既存のコード
});
```

## レート制限

### レート制限ミドルウェア

```typescript
// src/middleware/rateLimit.ts
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import { redis } from '../lib/redis';

// 一般的なエンドポイント
export const apiLimiter = rateLimit({
  store: new RedisStore({
    client: redis,
    prefix: 'rate-limit:',
  }),
  windowMs: 15 * 60 * 1000, // 15分
  max: 100,
  message: 'Too many requests',
});

// 認証エンドポイント（厳しめ）
export const authLimiter = rateLimit({
  store: new RedisStore({
    client: redis,
    prefix: 'rate-limit:auth:',
  }),
  windowMs: 15 * 60 * 1000,
  max: 5,
  message: 'Too many authentication attempts',
});
```

### レート制限適用

```typescript
// src/app.ts
import { apiLimiter, authLimiter } from './middleware/rateLimit';

app.use('/api/', apiLimiter);
app.use('/auth/', authLimiter);
```

## 全文検索

### PostgreSQL 全文検索

```typescript
// src/routes/search.ts
import express from 'express';
import { prisma } from '../lib/prisma';

const router = express.Router();

router.get('/search', async (req, res) => {
  try {
    const query = req.query.q as string;

    if (!query) {
      return res.status(400).json({ error: 'Query parameter required' });
    }

    const posts = await prisma.$queryRaw`
      SELECT
        id,
        title,
        content,
        ts_rank(to_tsvector('english', title || ' ' || content), plainto_tsquery('english', ${query})) as rank
      FROM "Post"
      WHERE to_tsvector('english', title || ' ' || content) @@ plainto_tsquery('english', ${query})
        AND published = true
      ORDER BY rank DESC
      LIMIT 20
    `;

    res.json(posts);
  } catch (error) {
    res.status(500).json({ error: 'Internal server error' });
  }
});

export default router;
```

## パフォーマンス最適化

### データローダー（N+1問題の解決）

```typescript
// src/lib/dataloader.ts
import DataLoader from 'dataloader';
import { prisma } from './prisma';

export const userLoader = new DataLoader(async (userIds: string[]) => {
  const users = await prisma.user.findMany({
    where: { id: { in: userIds } },
  });

  const userMap = new Map(users.map((u) => [u.id, u]));

  return userIds.map((id) => userMap.get(id) || null);
});
```

### クエリ最適化

```typescript
// 悪い例: N+1クエリ
const posts = await prisma.post.findMany();
for (const post of posts) {
  post.author = await prisma.user.findUnique({
    where: { id: post.authorId },
  });
}

// 良い例: 一度にロード
const posts = await prisma.post.findMany({
  include: {
    author: {
      select: { id: true, name: true, email: true },
    },
  },
});
```

## Docker化

### Dockerfile

```dockerfile
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
COPY prisma ./prisma

RUN npm ci

COPY . .

RUN npx prisma generate
RUN npm run build

FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
COPY prisma ./prisma

RUN npm ci --only=production

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules/.prisma ./node_modules/.prisma

EXPOSE 3000

CMD ["node", "dist/server.js"]
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - '3000:3000'
    environment:
      - DATABASE_URL=postgresql://postgres:password@db:5432/blog
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=your-secret-key
      - NODE_ENV=production
    depends_on:
      - db
      - redis

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=blog
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - '5432:5432'

  redis:
    image: redis:7-alpine
    ports:
      - '6379:6379'

volumes:
  postgres_data:
```

## CI/CD パイプライン

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '20'
      - run: npm ci
      - run: npm test

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        run: |
          # デプロイスクリプト
          echo "Deploying to production..."
```

## メインアプリケーション

### app.ts

```typescript
// src/app.ts
import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import authRoutes from './routes/auth';
import postRoutes from './routes/posts';
import commentRoutes from './routes/comments';
import likeRoutes from './routes/likes';
import searchRoutes from './routes/search';
import { apiLimiter, authLimiter } from './middleware/rateLimit';
import { errorHandler } from './middleware/errorHandler';

const app = express();

// ミドルウェア
app.use(helmet());
app.use(cors());
app.use(express.json());

// レート制限
app.use('/api/', apiLimiter);
app.use('/auth/', authLimiter);

// ルート
app.use('/auth', authRoutes);
app.use('/api/posts', postRoutes);
app.use('/api', commentRoutes);
app.use('/api', likeRoutes);
app.use('/api', searchRoutes);

// ヘルスチェック
app.get('/health', (req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});

// エラーハンドラー
app.use(errorHandler);

export { app };
```

### server.ts

```typescript
// src/server.ts
import { app } from './app';
import { prisma } from './lib/prisma';
import { connectRedis } from './lib/redis';

const PORT = process.env.PORT || 3000;

async function start() {
  try {
    // Redis接続
    await connectRedis();

    // サーバー起動
    app.listen(PORT, () => {
      console.log(`Server running on port ${PORT}`);
    });

    // グレースフルシャットダウン
    process.on('SIGTERM', async () => {
      console.log('SIGTERM received, shutting down gracefully');
      await prisma.$disconnect();
      process.exit(0);
    });
  } catch (error) {
    console.error('Failed to start server:', error);
    process.exit(1);
  }
}

start();
```

## 統合テスト

### E2Eテスト

```typescript
// src/__tests__/e2e.test.ts
import request from 'supertest';
import { app } from '../app';
import { prisma } from '../lib/prisma';

describe('Blog API E2E', () => {
  let authToken: string;
  let postId: string;

  beforeAll(async () => {
    await prisma.user.deleteMany();
    await prisma.post.deleteMany();
  });

  it('should complete full user journey', async () => {
    // 1. ユーザー登録
    const registerRes = await request(app)
      .post('/auth/register')
      .send({
        email: 'test@example.com',
        password: 'Password123',
        name: 'Test User',
      })
      .expect(201);

    authToken = registerRes.body.token;

    // 2. 記事作成
    const createPostRes = await request(app)
      .post('/api/posts')
      .set('Authorization', `Bearer ${authToken}`)
      .send({
        title: 'My First Post',
        content: 'This is my first blog post!',
        published: true,
        tags: ['test', 'intro'],
      })
      .expect(201);

    postId = createPostRes.body.id;

    // 3. 記事一覧取得
    const listRes = await request(app).get('/api/posts').expect(200);

    expect(listRes.body.data).toHaveLength(1);

    // 4. コメント追加
    await request(app)
      .post(`/api/posts/${postId}/comments`)
      .set('Authorization', `Bearer ${authToken}`)
      .send({
        content: 'Great post!',
      })
      .expect(201);

    // 5. いいね
    await request(app)
      .post(`/api/posts/${postId}/likes`)
      .set('Authorization', `Bearer ${authToken}`)
      .expect(200);

    // 6. 記事詳細取得
    const detailRes = await request(app).get(`/api/posts/${postId}`).expect(200);

    expect(detailRes.body.comments).toHaveLength(1);
    expect(detailRes.body._count.likes).toBe(1);
  });
});
```

## 本番運用チェックリスト

### セキュリティ

- ✅ JWT_SECRETを強力なランダム値に設定
- ✅ HTTPS通信のみ許可
- ✅ helmetでセキュリティヘッダー設定
- ✅ レート制限を適用
- ✅ 入力バリデーションを徹底

### パフォーマンス

- ✅ Redisキャッシングを有効化
- ✅ データベースインデックスを適切に設定
- ✅ N+1クエリを解消
- ✅ ページネーションを実装

### 監視

- ✅ ヘルスチェックエンドポイント
- ✅ ログ出力を構造化
- ✅ エラー通知を設定

### スケーラビリティ

- ✅ ステートレスな設計
- ✅ Redisでセッション管理
- ✅ 水平スケーリング可能

## まとめ

本書で学んだ内容を統合し、実践的なブログAPIシステムを構築しました:

### 実装した機能

- ✅ JWT認証システム
- ✅ 記事CRUD操作
- ✅ コメント・いいね機能
- ✅ タグ付け
- ✅ 全文検索
- ✅ ページネーション
- ✅ Redisキャッシング
- ✅ レート制限
- ✅ 統合テスト
- ✅ Docker化
- ✅ CI/CDパイプライン

### 技術スタック

- **Runtime**: Node.js 20
- **Framework**: Express
- **ORM**: Prisma
- **Database**: PostgreSQL
- **Cache**: Redis
- **Testing**: Jest, Supertest
- **Validation**: Zod
- **Authentication**: JWT
- **Containerization**: Docker

### 次のステップ

このケーススタディを基に、以下のような拡張が可能です:

1. **GraphQLAPI**: RESTからGraphQLへ移行
2. **リアルタイム通知**: WebSocketで通知機能
3. **画像アップロード**: S3統合
4. **メール送信**: SendGrid統合
5. **管理画面**: React + Next.js
6. **マイクロサービス化**: サービス分割

## 終わりに

本書『Node.js開発完全ガイド 2026』を最後までお読みいただき、ありがとうございました。

本書で学んだ知識を活かし、実践的なプロダクション環境で活躍されることを願っています。

Happy Coding! 🚀
