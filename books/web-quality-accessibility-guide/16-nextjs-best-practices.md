---
title: "Next.jsベストプラクティス - App Routerの活用"
---

# Next.jsベストプラクティス - App Routerの活用

## Next.js App Routerとは

Next.js 13で導入されたApp Routerは、React Server Components、ストリーミング、データフェッチングの改善など、多くの新機能を提供します。

## プロジェクト構成

### 推奨ディレクトリ構造

```
app/
├── (marketing)/          # ルートグループ（パスに含まれない）
│   ├── layout.tsx
│   ├── page.tsx         # /
│   └── about/page.tsx   # /about
├── dashboard/
│   ├── layout.tsx
│   ├── page.tsx         # /dashboard
│   └── settings/page.tsx # /dashboard/settings
├── api/                 # APIルート
│   └── users/route.ts   # /api/users
├── layout.tsx           # ルートレイアウト
└── globals.css

components/
├── ui/                  # UIコンポーネント
│   ├── button.tsx
│   └── card.tsx
└── features/            # 機能別コンポーネント
    ├── user-profile.tsx
    └── post-list.tsx

lib/
├── utils.ts
├── api.ts
└── db.ts
```

## Server ComponentsとClient Components

### Server Components（デフォルト）

```tsx
// app/posts/page.tsx
// ✅ Server Component（デフォルト）
async function PostList() {
  const posts = await fetch('https://api.example.com/posts').then(res => res.json())

  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}

export default PostList
```

### Client Components

```tsx
// components/Counter.tsx
'use client' // ✅ Client Component

import { useState } from 'react'

export function Counter() {
  const [count, setCount] = useState(0)

  return (
    <button onClick={() => setCount(count + 1)}>
      カウント: {count}
    </button>
  )
}
```

### 使い分け

| 用途 | 推奨 |
|-----|-----|
| データフェッチ | Server Components |
| インタラクティブUI | Client Components |
| useState、useEffect | Client Components |
| イベントハンドラ | Client Components |

## データフェッチング

### Server Componentsでのfetch

```tsx
// app/posts/[id]/page.tsx
async function PostPage({ params }: { params: { id: string } }) {
  // ✅ Server Componentで直接fetch
  const post = await fetch(`https://api.example.com/posts/${params.id}`, {
    next: { revalidate: 60 }, // 60秒ごとに再検証
  }).then(res => res.json())

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>
  )
}

export default PostPage
```

### 並列フェッチ

```tsx
// ✅ 良い例: 並列フェッチ
async function Dashboard() {
  const [user, posts, comments] = await Promise.all([
    fetchUser(),
    fetchPosts(),
    fetchComments(),
  ])

  return (
    <div>
      <UserProfile user={user} />
      <PostList posts={posts} />
      <CommentList comments={comments} />
    </div>
  )
}

// ❌ 悪い例: 逐次フェッチ
async function DashboardBad() {
  const user = await fetchUser()
  const posts = await fetchPosts() // userの後に実行
  const comments = await fetchComments() // postsの後に実行

  return (
    <div>
      <UserProfile user={user} />
      <PostList posts={posts} />
      <CommentList comments={comments} />
    </div>
  )
}
```

## メタデータ管理

### 静的メタデータ

```tsx
// app/about/page.tsx
import { Metadata } from 'next'

export const metadata: Metadata = {
  title: '会社概要',
  description: '当社の理念、ビジョン、ミッションについて',
}

export default function AboutPage() {
  return <div>会社概要</div>
}
```

### 動的メタデータ

```tsx
// app/posts/[id]/page.tsx
import { Metadata } from 'next'

export async function generateMetadata({ params }: { params: { id: string } }): Promise<Metadata> {
  const post = await fetchPost(params.id)

  return {
    title: `${post.title} | ブログ`,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: [post.image],
    },
  }
}

export default async function PostPage({ params }: { params: { id: string } }) {
  const post = await fetchPost(params.id)

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>
  )
}

async function fetchPost(id: string) {
  // 実装
  return { title: '', excerpt: '', content: '', image: '' }
}
```

## ローディングとエラー処理

### loading.tsx

```tsx
// app/dashboard/loading.tsx
export default function Loading() {
  return (
    <div className="flex items-center justify-center min-h-screen">
      <div className="animate-spin rounded-full h-32 w-32 border-b-2 border-blue-600" />
      <span className="sr-only">読み込み中...</span>
    </div>
  )
}
```

### error.tsx

```tsx
// app/dashboard/error.tsx
'use client'

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <div role="alert" className="p-4">
      <h2 className="text-xl font-bold">エラーが発生しました</h2>
      <p className="text-gray-600">{error.message}</p>
      <button
        onClick={reset}
        className="mt-4 px-4 py-2 bg-blue-600 text-white rounded"
      >
        再試行
      </button>
    </div>
  )
}
```

## 画像最適化

### next/image

```tsx
import Image from 'next/image'

