---
title: "Next.js App Router 完全ガイド"
---

# Chapter 5: Next.js App Router 完全ガイド

> Server ComponentsとClient Componentsを完全に理解し、実践で使いこなす

## この章で学べること

この章では、Next.js 13+のApp Routerを完全にマスターします。

- ✅ Server ComponentsとClient Componentsの使い分け
- ✅ データフェッチングの最適戦略
- ✅ キャッシングとRevalidation
- ✅ 実務で即使えるルーティングパターン

**前提知識**: Chapter 01-04の内容、Next.jsの基礎

**所要時間**: 70-90分

---

## 1. Server Components vs Client Components

### Server Components とは

**Server Components** は、サーバー上でのみ実行されるReactコンポーネントです。Next.js App Routerではデフォルトで全てのコンポーネントがServer Componentsです。

**主な特徴：**
- サーバーでレンダリング、HTMLとして送信
- クライアントバンドルに含まれない（バンドルサイズ0KB）
- データベースやAPIに直接アクセス可能
- 環境変数を安全に使用可能
- async/awaitで非同期処理が可能

### Client Components とは

**Client Components** は、クライアント（ブラウザ）で実行されるReactコンポーネントです。`'use client'` ディレクティブで明示的に指定します。

**主な特徴：**
- ブラウザでレンダリング
- React Hooksが使用可能（useState, useEffect等）
- イベントハンドラー（onClick, onChange等）
- ブラウザAPIアクセス（localStorage, window等）
- インタラクティブなUI

---

## 2. Server Componentsの基礎

### 基本的な実装

```tsx
// app/posts/page.tsx
// ✅ Server Component（デフォルト）

import { prisma } from '@/lib/prisma'

export default async function PostsPage() {
  // 直接データベースにアクセス
  const posts = await prisma.post.findMany({
    orderBy: { createdAt: 'desc' },
    take: 20,
  })

  return (
    <div>
      <h1>投稿一覧</h1>
      <ul>
        {posts.map(post => (
          <li key={post.id}>
            <h2>{post.title}</h2>
            <p>{post.excerpt}</p>
          </li>
        ))}
      </ul>
    </div>
  )
}
```tsx

### データフェッチングパターン

#### パターン1: fetch API（推奨）

```tsx
// app/users/page.tsx
interface User {
  id: number
  name: string
  email: string
}

async function getUsers(): Promise<User[]> {
  const res = await fetch('https://api.example.com/users', {
    next: { revalidate: 3600 } // 1時間キャッシュ
  })

  if (!res.ok) {
    throw new Error('ユーザー取得に失敗しました')
  }

  return res.json()
}

export default async function UsersPage() {
  const users = await getUsers()

  return (
    <div>
      {users.map(user => (
        <UserCard key={user.id} user={user} />
      ))}
    </div>
  )
}
```tsx

#### パターン2: 並列データフェッチング

```tsx
// app/dashboard/page.tsx
async function getStats() {
  const res = await fetch('https://api.example.com/stats')
  return res.json()
}

async function getRecentOrders() {
  const res = await fetch('https://api.example.com/orders/recent')
  return res.json()
}

async function getUserActivity() {
  const res = await fetch('https://api.example.com/activity')
  return res.json()
}

export default async function DashboardPage() {
  // 並列実行（高速）
  const [stats, orders, activity] = await Promise.all([
    getStats(),
    getRecentOrders(),
    getUserActivity(),
  ])

  return (
    <div>
      <StatsWidget data={stats} />
      <OrdersList orders={orders} />
      <ActivityFeed activity={activity} />
    </div>
  )
}
```tsx

### 環境変数の安全な使用

```tsx
// app/api-status/page.tsx
export default async function ApiStatusPage() {
  // ✅ サーバー側なので安全
  const apiKey = process.env.SECRET_API_KEY
  const apiUrl = process.env.INTERNAL_API_URL

  const res = await fetch(`${apiUrl}/status`, {
    headers: {
      'Authorization': `Bearer ${apiKey}`
    }
  })

  const status = await res.json()

  return (
    <div>
      <h1>API Status</h1>
      <pre>{JSON.stringify(status, null, 2)}</pre>
    </div>
  )
}
```tsx

---

## 3. Client Componentsの基礎

### 基本的な実装

```tsx
// components/Counter.tsx
'use client' // ← 必須

import { useState } from 'react'

export function Counter() {
  const [count, setCount] = useState(0)

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </div>
  )
}
```tsx

### インタラクティブなフォーム

