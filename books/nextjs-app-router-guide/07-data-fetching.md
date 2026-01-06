---
title: "データフェッチング - fetch、Prisma、外部API"
---

# データフェッチング

本章では、Next.js App Routerにおけるデータフェッチングのベストプラクティスを完全に理解します。

## fetch APIの拡張機能

Next.jsは`fetch` APIを拡張し、キャッシングとリバリデーションを自動化しています。

### 基本的なfetch

```tsx:app/posts/page.tsx
async function getPosts() {
  const res = await fetch('https://jsonplaceholder.typicode.com/posts')

  if (!res.ok) {
    throw new Error('Failed to fetch posts')
  }

  return res.json()
}

export default async function PostsPage() {
  const posts = await getPosts()

  return (
    <div>
      <h1>Posts</h1>
      <ul>
        {posts.map((post: any) => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </div>
  )
}
```

### キャッシュオプション

| オプション | 説明 | 使用例 |
|----------|------|--------|
| `force-cache` | 永続的にキャッシュ（デフォルト） | 静的コンテンツ |
| `no-store` | キャッシュしない | リアルタイムデータ |
| `revalidate` | 指定秒数後に再検証 | 準リアルタイム |

### 静的データ（デフォルト）

```tsx
async function getStaticData() {
  // デフォルトでキャッシュされる
  const res = await fetch('https://api.example.com/data')
  return res.json()
}
```

### 動的データ（キャッシュなし）

```tsx
async function getDynamicData() {
  const res = await fetch('https://api.example.com/data', {
    cache: 'no-store' // 常に最新データを取得
  })
  return res.json()
}
```

### 時間ベースリバリデーション

```tsx
async function getRevalidatedData() {
  const res = await fetch('https://api.example.com/data', {
    next: { revalidate: 3600 } // 1時間ごとに再検証
  })
  return res.json()
}
```

## データベース直接アクセス

### Prismaのセットアップ

