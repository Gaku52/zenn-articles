---
title: "実践ケーススタディ（前編） - ブログAPIシステムの構築"
---

# 実践ケーススタディ（前編） - ブログAPIシステムの構築

本章と次章では、これまで学んだ全ての知識を統合し、実践的なブログAPIシステムを0から構築します。

## プロジェクト概要

### 要件

- ユーザー認証（JWT）
- 記事のCRUD操作
- コメント機能
- タグ付け
- いいね機能
- 全文検索
- ページネーション
- レート制限
- キャッシング

### 技術スタック

- **Runtime**: Node.js 20
- **Framework**: Express
- **ORM**: Prisma
- **Database**: PostgreSQL
- **Cache**: Redis
- **Testing**: Jest, Supertest
- **Validation**: Zod
- **Authentication**: JWT

## プロジェクトセットアップ

### 初期化

```bash
mkdir blog-api
cd blog-api
npm init -y
npm install express prisma @prisma/client zod bcryptjs jsonwebtoken redis ioredis
npm install -D typescript @types/node @types/express @types/bcryptjs @types/jsonwebtoken ts-node-dev jest @types/jest supertest @types/supertest
```

### TypeScript 設定

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

## データベース設計

### Prisma Schema

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  password  String
  name      String
  bio       String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  posts     Post[]
  comments  Comment[]
  likes     Like[]
}

model Post {
  id        String   @id @default(uuid())
  title     String
  content   String
  published Boolean  @default(false)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  authorId  String
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)

  tags      Tag[]
  comments  Comment[]
  likes     Like[]

  @@index([authorId])
  @@index([published])
}

model Comment {
  id        String   @id @default(uuid())
  content   String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  postId    String
  post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)

  authorId  String
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)

  @@index([postId])
  @@index([authorId])
}

model Tag {
  id    String @id @default(uuid())
  name  String @unique
  posts Post[]
}

model Like {
  id        String   @id @default(uuid())
  createdAt DateTime @default(now())

  postId    String
  post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)

  userId    String
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([postId, userId])
  @@index([postId])
  @@index([userId])
}
```

### マイグレーション

```bash
npx prisma migrate dev --name init
npx prisma generate
```

## 認証システム

### JWT ユーティリティ

```typescript
// src/utils/jwt.ts
import jwt from 'jsonwebtoken';

const JWT_SECRET = process.env.JWT_SECRET || 'your-secret-key';

export interface JwtPayload {
  userId: string;
  email: string;
}

export function generateToken(payload: JwtPayload): string {
  return jwt.sign(payload, JWT_SECRET, { expiresIn: '7d' });
}

export function verifyToken(token: string): JwtPayload {
  return jwt.verify(token, JWT_SECRET) as JwtPayload;
}
```

### 認証ミドルウェア

```typescript
// src/middleware/auth.ts
import { Request, Response, NextFunction } from 'express';
import { verifyToken, JwtPayload } from '../utils/jwt';

declare global {
  namespace Express {
    interface Request {
      user?: JwtPayload;
    }
  }
}

export function authenticate(
  req: Request,
  res: Response,
  next: NextFunction
) {
  const authHeader = req.headers.authorization;

  if (!authHeader) {
    return res.status(401).json({ error: 'No token provided' });
  }

  const token = authHeader.replace('Bearer ', '');

  try {
    const payload = verifyToken(token);
    req.user = payload;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
}
```

### 認証エンドポイント

```typescript
// src/routes/auth.ts
import express from 'express';
import bcrypt from 'bcryptjs';
import { z } from 'zod';
import { prisma } from '../lib/prisma';
import { generateToken } from '../utils/jwt';

const router = express.Router();

// バリデーションスキーマ
const registerSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  name: z.string().min(1),
});

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string(),
});

