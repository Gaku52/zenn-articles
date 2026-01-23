---
title: "Prisma完全マスター"
---

# Prisma完全マスター

Prismaは、次世代のTypeScript ORMとして、型安全性と開発体験を極限まで高めたツールです。この章では、Prismaの高度な機能、最適化手法、実践的なパターンを想定される効果とともに解説します。

## Prismaの基礎

Prismaを効果的に使用することで、以下の想定効果が得られます:

**想定される効果:**
- 型エラー検出: **開発時に100%** (実行時エラー削減)
- クエリ応答時間: **850ms → 12ms** (-99%)
- N+1問題解消: **150クエリ → 3クエリ** (-98%)
- 開発速度: **40%向上** (自動補完と型安全性)

## スキーマ設計のベストプラクティス

### 基本的なスキーマ定義

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
  previewFeatures = ["fullTextSearch", "jsonProtocol"]
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        Int       @id @default(autoincrement())
  email     String    @unique @db.VarChar(255)
  username  String    @db.VarChar(50)
  createdAt DateTime  @default(now()) @map("created_at")
  updatedAt DateTime  @updatedAt @map("updated_at")

  posts     Post[]
  profile   Profile?
  comments  Comment[]

  @@index([email])
  @@index([username])
  @@map("users")
}

model Profile {
  id     Int     @id @default(autoincrement())
  userId Int     @unique @map("user_id")
  user   User    @relation(fields: [userId], references: [id], onDelete: Cascade)
  bio    String? @db.Text
  avatar String? @db.VarChar(500)

  @@map("profiles")
}

model Post {
  id          Int       @id @default(autoincrement())
  title       String    @db.VarChar(255)
  content     String    @db.Text
  published   Boolean   @default(false)
  publishedAt DateTime? @map("published_at")
  userId      Int       @map("user_id")
  user        User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  createdAt   DateTime  @default(now()) @map("created_at")
  updatedAt   DateTime  @updatedAt @map("updated_at")

  comments    Comment[]
  tags        TagOnPost[]

  @@index([userId, createdAt])
  @@index([publishedAt])
  @@map("posts")
}

model Comment {
  id        Int      @id @default(autoincrement())
  content   String   @db.Text
  postId    Int      @map("post_id")
  post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  userId    Int      @map("user_id")
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  createdAt DateTime @default(now()) @map("created_at")

  @@index([postId])
  @@index([userId])
  @@map("comments")
}

model Tag {
  id    Int         @id @default(autoincrement())
  name  String      @unique @db.VarChar(50)
  posts TagOnPost[]

  @@map("tags")
}

model TagOnPost {
  postId Int  @map("post_id")
  tagId  Int  @map("tag_id")
  post   Post @relation(fields: [postId], references: [id], onDelete: Cascade)
  tag    Tag  @relation(fields: [tagId], references: [id], onDelete: Cascade)

  @@id([postId, tagId])
  @@map("tags_on_posts")
}
```

### データ型の最適化

```prisma
model Product {
  id          Int      @id @default(autoincrement())
  name        String   @db.VarChar(255)  // 可変長文字列
  description String   @db.Text           // 長文
  price       Decimal  @db.Decimal(10, 2) // 価格(精度指定)
  stock       Int      @db.Integer        // 整数
  isActive    Boolean  @default(true)     // ブール値
  attributes  Json                        // JSON型
  createdAt   DateTime @default(now()) @map("created_at")

  @@map("products")
}
```

## クエリ最適化

### N+1問題の解消

```typescript
// ❌ N+1問題(1 + N回のクエリ)
const users = await prisma.user.findMany()

for (const user of users) {
  const posts = await prisma.post.findMany({
    where: { userId: user.id }
  })
  console.log(`${user.username}: ${posts.length} posts`)
}
// 100ユーザー → 101クエリ実行
// 応答時間: 15,000ms

// ✅ Eager Loading(1回のクエリ)
const users = await prisma.user.findMany({
  include: {
    posts: true
  }
})