```bash
npm install prisma @prisma/client
npx prisma init
```

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
  posts     Post[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Post {
  id        String   @id @default(cuid())
  title     String
  content   String?
  published Boolean  @default(false)
  author    User     @relation(fields: [authorId], references: [id])
  authorId  String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

### Prismaクライアントの初期化

```ts:lib/prisma.ts
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined
}

export const prisma = globalForPrisma.prisma ?? new PrismaClient({
  log: process.env.NODE_ENV === 'development' ? ['query', 'error', 'warn'] : ['error'],
})

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma
}
```

**重要:** この実装により、開発環境での接続プールの枯渇を防ぎます。

### 基本的なクエリ

```tsx:app/users/page.tsx
import { prisma } from '@/lib/prisma'

async function getUsers() {
  return await prisma.user.findMany({
    include: {
      posts: true
    },
    orderBy: {
      createdAt: 'desc'
    }
  })
}

export default async function UsersPage() {
  const users = await getUsers()

  return (
    <div>
      <h1>ユーザー一覧</h1>
      <ul>
        {users.map(user => (
          <li key={user.id}>
            {user.name} - {user.posts.length} posts
          </li>
        ))}
      </ul>
    </div>
  )
}
```

### リレーションの取得

```tsx:app/posts/[id]/page.tsx
import { prisma } from '@/lib/prisma'
import { notFound } from 'next/navigation'

async function getPost(id: string) {
  const post = await prisma.post.findUnique({
    where: { id },
    include: {
      author: {
        select: {
          id: true,
          name: true,
          email: true,
        }
      },
      comments: {
        include: {
          author: true
        },
        orderBy: {
          createdAt: 'desc'
        }
      }
    }
  })

  return post
}

export default async function PostPage({ params }: { params: { id: string } }) {
  const post = await getPost(params.id)

  if (!post) {
    notFound()
  }

  return (
    <article>
      <h1>{post.title}</h1>
      <p>by {post.author.name}</p>
      <div>{post.content}</div>

      <section>
        <h2>コメント ({post.comments.length})</h2>
        {post.comments.map(comment => (
          <div key={comment.id}>
            <p><strong>{comment.author.name}</strong></p>
            <p>{comment.content}</p>
          </div>
        ))}
      </section>
    </article>
  )
}
```

## 並列データフェッチング

### パターン1: Promise.all

```tsx:app/dashboard/page.tsx
import { prisma } from '@/lib/prisma'

async function getDashboardData() {
  const [users, posts, comments] = await Promise.all([
    prisma.user.count(),
    prisma.post.count(),
    prisma.comment.count(),
  ])

  return { users, posts, comments }
}

export default async function DashboardPage() {
  const stats = await getDashboardData()

  return (
    <div className="grid grid-cols-3 gap-4">
      <div className="p-6 border rounded">
        <h2>ユーザー</h2>
        <p className="text-3xl font-bold">{stats.users}</p>
      </div>
      <div className="p-6 border rounded">
        <h2>投稿</h2>
        <p className="text-3xl font-bold">{stats.posts}</p>
      </div>
      <div className="p-6 border rounded">
        <h2>コメント</h2>
        <p className="text-3xl font-bold">{stats.comments}</p>
      </div>
    </div>
  )
}
```

**パフォーマンス比較:**
- 逐次実行: 0.3s + 0.2s + 0.2s = **0.7s**
- 並列実行: max(0.3s, 0.2s, 0.2s) = **0.3s**（**57%高速化**）

### パターン2: 個別のSuspense境界

```tsx:app/profile/page.tsx
import { Suspense } from 'react'
import { UserInfo } from '@/components/UserInfo'
import { UserPosts } from '@/components/UserPosts'
import { UserActivity } from '@/components/UserActivity'

export default function ProfilePage() {
  return (
    <div className="grid grid-cols-3 gap-8">
      <div className="col-span-2">
        <Suspense fallback={<div>Loading user info...</div>}>
          <UserInfo />
        </Suspense>

        <Suspense fallback={<div>Loading posts...</div>}>
          <UserPosts />
        </Suspense>
      </div>

      <aside>
        <Suspense fallback={<div>Loading activity...</div>}>
          <UserActivity />
        </Suspense>
      </aside>
    </div>
  )
}
```

## エラーハンドリング

### try-catchパターン

```tsx:app/posts/page.tsx
import { prisma } from '@/lib/prisma'

async function getPosts() {
  try {
    const posts = await prisma.post.findMany()
    return { posts, error: null }
  } catch (error) {
    console.error('Database error:', error)
    return { posts: [], error: 'Failed to fetch posts' }
  }
}

export default async function PostsPage() {
  const { posts, error } = await getPosts()

  if (error) {
    return (
      <div className="p-4 bg-red-50 border border-red-200 rounded">
        <p className="text-red-700">{error}</p>
      </div>
    )
  }

  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
```

### error.tsxによるエラー境界

```tsx:app/posts/error.tsx
'use client'

export default function PostsError({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <div className="p-8 text-center">
      <h2 className="text-2xl font-bold mb-4">投稿の読み込みに失敗しました</h2>
      <p className="text-gray-600 mb-4">{error.message}</p>
      <button
        onClick={reset}
        className="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600"
      >
        再試行
      </button>
    </div>
  )
}
```

## 外部API連携

### REST APIの呼び出し

```tsx:app/weather/page.tsx
interface WeatherData {
  location: string
  temperature: number
  condition: string
}

async function getWeather(city: string): Promise<WeatherData> {
  const apiKey = process.env.WEATHER_API_KEY

  const res = await fetch(
    `https://api.weather.com/v1/weather?city=${city}&key=${apiKey}`,
    {
      next: { revalidate: 3600 } // 1時間ごとに更新
    }
  )

  if (!res.ok) {
    throw new Error('Failed to fetch weather data')
  }

  return res.json()
}

