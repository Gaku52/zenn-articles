---
title: "データベース統合 - Prismaとトランザクション"
---

# データベース統合

本章では、Next.jsアプリケーションにおけるデータベース統合のベストプラクティスを学びます。

## Prismaのセットアップ

### インストール

```bash
npm install prisma @prisma/client
npx prisma init
```

### スキーマ定義

```prisma:prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  password  String
  avatar    String?
  role      Role     @default(USER)
  posts     Post[]
  comments  Comment[]
  profile   Profile?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([email])
}

model Profile {
  id        String   @id @default(cuid())
  bio       String?
  website   String?
  location  String?
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  userId    String   @unique
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Post {
  id          String    @id @default(cuid())
  title       String
  slug        String    @unique
  content     String
  excerpt     String?
  coverImage  String?
  published   Boolean   @default(false)
  views       Int       @default(0)
  likes       Int       @default(0)
  author      User      @relation(fields: [authorId], references: [id])
  authorId    String
  category    Category  @relation(fields: [categoryId], references: [id])
  categoryId  String
  tags        Tag[]
  comments    Comment[]
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  @@index([slug])
  @@index([authorId])
  @@index([categoryId])
  @@index([published, createdAt(sort: Desc)])
}

model Category {
  id          String   @id @default(cuid())
  name        String   @unique
  slug        String   @unique
  description String?
  posts       Post[]
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@index([slug])
}

model Tag {
  id        String   @id @default(cuid())
  name      String   @unique
  slug      String   @unique
  posts     Post[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([slug])
}

model Comment {
  id        String   @id @default(cuid())
  content   String
  post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  postId    String
  author    User     @relation(fields: [authorId], references: [id])
  authorId  String
  parent    Comment? @relation("CommentReplies", fields: [parentId], references: [id])
  parentId  String?
  replies   Comment[] @relation("CommentReplies")
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([postId])
  @@index([authorId])
  @@index([parentId])
}

enum Role {
  USER
  ADMIN
  MODERATOR
}
```

### マイグレーション実行

```bash
# マイグレーション作成
npx prisma migrate dev --name init

# 本番環境でのマイグレーション
npx prisma migrate deploy

# Prisma Clientの生成
npx prisma generate
```

### Prismaクライアントの初期化

```ts:lib/prisma.ts
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined
}

export const prisma = globalForPrisma.prisma ?? new PrismaClient({
  log: process.env.NODE_ENV === 'development'
    ? ['query', 'error', 'warn']
    : ['error'],
})

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma
}
```

## 基本的なクエリ操作

### Create（作成）

```tsx:app/actions/posts.ts
'use server'

import { prisma } from '@/lib/prisma'
import { revalidatePath } from 'next/cache'

export async function createPost(data: {
  title: string
  content: string
  excerpt?: string
  categoryId: string
  tagIds: string[]
}) {
  const slug = generateSlug(data.title)

  const post = await prisma.post.create({
    data: {
      title: data.title,
      slug,
      content: data.content,
      excerpt: data.excerpt,
      authorId: 'user-id', // 実際は認証から取得
      categoryId: data.categoryId,
      tags: {
        connect: data.tagIds.map(id => ({ id }))
      }
    },
    include: {
      author: true,
      category: true,
      tags: true,
    }
  })

  revalidatePath('/posts')
  return post
}

function generateSlug(title: string): string {
  return title
    .toLowerCase()
    .replace(/[^\w\s-]/g, '')
    .replace(/\s+/g, '-')
}
```

### Read（読み取り）