users.forEach(user => {
  console.log(`${user.username}: ${user.posts.length} posts`)
})
// 100ユーザー → 1クエリ実行
// 応答時間: 120ms (-99%)
```

### select vs include

```typescript
// ✅ 必要なカラムのみ取得(select)
const users = await prisma.user.findMany({
  select: {
    id: true,
    username: true,
    email: true
  }
})
// データ転送: 1.5KB

// ❌ すべてのカラムを取得
const users = await prisma.user.findMany()
// データ転送: 5KB

// ✅ リレーションと特定のカラム
const users = await prisma.user.findMany({
  select: {
    id: true,
    username: true,
    posts: {
      select: {
        id: true,
        title: true,
        publishedAt: true
      }
    }
  }
})
```

### where句の最適化

```typescript
// ✅ インデックスを活用したクエリ
const posts = await prisma.post.findMany({
  where: {
    userId: 123,
    publishedAt: {
      gte: new Date('2025-01-01')
    }
  },
  orderBy: {
    createdAt: 'desc'
  }
})
// Index Scan using idx_posts_user_created
// クエリ時間: 12ms

// ❌ インデックスが効かないクエリ
const posts = await prisma.post.findMany({
  where: {
    title: {
      contains: 'database'  // LIKE '%database%'
    }
  }
})
// Seq Scan on posts
// クエリ時間: 850ms
```

### 集計とカウント

```typescript
// ❌ すべてのデータを取得してカウント
const users = await prisma.user.findMany({
  include: { posts: true }
})
const totalPosts = users.reduce((sum, user) => sum + user.posts.length, 0)
// データ転送: 大量、メモリ使用: 高

// ✅ データベース側でカウント
const users = await prisma.user.findMany({
  select: {
    id: true,
    username: true,
    _count: {
      select: { posts: true }
    }
  }
})
// データ転送: 最小限
// クエリ時間: 45ms → 8ms (-82%)
```

## トランザクション

### インタラクティブトランザクション

```typescript
// ✅ インタラクティブトランザクション
const result = await prisma.$transaction(async (tx) => {
  // 1. ユーザー作成
  const user = await tx.user.create({
    data: {
      email: 'user@example.com',
      username: 'newuser'
    }
  })

  // 2. プロフィール作成
  const profile = await tx.profile.create({
    data: {
      userId: user.id,
      bio: 'Hello, world!'
    }
  })

  // 3. 初期投稿作成
  const post = await tx.post.create({
    data: {
      title: 'First Post',
      content: 'This is my first post!',
      userId: user.id,
      published: true,
      publishedAt: new Date()
    }
  })

  return { user, profile, post }
})

// すべて成功 or すべてロールバック
```

### バッチトランザクション

```typescript
// ✅ 複数の操作を1つのトランザクションで実行
const [user, posts] = await prisma.$transaction([
  prisma.user.create({
    data: { email: 'user@example.com', username: 'newuser' }
  }),
  prisma.post.createMany({
    data: [
      { title: 'Post 1', content: 'Content 1', userId: 1 },
      { title: 'Post 2', content: 'Content 2', userId: 1 }
    ]
  })
])
```

### 楽観的ロック

```typescript
// ✅ バージョンカラムで楽観的ロック
async function updateProduct(id: number, newPrice: number) {
  const product = await prisma.product.findUnique({
    where: { id }
  })

  if (!product) throw new Error('Product not found')

  try {
    const updated = await prisma.product.update({
      where: {
        id,
        version: product.version  // 同じバージョンの場合のみ更新
      },
      data: {
        price: newPrice,
        version: { increment: 1 }  // バージョンをインクリメント
      }
    })
    return updated
  } catch (error) {
    throw new Error('Product was updated by another user')
  }
}
```

**スキーマ定義:**

```prisma
model Product {
  id      Int     @id @default(autoincrement())
  name    String  @db.VarChar(255)
  price   Decimal @db.Decimal(10, 2)
  version Int     @default(1)  // バージョンカラム

  @@map("products")
}
```

## 高度なクエリパターン

### カーソルページネーション

```typescript
// ✅ カーソルベースのページネーション
async function getPosts(cursor?: number, limit: number = 20) {
  const posts = await prisma.post.findMany({
    take: limit,
    skip: cursor ? 1 : 0,  // カーソルの次から取得
    cursor: cursor ? { id: cursor } : undefined,
    orderBy: {
      createdAt: 'desc'
    },
    select: {
      id: true,
      title: true,
      publishedAt: true,
      user: {
        select: {
          username: true
        }
      }
    }
  })

  return {
    posts,
    nextCursor: posts.length === limit ? posts[posts.length - 1].id : null
  }
}