export default async function WeatherPage() {
  const weather = await getWeather('Tokyo')

  return (
    <div className="p-8">
      <h1 className="text-3xl font-bold mb-4">{weather.location}</h1>
      <p className="text-6xl font-bold">{weather.temperature}°C</p>
      <p className="text-xl text-gray-600">{weather.condition}</p>
    </div>
  )
}
```

### GraphQL APIの呼び出し

```tsx:app/repositories/page.tsx
interface Repository {
  id: string
  name: string
  description: string
  stargazerCount: number
}

async function getRepositories(): Promise<Repository[]> {
  const query = `
    query {
      user(login: "vercel") {
        repositories(first: 10, orderBy: {field: STARGAZERS, direction: DESC}) {
          nodes {
            id
            name
            description
            stargazerCount
          }
        }
      }
    }
  `

  const res = await fetch('https://api.github.com/graphql', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.GITHUB_TOKEN}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ query }),
    next: { revalidate: 3600 }
  })

  const json = await res.json()
  return json.data.user.repositories.nodes
}

export default async function RepositoriesPage() {
  const repos = await getRepositories()

  return (
    <div>
      <h1>人気リポジトリ</h1>
      <ul>
        {repos.map(repo => (
          <li key={repo.id} className="border p-4 rounded mb-4">
            <h2 className="text-xl font-bold">{repo.name}</h2>
            <p className="text-gray-600">{repo.description}</p>
            <p className="text-yellow-500">★ {repo.stargazerCount}</p>
          </li>
        ))}
      </ul>
    </div>
  )
}
```

## ページネーション

### オフセットベース

```tsx:app/posts/page.tsx
import { prisma } from '@/lib/prisma'
import Link from 'next/link'

interface PageProps {
  searchParams: { page?: string }
}

const POSTS_PER_PAGE = 10

async function getPosts(page: number) {
  const skip = (page - 1) * POSTS_PER_PAGE

  const [posts, total] = await Promise.all([
    prisma.post.findMany({
      take: POSTS_PER_PAGE,
      skip,
      orderBy: { createdAt: 'desc' },
      include: { author: true },
    }),
    prisma.post.count(),
  ])

  return {
    posts,
    totalPages: Math.ceil(total / POSTS_PER_PAGE),
  }
}

export default async function PostsPage({ searchParams }: PageProps) {
  const page = Number(searchParams.page) || 1
  const { posts, totalPages } = await getPosts(page)

  return (
    <div>
      <h1>投稿一覧</h1>

      <div className="space-y-4">
        {posts.map(post => (
          <article key={post.id} className="border p-4 rounded">
            <h2 className="text-xl font-bold">{post.title}</h2>
            <p className="text-gray-600">by {post.author.name}</p>
          </article>
        ))}
      </div>

      <div className="flex gap-2 mt-8">
        {page > 1 && (
          <Link
            href={`/posts?page=${page - 1}`}
            className="px-4 py-2 bg-blue-500 text-white rounded"
          >
            前へ
          </Link>
        )}

        <span className="px-4 py-2">
          {page} / {totalPages}
        </span>

        {page < totalPages && (
          <Link
            href={`/posts?page=${page + 1}`}
            className="px-4 py-2 bg-blue-500 text-white rounded"
          >
            次へ
          </Link>
        )}
      </div>
    </div>
  )
}
```

### カーソルベース（推奨）

```tsx:app/feed/page.tsx
import { prisma } from '@/lib/prisma'
import Link from 'next/link'

interface PageProps {
  searchParams: { cursor?: string }
}

const POSTS_PER_PAGE = 20

async function getFeed(cursor?: string) {
  const posts = await prisma.post.findMany({
    take: POSTS_PER_PAGE + 1, // 1つ多く取得して「次がある」か判定
    ...(cursor && {
      cursor: { id: cursor },
      skip: 1, // カーソル自体をスキップ
    }),
    orderBy: { createdAt: 'desc' },
    include: { author: true },
  })

  const hasMore = posts.length > POSTS_PER_PAGE
  const items = hasMore ? posts.slice(0, -1) : posts
  const nextCursor = hasMore ? posts[POSTS_PER_PAGE - 1].id : null

  return { items, nextCursor }
}