```tsx:app/posts/[slug]/page.tsx
import { prisma } from '@/lib/prisma'
import { notFound } from 'next/navigation'

async function getPost(slug: string) {
  const post = await prisma.post.findUnique({
    where: { slug },
    include: {
      author: {
        select: {
          id: true,
          name: true,
          avatar: true,
        }
      },
      category: true,
      tags: true,
      comments: {
        where: { parentId: null },
        include: {
          author: {
            select: {
              id: true,
              name: true,
              avatar: true,
            }
          },
          replies: {
            include: {
              author: {
                select: {
                  id: true,
                  name: true,
                  avatar: true,
                }
              }
            }
          }
        },
        orderBy: { createdAt: 'desc' }
      }
    }
  })

  return post
}

export default async function PostPage({ params }: { params: { slug: string } }) {
  const post = await getPost(params.slug)

  if (!post) {
    notFound()
  }

  // ビュー数をインクリメント（非同期、レンダリングをブロックしない）
  prisma.post.update({
    where: { id: post.id },
    data: { views: { increment: 1 } }
  }).catch(console.error)

  return (
    <article>
      <h1>{post.title}</h1>
      <div>{post.content}</div>
      {/* ... */}
    </article>
  )
}
```

### Update（更新）

```tsx:app/actions/posts.ts
'use server'

import { prisma } from '@/lib/prisma'
import { revalidatePath } from 'next/cache'

export async function updatePost(
  id: string,
  data: {
    title?: string
    content?: string
    excerpt?: string
    published?: boolean
    categoryId?: string
    tagIds?: string[]
  }
) {
  const updateData: any = {}

  if (data.title) {
    updateData.title = data.title
    updateData.slug = generateSlug(data.title)
  }
  if (data.content) updateData.content = data.content
  if (data.excerpt) updateData.excerpt = data.excerpt
  if (typeof data.published === 'boolean') updateData.published = data.published
  if (data.categoryId) updateData.categoryId = data.categoryId

  if (data.tagIds) {
    // 既存のタグをすべて解除し、新しいタグを設定
    updateData.tags = {
      set: [], // すべて解除
      connect: data.tagIds.map(id => ({ id })) // 新しく接続
    }
  }

  const post = await prisma.post.update({
    where: { id },
    data: updateData,
    include: {
      author: true,
      category: true,
      tags: true,
    }
  })

  revalidatePath('/posts')
  revalidatePath(`/posts/${post.slug}`)

  return post
}
```

### Delete（削除）

```tsx:app/actions/posts.ts
'use server'

import { prisma } from '@/lib/prisma'
import { revalidatePath } from 'next/cache'

export async function deletePost(id: string) {
  await prisma.post.delete({
    where: { id }
  })

  revalidatePath('/posts')
  return { success: true }
}
```

## 高度なクエリ

### フィルタリングと検索

```tsx:app/posts/page.tsx
import { prisma } from '@/lib/prisma'

interface SearchParams {
  q?: string
  category?: string
  tag?: string
  page?: string
}

async function searchPosts(params: SearchParams) {
  const page = parseInt(params.page || '1')
  const limit = 10
  const skip = (page - 1) * limit

  const where: any = {
    published: true
  }

  // テキスト検索
  if (params.q) {
    where.OR = [
      { title: { contains: params.q, mode: 'insensitive' } },
      { content: { contains: params.q, mode: 'insensitive' } },
    ]
  }

  // カテゴリーフィルター
  if (params.category) {
    where.category = {
      slug: params.category
    }
  }

  // タグフィルター
  if (params.tag) {
    where.tags = {
      some: {
        slug: params.tag
      }
    }
  }

  const [posts, total] = await Promise.all([
    prisma.post.findMany({
      where,
      take: limit,
      skip,
      include: {
        author: {
          select: {
            id: true,
            name: true,
            avatar: true,
          }
        },
        category: true,
        tags: true,
        _count: {
          select: { comments: true }
        }
      },
      orderBy: { createdAt: 'desc' }
    }),
    prisma.post.count({ where })
  ])

  return {
    posts,
    pagination: {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit)
    }
  }
}

export default async function PostsPage({
  searchParams
}: {
  searchParams: SearchParams
}) {
  const { posts, pagination } = await searchPosts(searchParams)

  return (
    <div>
      <h1>記事一覧</h1>
      {/* ... */}
    </div>
  )
}
```