// 使用例
const page1 = await getPosts()
const page2 = await getPosts(page1.nextCursor)
```

### 全文検索

```typescript
// PostgreSQLの全文検索
const posts = await prisma.post.findMany({
  where: {
    OR: [
      {
        title: {
          search: 'database optimization'
        }
      },
      {
        content: {
          search: 'database optimization'
        }
      }
    ]
  }
})
```

**スキーマ設定:**

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["fullTextSearch"]
}
```

### JSON操作

```typescript
// ✅ JSON型の操作
const products = await prisma.product.findMany({
  where: {
    attributes: {
      path: ['color'],
      equals: 'red'
    }
  }
})

// JSON配列のフィルター
const products = await prisma.product.findMany({
  where: {
    attributes: {
      path: ['tags'],
      array_contains: 'premium'
    }
  }
})
```

## パフォーマンス最適化

### コネクションプーリング

```typescript
// prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// .env
DATABASE_URL="postgresql://user:password@localhost:5432/mydb?connection_limit=10&pool_timeout=20"
```

**推奨設定:**

- **開発環境**: connection_limit=5
- **本番環境**: connection_limit=10-20 (サーバーリソースに応じて)

### クエリバッチング

```typescript
// ✅ findManyを使ってバッチ処理
const userIds = [1, 2, 3, 4, 5]

const users = await prisma.user.findMany({
  where: {
    id: {
      in: userIds
    }
  }
})
// 1回のクエリで複数のユーザーを取得

// ❌ 個別にクエリ
for (const id of userIds) {
  const user = await prisma.user.findUnique({ where: { id } })
}
// 5回のクエリ
```

### クエリキャッシング

```typescript
// ✅ Redisでクエリ結果をキャッシュ
import Redis from 'ioredis'

const redis = new Redis()

async function getCachedUser(id: number) {
  // キャッシュチェック
  const cached = await redis.get(`user:${id}`)
  if (cached) {
    return JSON.parse(cached)
  }

  // データベースから取得
  const user = await prisma.user.findUnique({
    where: { id },
    include: { profile: true }
  })

  // キャッシュに保存(5分間)
  await redis.set(`user:${id}`, JSON.stringify(user), 'EX', 300)

  return user
}
```

## マイグレーション実践

### 開発環境ワークフロー

```bash
# 1. スキーマを編集
# prisma/schema.prisma

# 2. マイグレーション作成と適用
npx prisma migrate dev --name add_user_profile

# 3. クライアント生成
npx prisma generate

# 4. データベースリセット(開発時のみ)
npx prisma migrate reset
```

### 本番環境デプロイ

```bash
# 1. マイグレーション適用
npx prisma migrate deploy

# 2. クライアント生成
npx prisma generate

# 3. 検証
npx prisma migrate status
```

### カスタムマイグレーション

```bash
# マイグレーション作成(適用なし)
npx prisma migrate dev --create-only --name add_full_text_search
```

```sql
-- prisma/migrations/xxx_add_full_text_search/migration.sql

-- 全文検索インデックスの作成
CREATE INDEX idx_posts_title_search
ON posts USING GIN(to_tsvector('english', title));

CREATE INDEX idx_posts_content_search
ON posts USING GIN(to_tsvector('english', content));

-- updated_atトリガー
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_posts_updated_at
BEFORE UPDATE ON posts
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();
```

## 実装パターン

### パターン1: Repository パターン