// ✅ 良い例: next/imageを使用
<Image
  src="/product.jpg"
  alt="青いTシャツ"
  width={500}
  height={500}
  priority // LCPの場合
/>

// ✅ 良い例: 外部画像
<Image
  src="https://example.com/image.jpg"
  alt="説明"
  width={500}
  height={500}
  // next.config.jsでremotePatternsを設定
/>

// ❌ 悪い例: 通常のimg
<img src="/product.jpg" alt="青いTシャツ" />
```

### next.config.js

```javascript
// next.config.js
module.exports = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'example.com',
      },
    ],
  },
}
```

## フォント最適化

### next/font

```tsx
// app/layout.tsx
import { Inter, Noto_Sans_JP } from 'next/font/google'

const inter = Inter({ subsets: ['latin'] })
const notoSansJP = Noto_Sans_JP({ subsets: ['latin'] })

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja" className={notoSansJP.className}>
      <body>{children}</body>
    </html>
  )
}
```

## アクセシビリティ対応

### ルートレイアウト

```tsx
// app/layout.tsx
import { Metadata } from 'next'

export const metadata: Metadata = {
  title: {
    template: '%s | サイト名',
    default: 'サイト名',
  },
  description: 'サイトの説明',
}

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja">
      <body>
        <a
          href="#main"
          className="sr-only focus:not-sr-only focus:absolute focus:top-4 focus:left-4 focus:z-50 focus:px-4 focus:py-2 focus:bg-blue-600 focus:text-white"
        >
          メインコンテンツへスキップ
        </a>
        <header>
          <nav aria-label="メインナビゲーション">
            {/* ナビゲーション */}
          </nav>
        </header>
        <main id="main" tabIndex={-1}>
          {children}
        </main>
        <footer>
          {/* フッター */}
        </footer>
      </body>
    </html>
  )
}
```

## API Routes

### Route Handlers

```typescript
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server'

export async function GET(request: NextRequest) {
  const users = await fetchUsers()

  return NextResponse.json(users)
}

export async function POST(request: NextRequest) {
  const body = await request.json()
  const user = await createUser(body)

  return NextResponse.json(user, { status: 201 })
}

async function fetchUsers() {
  // 実装
  return []
}

async function createUser(data: any) {
  // 実装
  return data
}
```

## パフォーマンス最適化

### 動的インポート

```tsx
import dynamic from 'next/dynamic'

// ✅ 良い例: 動的インポート（クライアントコンポーネントの遅延読み込み）
const HeavyComponent = dynamic(() => import('@/components/HeavyComponent'), {
  loading: () => <div>読み込み中...</div>,
  ssr: false, // SSRを無効化（必要に応じて）
})

export default function Page() {
  return (
    <div>
      <h1>ページタイトル</h1>
      <HeavyComponent />
    </div>
  )
}
```

### Suspense

```tsx
import { Suspense } from 'react'

export default function Dashboard() {
  return (
    <div>
      <h1>ダッシュボード</h1>
      <Suspense fallback={<div>ユーザー情報読み込み中...</div>}>
        <UserInfo />
      </Suspense>
      <Suspense fallback={<div>投稿読み込み中...</div>}>
        <Posts />
      </Suspense>
    </div>
  )
}

async function UserInfo() {
  const user = await fetchUser()
  return <div>{user.name}</div>
}

async function Posts() {
  const posts = await fetchPosts()
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>
}

async function fetchUser() {
  return { name: '' }
}

async function fetchPosts() {
  return []
}
```

## まとめ

Next.js App Routerは、Server Components、データフェッチング、メタデータ管理など、多くの機能を提供します。

### チェックリスト

- Server ComponentsとClient Componentsを適切に使い分け
- メタデータを適切に設定
- next/imageでseo画像最適化
- next/fontでフォント最適化
- loading.tsx、error.tsxでUX向上
- アクセシビリティ対応（スキップリンク、見出し階層）

## 次のステップ

次章では、状態管理について詳しく解説します。