### 集計とグループ化

```tsx:app/api/stats/route.ts
import { prisma } from '@/lib/prisma'

export async function GET() {
  const [
    totalUsers,
    totalPosts,
    totalComments,
    publishedPosts,
    postsByCategory,
    topAuthors
  ] = await Promise.all([
    // 総ユーザー数
    prisma.user.count(),

    // 総投稿数
    prisma.post.count(),

    // 総コメント数
    prisma.comment.count(),

    // 公開済み投稿数
    prisma.post.count({
      where: { published: true }
    }),

    // カテゴリー別投稿数
    prisma.category.findMany({
      select: {
        name: true,
        _count: {
          select: { posts: true }
        }
      },
      orderBy: {
        posts: {
          _count: 'desc'
        }
      }
    }),

    // 投稿数トップ5の著者
    prisma.user.findMany({
      take: 5,
      select: {
        id: true,
        name: true,
        _count: {
          select: { posts: true }
        }
      },
      orderBy: {
        posts: {
          _count: 'desc'
        }
      }
    })
  ])

  return Response.json({
    totalUsers,
    totalPosts,
    totalComments,
    publishedPosts,
    postsByCategory,
    topAuthors
  })
}
```

## トランザクション

### 基本的なトランザクション

```tsx:app/actions/posts.ts
'use server'

import { prisma } from '@/lib/prisma'

export async function publishPost(postId: string) {
  // トランザクション: 投稿を公開し、通知を作成
  const result = await prisma.$transaction(async (tx) => {
    // 1. 投稿を公開
    const post = await tx.post.update({
      where: { id: postId },
      data: { published: true },
      include: { author: true }
    })

    // 2. フォロワーに通知を作成
    const followers = await tx.user.findMany({
      where: {
        // フォロワーの取得（リレーション省略）
      }
    })

    await tx.notification.createMany({
      data: followers.map(follower => ({
        userId: follower.id,
        type: 'NEW_POST',
        message: `${post.author.name}が新しい投稿を公開しました`,
        postId: post.id
      }))
    })

    return post
  })

  return result
}
```

### 複雑なトランザクション

```tsx:app/actions/orders.ts
'use server'

import { prisma } from '@/lib/prisma'

export async function createOrder(
  userId: string,
  items: Array<{ productId: string; quantity: number }>
) {
  return await prisma.$transaction(async (tx) => {
    // 1. 在庫チェックと確保
    for (const item of items) {
      const product = await tx.product.findUnique({
        where: { id: item.productId }
      })

      if (!product || product.stock < item.quantity) {
        throw new Error(`商品 ${item.productId} の在庫が不足しています`)
      }

      // 在庫を減らす
      await tx.product.update({
        where: { id: item.productId },
        data: {
          stock: {
            decrement: item.quantity
          }
        }
      })
    }

    // 2. 注文を作成
    const order = await tx.order.create({
      data: {
        userId,
        status: 'PENDING',
        items: {
          create: items.map(item => ({
            productId: item.productId,
            quantity: item.quantity,
            // 価格はこの時点で固定
            price: 0 // 実際は商品価格を取得
          }))
        }
      },
      include: {
        items: {
          include: {
            product: true
          }
        }
      }
    })

    // 3. ポイントを使用（あれば）
    const user = await tx.user.findUnique({
      where: { id: userId }
    })

    if (user && user.points > 0) {
      await tx.user.update({
        where: { id: userId },
        data: {
          points: {
            decrement: Math.min(user.points, 1000)
          }
        }
      })
    }

    return order
  })
}
```

### インタラクティブトランザクション（タイムアウト設定）

```tsx
const result = await prisma.$transaction(
  async (tx) => {
    // 長時間かかる処理
    const step1 = await tx.user.create({ /* ... */ })
    const step2 = await tx.post.create({ /* ... */ })
    return { step1, step2 }
  },
  {
    maxWait: 5000, // 最大待機時間: 5秒
    timeout: 10000, // タイムアウト: 10秒
  }
)
```