// 登録
router.post('/register', async (req, res) => {
  try {
    const { email, password, name } = registerSchema.parse(req.body);

    // 既存ユーザーチェック
    const existing = await prisma.user.findUnique({ where: { email } });
    if (existing) {
      return res.status(409).json({ error: 'Email already exists' });
    }

    // パスワードハッシュ化
    const hashedPassword = await bcrypt.hash(password, 10);

    // ユーザー作成
    const user = await prisma.user.create({
      data: {
        email,
        password: hashedPassword,
        name,
      },
      select: {
        id: true,
        email: true,
        name: true,
        createdAt: true,
      },
    });

    // トークン生成
    const token = generateToken({
      userId: user.id,
      email: user.email,
    });

    res.status(201).json({ user, token });
  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({ error: error.errors });
    }
    res.status(500).json({ error: 'Internal server error' });
  }
});

// ログイン
router.post('/login', async (req, res) => {
  try {
    const { email, password } = loginSchema.parse(req.body);

    // ユーザー検索
    const user = await prisma.user.findUnique({ where: { email } });

    if (!user || !(await bcrypt.compare(password, user.password))) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    // トークン生成
    const token = generateToken({
      userId: user.id,
      email: user.email,
    });

    res.json({
      user: {
        id: user.id,
        email: user.email,
        name: user.name,
      },
      token,
    });
  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({ error: error.errors });
    }
    res.status(500).json({ error: 'Internal server error' });
  }
});

export default router;
```

## 記事 CRUD

### バリデーションスキーマ

```typescript
// src/schemas/post.ts
import { z } from 'zod';

export const createPostSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(1),
  published: z.boolean().optional(),
  tags: z.array(z.string()).optional(),
});

export const updatePostSchema = createPostSchema.partial();
```

### 記事ルート

```typescript
// src/routes/posts.ts
import express from 'express';
import { prisma } from '../lib/prisma';
import { authenticate } from '../middleware/auth';
import { createPostSchema, updatePostSchema } from '../schemas/post';

const router = express.Router();

// 記事一覧取得（ページネーション）
router.get('/', async (req, res) => {
  try {
    const page = parseInt(req.query.page as string) || 1;
    const limit = parseInt(req.query.limit as string) || 10;
    const skip = (page - 1) * limit;

    const [posts, total] = await Promise.all([
      prisma.post.findMany({
        where: { published: true },
        include: {
          author: {
            select: { id: true, name: true, email: true },
          },
          tags: true,
          _count: {
            select: { comments: true, likes: true },
          },
        },
        orderBy: { createdAt: 'desc' },
        skip,
        take: limit,
      }),
      prisma.post.count({ where: { published: true } }),
    ]);

    res.json({
      data: posts,
      meta: {
        page,
        limit,
        total,
        totalPages: Math.ceil(total / limit),
      },
    });
  } catch (error) {
    res.status(500).json({ error: 'Internal server error' });
  }
});

// 記事詳細取得
router.get('/:id', async (req, res) => {
  try {
    const post = await prisma.post.findUnique({
      where: { id: req.params.id },
      include: {
        author: {
          select: { id: true, name: true, email: true, bio: true },
        },
        tags: true,
        comments: {
          include: {
            author: {
              select: { id: true, name: true },
            },
          },
          orderBy: { createdAt: 'desc' },
        },
        _count: {
          select: { likes: true },
        },
      },
    });

    if (!post) {
      return res.status(404).json({ error: 'Post not found' });
    }

    res.json(post);
  } catch (error) {
    res.status(500).json({ error: 'Internal server error' });
  }
});

// 記事作成
router.post('/', authenticate, async (req, res) => {
  try {
    const data = createPostSchema.parse(req.body);

    const post = await prisma.post.create({
      data: {
        title: data.title,
        content: data.content,
        published: data.published ?? false,
        authorId: req.user!.userId,
        tags: data.tags
          ? {
              connectOrCreate: data.tags.map((tag) => ({
                where: { name: tag },
                create: { name: tag },
              })),
            }
          : undefined,
      },
      include: {
        author: {
          select: { id: true, name: true, email: true },
        },
        tags: true,
      },
    });

    res.status(201).json(post);
  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({ error: error.errors });
    }
    res.status(500).json({ error: 'Internal server error' });
  }
});