```tsx
// components/SearchForm.tsx
'use client'

import { useState, useCallback } from 'react'
import { useRouter } from 'next/navigation'

interface SearchFormProps {
  initialQuery?: string
}

export function SearchForm({ initialQuery = '' }: SearchFormProps) {
  const router = useRouter()
  const [query, setQuery] = useState(initialQuery)

  const handleSubmit = useCallback((e: React.FormEvent) => {
    e.preventDefault()
    if (query.trim()) {
      router.push(`/search?q=${encodeURIComponent(query)}`)
    }
  }, [query, router])

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="検索..."
      />
      <button type="submit">検索</button>
    </form>
  )
}
```tsx

### ブラウザAPIの使用

```tsx
// components/ThemeToggle.tsx
'use client'

import { useState, useEffect } from 'react'

type Theme = 'light' | 'dark'

export function ThemeToggle() {
  const [theme, setTheme] = useState<Theme>('light')

  useEffect(() => {
    // localStorage から読み込み
    const saved = localStorage.getItem('theme') as Theme
    if (saved) {
      setTheme(saved)
      document.documentElement.classList.toggle('dark', saved === 'dark')
    }
  }, [])

  const toggleTheme = () => {
    const newTheme = theme === 'light' ? 'dark' : 'light'
    setTheme(newTheme)
    localStorage.setItem('theme', newTheme)
    document.documentElement.classList.toggle('dark', newTheme === 'dark')
  }

  return (
    <button onClick={toggleTheme}>
      {theme === 'light' ? '🌙' : '☀️'}
    </button>
  )
}
```tsx

---

## 4. 使い分け戦略

### 決定フローチャート

```tsx
コンポーネントを作成する
↓
インタラクティブか？
├─ YES → Client Component
│   ├─ useState/useEffectを使う？ → Client Component
│   ├─ onClick等のイベントハンドラー？ → Client Component
│   └─ ブラウザAPI（localStorage等）？ → Client Component
│
└─ NO → Server Component（デフォルト）
    ├─ データベース直接アクセス？ → Server Component
    ├─ 環境変数（秘密鍵）を使う？ → Server Component
    └─ 静的コンテンツ？ → Server Component
```tsx

### パターン別実装

#### パターン1: Server Component のみ

```tsx
// app/about/page.tsx
// ✅ 静的コンテンツ → Server Component

export default function AboutPage() {
  return (
    <div>
      <h1>会社概要</h1>
      <p>私たちは...</p>
    </div>
  )
}
```tsx

#### パターン2: Server + Client 混在（推奨）

```tsx
// app/products/page.tsx（Server Component）
import { prisma } from '@/lib/prisma'
import { ProductFilters } from '@/components/ProductFilters' // Client
import { ProductCard } from '@/components/ProductCard' // Server

export default async function ProductsPage() {
  // サーバーでデータ取得
  const products = await prisma.product.findMany()
  const categories = await prisma.category.findMany()

  return (
    <div>
      {/* Client Component: フィルタリング機能 */}
      <ProductFilters categories={categories} />

      {/* Server Component: 商品カード */}
      <div className="grid">
        {products.map(product => (
          <ProductCard key={product.id} product={product} />
        ))}
      </div>
    </div>
  )
}

// components/ProductFilters.tsx（Client Component）
'use client'

import { useRouter, useSearchParams } from 'next/navigation'

export function ProductFilters({ categories }) {
  const router = useRouter()
  const searchParams = useSearchParams()

  const handleFilter = (categoryId: string) => {
    const params = new URLSearchParams(searchParams)
    params.set('category', categoryId)
    router.push(`?${params.toString()}`)
  }

  return (
    <div>
      {categories.map(cat => (
        <button key={cat.id} onClick={() => handleFilter(cat.id)}>
          {cat.name}
        </button>
      ))}
    </div>
  )
}

// components/ProductCard.tsx（Server Component）
export function ProductCard({ product }) {
  return (
    <div>
      <h3>{product.name}</h3>
      <p>{product.price}</p>
    </div>
  )
}
```tsx

---

## 5. データフェッチングとキャッシング

### キャッシング戦略

```tsx
// 1. デフォルト: 無期限キャッシュ
fetch('https://api.example.com/data')

// 2. Revalidate: 60秒ごとに再検証
fetch('https://api.example.com/data', {
  next: { revalidate: 60 }
})

// 3. No Cache: キャッシュしない
fetch('https://api.example.com/data', {
  cache: 'no-store'
})

// 4. Force Cache: 強制的にキャッシュ
fetch('https://api.example.com/data', {
  cache: 'force-cache'
})
```tsx

### Revalidation パターン

```tsx
// app/posts/[id]/page.tsx
interface Post {
  id: string
  title: string
  content: string
  updatedAt: Date
}

async function getPost(id: string): Promise<Post> {
  const res = await fetch(`https://api.example.com/posts/${id}`, {
    // 10秒ごとに再検証
    next: { revalidate: 10 }
  })

  if (!res.ok) {
    throw new Error('Failed to fetch post')
  }

  return res.json()
}