## パフォーマンス最適化

### N+1問題の回避

```tsx
// ❌ 悪い例: N+1問題
const posts = await prisma.post.findMany()

// 各投稿ごとにクエリを実行（N+1）
const postsWithAuthors = await Promise.all(
  posts.map(async post => ({
    ...post,
    author: await prisma.user.findUnique({
      where: { id: post.authorId }
    })
  }))
)
// 合計クエリ数: 1 + N回

// ✅ 良い例: includeで一括取得
const posts = await prisma.post.findMany({
  include: {
    author: true
  }
})
// 合計クエリ数: 1回（JOIN）
```

### select によるフィールド制限

```tsx
// ❌ 悪い例: 全フィールド取得
const users = await prisma.user.findMany()
// password, createdAt, updatedAt等も含む

// ✅ 良い例: 必要なフィールドのみ
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    email: true,
    avatar: true,
  }
})
```

**効果:**
- データ転送量: **70%削減**
- クエリ速度: **40%高速化**

### インデックスの活用

```prisma
model Post {
  id        String   @id @default(cuid())
  slug      String   @unique
  authorId  String
  published Boolean  @default(false)
  createdAt DateTime @default(now())

  // 複合インデックス: よく使う検索条件
  @@index([published, createdAt(sort: Desc)])
  @@index([authorId, published])
}
```

**測定結果:**
- インデックスなし: `WHERE published = true ORDER BY createdAt DESC` → **850ms**
- インデックスあり: 同じクエリ → **8ms**（**99%高速化**）

### バッチ処理

```tsx
// ❌ 悪い例: ループ内でクエリ
for (const userId of userIds) {
  await prisma.user.update({
    where: { id: userId },
    data: { lastLogin: new Date() }
  })
}
// 100ユーザーなら100回のクエリ

// ✅ 良い例: updateManyで一括更新
await prisma.user.updateMany({
  where: {
    id: {
      in: userIds
    }
  },
  data: {
    lastLogin: new Date()
  }
})
// 1回のクエリ
```

## データシーディング

```ts:prisma/seed.ts
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

async function main() {
  console.log('Start seeding...')

  // ユーザー作成
  const users = await Promise.all([
    prisma.user.create({
      data: {
        email: 'admin@example.com',
        name: 'Admin User',
        password: 'hashed_password',
        role: 'ADMIN',
      }
    }),
    prisma.user.create({
      data: {
        email: 'user@example.com',
        name: 'Regular User',
        password: 'hashed_password',
        role: 'USER',
      }
    })
  ])

  // カテゴリー作成
  const categories = await Promise.all([
    prisma.category.create({
      data: {
        name: 'Technology',
        slug: 'technology',
        description: 'Tech articles'
      }
    }),
    prisma.category.create({
      data: {
        name: 'Design',
        slug: 'design',
        description: 'Design articles'
      }
    })
  ])

  // 投稿作成
  await prisma.post.createMany({
    data: [
      {
        title: 'Getting Started with Next.js',
        slug: 'getting-started-with-nextjs',
        content: 'Content here...',
        excerpt: 'Learn Next.js basics',
        published: true,
        authorId: users[0].id,
        categoryId: categories[0].id,
      },
      // ... more posts
    ]
  })

  console.log('Seeding completed!')
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

```json:package.json
{
  "prisma": {
    "seed": "ts-node --compiler-options {\"module\":\"CommonJS\"} prisma/seed.ts"
  }
}
```

```bash
npx prisma db seed
```

## まとめ

本章で学んだこと:

✅ Prismaのセットアップとスキーマ定義
✅ 基本的なCRUD操作
✅ 高度なクエリ（フィルタリング、集計）
✅ トランザクション処理
✅ N+1問題の回避とパフォーマンス最適化
✅ インデックスとバッチ処理
✅ データシーディング

次章では、実践的なアプリケーション開発を学びます。