// 記事更新
router.patch('/:id', authenticate, async (req, res) => {
  try {
    const data = updatePostSchema.parse(req.body);

    // 所有者チェック
    const existingPost = await prisma.post.findUnique({
      where: { id: req.params.id },
    });

    if (!existingPost) {
      return res.status(404).json({ error: 'Post not found' });
    }

    if (existingPost.authorId !== req.user!.userId) {
      return res.status(403).json({ error: 'Forbidden' });
    }

    const post = await prisma.post.update({
      where: { id: req.params.id },
      data: {
        title: data.title,
        content: data.content,
        published: data.published,
        tags: data.tags
          ? {
              set: [],
              connectOrCreate: data.tags.map((tag) => ({
                where: { name: tag },
                create: { name: tag },
              })),
            }
          : undefined,
      },
      include: {
        author: {
          select: { id: true, name: true, email: true },
        },
        tags: true,
      },
    });

    res.json(post);
  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({ error: error.errors });
    }
    res.status(500).json({ error: 'Internal server error' });
  }
});

// 記事削除
router.delete('/:id', authenticate, async (req, res) => {
  try {
    const existingPost = await prisma.post.findUnique({
      where: { id: req.params.id },
    });

    if (!existingPost) {
      return res.status(404).json({ error: 'Post not found' });
    }

    if (existingPost.authorId !== req.user!.userId) {
      return res.status(403).json({ error: 'Forbidden' });
    }

    await prisma.post.delete({
      where: { id: req.params.id },
    });

    res.status(204).send();
  } catch (error) {
    res.status(500).json({ error: 'Internal server error' });
  }
});

export default router;
```

## テスト

### 認証テスト

```typescript
// src/routes/__tests__/auth.test.ts
import request from 'supertest';
import { app } from '../../app';
import { prisma } from '../../lib/prisma';

describe('Auth API', () => {
  beforeEach(async () => {
    await prisma.user.deleteMany();
  });

  describe('POST /auth/register', () => {
    it('should register a new user', async () => {
      const response = await request(app)
        .post('/auth/register')
        .send({
          email: 'test@example.com',
          password: 'Password123',
          name: 'Test User',
        })
        .expect(201);

      expect(response.body).toHaveProperty('user');
      expect(response.body).toHaveProperty('token');
      expect(response.body.user.email).toBe('test@example.com');
    });

    it('should reject duplicate email', async () => {
      await request(app).post('/auth/register').send({
        email: 'test@example.com',
        password: 'Password123',
        name: 'Test User',
      });

      await request(app)
        .post('/auth/register')
        .send({
          email: 'test@example.com',
          password: 'Password123',
          name: 'Another User',
        })
        .expect(409);
    });
  });

  describe('POST /auth/login', () => {
    beforeEach(async () => {
      await request(app).post('/auth/register').send({
        email: 'test@example.com',
        password: 'Password123',
        name: 'Test User',
      });
    });

    it('should login with valid credentials', async () => {
      const response = await request(app)
        .post('/auth/login')
        .send({
          email: 'test@example.com',
          password: 'Password123',
        })
        .expect(200);

      expect(response.body).toHaveProperty('token');
    });

    it('should reject invalid credentials', async () => {
      await request(app)
        .post('/auth/login')
        .send({
          email: 'test@example.com',
          password: 'wrongpassword',
        })
        .expect(401);
    });
  });
});
```

## まとめ

前編では以下を実装しました:

- ✅ プロジェクトセットアップ
- ✅ データベース設計（Prisma）
- ✅ JWT認証システム
- ✅ 記事のCRUD操作
- ✅ バリデーション（Zod）
- ✅ ページネーション
- ✅ 統合テスト

次の章（後編）では、以下を実装します:

- コメント機能
- いいね機能
- 全文検索
- Redisキャッシング
- レート制限
- パフォーマンス最適化
- デプロイ準備