export default async function FeedPage({ searchParams }: PageProps) {
  const { items, nextCursor } = await getFeed(searchParams.cursor)

  return (
    <div>
      <h1>フィード</h1>

      <div className="space-y-4">
        {items.map(post => (
          <article key={post.id} className="border p-4 rounded">
            <h2 className="text-xl font-bold">{post.title}</h2>
            <p className="text-gray-600">by {post.author.name}</p>
          </article>
        ))}
      </div>

      {nextCursor && (
        <Link
          href={`/feed?cursor=${nextCursor}`}
          className="block mt-8 text-center px-4 py-2 bg-blue-500 text-white rounded"
        >
          さらに読み込む
        </Link>
      )}
    </div>
  )
}
```

**カーソルベースのメリット:**
- 新規データ追加時も正確
- 大量データでも高速
- リアルタイムフィードに最適

## データ変更の反映

### revalidatePath

```tsx:app/actions.ts
'use server'

import { revalidatePath } from 'next/cache'
import { prisma } from '@/lib/prisma'

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string
  const content = formData.get('content') as string

  await prisma.post.create({
    data: {
      title,
      content,
      authorId: 'user-id',
    },
  })

  // /postsページのキャッシュを無効化
  revalidatePath('/posts')
}
```

### revalidateTag

```tsx:app/posts/page.tsx
async function getPosts() {
  const res = await fetch('https://api.example.com/posts', {
    next: {
      revalidate: 3600,
      tags: ['posts'] // タグ付け
    }
  })
  return res.json()
}
```

```tsx:app/actions.ts
'use server'

import { revalidateTag } from 'next/cache'

export async function createPost(data: any) {
  // 投稿作成
  await fetch('https://api.example.com/posts', {
    method: 'POST',
    body: JSON.stringify(data),
  })

  // 'posts'タグのキャッシュを無効化
  revalidateTag('posts')
}
```

## パフォーマンス最適化

### N+1問題の回避

```tsx
// ❌ 悪い例: N+1問題
async function getBadUsers() {
  const users = await prisma.user.findMany()

  // 各ユーザーごとにクエリ実行（N+1）
  const usersWithPosts = await Promise.all(
    users.map(async user => ({
      ...user,
      posts: await prisma.post.findMany({
        where: { authorId: user.id }
      })
    }))
  )

  return usersWithPosts
}
// 合計クエリ数: 1 + N回

// ✅ 良い例: includeで一括取得
async function getGoodUsers() {
  return await prisma.user.findMany({
    include: {
      posts: true
    }
  })
}
// 合計クエリ数: 1回（JOIN）
```

**パフォーマンス比較:**
- N+1問題: 100ユーザー × 10ms = **1,000ms**
- JOIN: 1回 × 50ms = **50ms**（**95%高速化**）

### データベースインデックス

```prisma:prisma/schema.prisma
model Post {
  id        String   @id @default(cuid())
  title     String
  published Boolean  @default(false)
  authorId  String
  createdAt DateTime @default(now())

  author User @relation(fields: [authorId], references: [id])

  // インデックス定義
  @@index([authorId])
  @@index([published, createdAt])
  @@index([createdAt(sort: Desc)])
}
```

**効果:**
- インデックスなし: 検索 **500ms**
- インデックスあり: 検索 **5ms**（**99%高速化**）

## まとめ

本章で学んだこと:

✅ fetch APIの拡張機能とキャッシング
✅ Prismaによるデータベースアクセス
✅ 並列データフェッチングとエラーハンドリング
✅ 外部API連携（REST/GraphQL）
✅ ページネーション（オフセット/カーソル）
✅ データ変更の反映とキャッシュ無効化
✅ N+1問題の回避とインデックス最適化

次章では、キャッシング戦略を詳しく学びます。