export default async function PostPage({ params }: { params: { id: string } }) {
  const post = await getPost(params.id)

  return (
    <article>
      <h1>{post.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
      <time>{post.updatedAt.toLocaleString()}</time>
    </article>
  )
}
```tsx

### On-Demand Revalidation

```tsx
// app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from 'next/cache'
import { NextRequest } from 'next/server'

export async function POST(request: NextRequest) {
  const secret = request.nextUrl.searchParams.get('secret')

  // シークレットトークンで認証
  if (secret !== process.env.REVALIDATE_SECRET) {
    return new Response('Invalid secret', { status: 401 })
  }

  const path = request.nextUrl.searchParams.get('path')

  if (path) {
    // 特定のパスを再検証
    revalidatePath(path)
    return Response.json({ revalidated: true, path })
  }

  const tag = request.nextUrl.searchParams.get('tag')

  if (tag) {
    // 特定のタグを再検証
    revalidateTag(tag)
    return Response.json({ revalidated: true, tag })
  }

  return Response.json({ revalidated: false })
}

// 使用例: タグ付きフェッチ
async function getData() {
  const res = await fetch('https://api.example.com/data', {
    next: { tags: ['posts'] }
  })
  return res.json()
}

// CMSからWebhookで呼び出し
// POST /api/revalidate?secret=xxx&tag=posts
```tsx

---

## 6. ルーティングパターン

### Dynamic Routes

```tsx
// app/blog/[slug]/page.tsx
interface PageProps {
  params: { slug: string }
  searchParams: { [key: string]: string | string[] | undefined }
}

export default async function BlogPostPage({ params, searchParams }: PageProps) {
  const post = await getPost(params.slug)

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>
  )
}

// Static Params生成（ビルド時に生成）
export async function generateStaticParams() {
  const posts = await getPosts()

  return posts.map(post => ({
    slug: post.slug
  }))
}
```tsx

### Catch-all Routes

```tsx
// app/shop/[...slug]/page.tsx
// /shop/electronics
// /shop/electronics/phones
// /shop/electronics/phones/iphone

interface PageProps {
  params: { slug: string[] }
}

export default function ShopPage({ params }: PageProps) {
  const { slug } = params

  // slug = ['electronics', 'phones', 'iphone']
  const breadcrumbs = slug.join(' > ')

  return (
    <div>
      <nav>{breadcrumbs}</nav>
      <ProductListing category={slug[slug.length - 1]} />
    </div>
  )
}
```tsx

### Parallel Routes

```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,
  analytics,
  team
}: {
  children: React.ReactNode
  analytics: React.ReactNode
  team: React.ReactNode
}) {
  return (
    <div>
      <div>{children}</div>
      <div className="grid grid-cols-2">
        <div>{analytics}</div>
        <div>{team}</div>
      </div>
    </div>
  )
}

// app/dashboard/@analytics/page.tsx
export default function AnalyticsSlot() {
  return <div>Analytics Dashboard</div>
}

// app/dashboard/@team/page.tsx
export default function TeamSlot() {
  return <div>Team Overview</div>
}
```tsx

---

## 7. Loading と Error ハンドリング

### Loading UI

```tsx
// app/posts/loading.tsx
export default function Loading() {
  return (
    <div className="loading">
      <Spinner />
      <p>読み込み中...</p>
    </div>
  )
}

// Suspenseを使った部分的なLoading
// app/dashboard/page.tsx
import { Suspense } from 'react'

export default function DashboardPage() {
  return (
    <div>
      <h1>Dashboard</h1>
      <Suspense fallback={<SkeletonStats />}>
        <Stats />
      </Suspense>
      <Suspense fallback={<SkeletonChart />}>
        <RevenueChart />
      </Suspense>
    </div>
  )
}
```tsx

### Error Handling

```tsx
// app/posts/error.tsx
'use client' // Error components must be Client Components

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <div>
      <h2>エラーが発生しました</h2>
      <p>{error.message}</p>
      <button onClick={() => reset()}>
        再試行
      </button>
    </div>
  )
}
```tsx

---

## まとめ

この章では、Next.js App Routerの実践的な使い方を学びました。

**重要ポイント**:
- ✅ Server ComponentsとClient Componentsを正しく使い分ける
- ✅ デフォルトはServer Component、必要な時だけClient Component
- ✅ データフェッチングはServer Componentで行う
- ✅ キャッシングとRevalidationを適切に設定
- ✅ Loading/Error UIで優れたUXを実現

**次章予告**: Chapter 6では、レンダリング最適化とCore Web Vitalsを学びます。

---

**参考リンク**:
- [Next.js App Router](https://nextjs.org/docs/app)
- [Server Components](https://nextjs.org/docs/app/building-your-application/rendering/server-components)
- [Data Fetching](https://nextjs.org/docs/app/building-your-application/data-fetching)