```typescript
// repositories/UserRepository.ts
import { PrismaClient, User, Prisma } from '@prisma/client'

export class UserRepository {
  constructor(private prisma: PrismaClient) {}

  async findById(id: number): Promise<User | null> {
    return this.prisma.user.findUnique({
      where: { id },
      include: {
        profile: true,
        posts: {
          where: { published: true },
          orderBy: { createdAt: 'desc' },
          take: 10
        }
      }
    })
  }

  async create(data: Prisma.UserCreateInput): Promise<User> {
    return this.prisma.user.create({ data })
  }

  async update(id: number, data: Prisma.UserUpdateInput): Promise<User> {
    return this.prisma.user.update({
      where: { id },
      data
    })
  }

  async delete(id: number): Promise<void> {
    await this.prisma.user.delete({ where: { id } })
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.prisma.user.findUnique({
      where: { email }
    })
  }

  async searchUsers(query: string, limit: number = 20): Promise<User[]> {
    return this.prisma.user.findMany({
      where: {
        OR: [
          { username: { contains: query, mode: 'insensitive' } },
          { email: { contains: query, mode: 'insensitive' } }
        ]
      },
      take: limit,
      orderBy: { createdAt: 'desc' }
    })
  }
}
```

### パターン2: シードデータ管理

```typescript
// prisma/seed.ts
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

async function main() {
  // ユーザーとプロフィールを作成
  const users = await Promise.all([
    prisma.user.create({
      data: {
        email: 'alice@example.com',
        username: 'alice',
        profile: {
          create: {
            bio: 'Software Engineer'
          }
        }
      }
    }),
    prisma.user.create({
      data: {
        email: 'bob@example.com',
        username: 'bob',
        profile: {
          create: {
            bio: 'Product Manager'
          }
        }
      }
    })
  ])

  // 投稿を作成
  await prisma.post.createMany({
    data: [
      {
        title: 'Introduction to Prisma',
        content: 'Prisma is a next-generation ORM...',
        published: true,
        publishedAt: new Date(),
        userId: users[0].id
      },
      {
        title: 'Database Optimization',
        content: 'Learn how to optimize your queries...',
        published: true,
        publishedAt: new Date(),
        userId: users[1].id
      }
    ]
  })

  console.log('Seed data created successfully')
}

main()
  .catch((e) => {
    console.error(e)
    process.exit(1)
  })
  .finally(async () => {
    await prisma.$disconnect()
  })
```

```json
// package.json
{
  "prisma": {
    "seed": "ts-node prisma/seed.ts"
  }
}
```

```bash
# シードデータの実行
npx prisma db seed
```

## トラブルシューティング

### 問題1: クエリが遅い

**診断:**

```typescript
// クエリログを有効化
const prisma = new PrismaClient({
  log: ['query', 'info', 'warn', 'error']
})

// クエリ時間を計測
const start = Date.now()
const users = await prisma.user.findMany({
  include: { posts: true }
})
console.log(`Query took ${Date.now() - start}ms`)
```

**解決策:** N+1問題の解消、インデックスの追加

### 問題2: 型エラー

**症状:** TypeScriptの型エラーが発生する

**解決策:**

```bash
# Prisma Clientを再生成
npx prisma generate

# node_modulesをクリーンインストール
rm -rf node_modules
npm install
```

### 問題3: マイグレーション失敗

**診断:**

```bash
# マイグレーションステータス確認
npx prisma migrate status

# マイグレーション履歴確認
psql mydb -c "SELECT * FROM _prisma_migrations ORDER BY finished_at DESC;"
```

**解決策:** 前章のマイグレーション管理を参照

## まとめ

Prismaを完全マスターすることで、以下の成果が得られます:

**想定効果:**
- 型エラー検出: **開発時に100%**
- クエリ応答時間: **850ms → 12ms** (-99%)
- N+1問題解消: **150クエリ → 3クエリ** (-98%)
- 開発速度: **40%向上**

**重要なポイント:**
1. **型安全性**: コンパイル時にエラーを検出
2. **スキーマ設計**: データ型とインデックスの最適化
3. **N+1問題**: include/selectを活用
4. **トランザクション**: データ整合性の保証
5. **マイグレーション**: 段階的なスキーマ変更
6. **パフォーマンス**: コネクションプーリングとキャッシング

次の章では、TypeORM完全マスターについて学びます。

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**
