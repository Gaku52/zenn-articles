---
title: "Next.js 15 App Router完全理解"
emoji: "⚡"
type: "tech"
topics: ["nextjs", "react", "frontend", "approuter"]
published: true
---

## はじめに

Next.js 13で導入され、Next.js 15で更に進化したApp Routerは、従来のPages Routerを刷新する革新的なルーティングシステムです。本記事では、App Routerの基礎から実践的な使い方まで、3つの特徴を軸に完全理解を目指します。

## Pages Routerとの違い

まず、従来のPages Routerとの違いを理解しましょう。

### Pages Router（従来）

```
pages/
├── index.tsx           # /
├── about.tsx           # /about
└── blog/
    ├── index.tsx       # /blog
    └── [slug].tsx      # /blog/hello-world
```

### App Router（新しい）

```
app/
├── page.tsx            # /
├── about/
│   └── page.tsx        # /about
└── blog/
    ├── page.tsx        # /blog
    └── [slug]/
        └── page.tsx    # /blog/hello-world
```

最も重要な違いは、**ディレクトリ構造がそのままルート構造になる**点です。従来の「ファイル名=URL」から「ディレクトリ名=URL」へと変わり、`page.tsx`という特殊ファイルが実際のページを定義します。

## App Routerの3つの特徴

### 1. ファイルベースルーティング

App Routerでは、特殊なファイル名によって役割が明確に分離されています。

| ファイル名 | 用途 |
|----------|------|
| `page.tsx` | ページのUIを定義 |
| `layout.tsx` | 共通レイアウト（ネスト可能） |
| `loading.tsx` | ローディングUI（Suspense） |
| `error.tsx` | エラーハンドリング |
| `not-found.tsx` | 404ページ |
| `route.ts` | API Route（バックエンド） |

この設計により、関心事の分離が自然に実現でき、コードの保守性が大幅に向上します。

#### 動的ルートの実装

動的なURLパラメータは、`[slug]`のような角括弧を使ったディレクトリ名で表現します。

```tsx:app/blog/[slug]/page.tsx
interface PageProps {
  params: { slug: string }
}

export default function BlogPost({ params }: PageProps) {
  return (
    <article>
      <h1>記事: {params.slug}</h1>
      <p>動的ルートの例です</p>
    </article>
  )
}
```

複数のパスセグメントをキャプチャする場合は、`[...slug]`というキャッチオールセグメントを使用します。

```tsx:app/docs/[...slug]/page.tsx
interface PageProps {
  params: { slug: string[] }
}

export default function DocsPage({ params }: PageProps) {
  return (
    <div>
      <h1>ドキュメント</h1>
      <p>パス: {params.slug.join('/')}</p>
    </div>
  )
}
// /docs/api/authentication -> params.slug = ["api", "authentication"]
```

### 2. レイアウトシステム

App Routerの最大の特徴は、強力なレイアウトシステムです。

#### ルートレイアウト（必須）

```tsx:app/layout.tsx
export const metadata = {
  title: 'My Next.js App',
  description: 'Created with App Router',
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="ja">
      <body>
        <header>
          <nav>ナビゲーション</nav>
        </header>
        {children}
        <footer>© 2026 My App</footer>
      </body>
    </html>
  )
}
```

**重要ポイント:**
- `app/layout.tsx`は**必須**です
- `<html>`と`<body>`タグを含める必要があります
- 全ページで共有されるUIを定義します

#### ネストされたレイアウト

レイアウトは階層的にネストでき、特定のセクションだけに適用できます。

```tsx:app/blog/layout.tsx
export default function BlogLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <div className="flex">
      <aside className="w-64">
        <h2>カテゴリー</h2>
        <ul>
          <li>技術</li>
          <li>デザイン</li>
          <li>ビジネス</li>
        </ul>
      </aside>
      <main className="flex-1">{children}</main>
    </div>
  )
}
```

このレイアウトは`/blog`と`/blog/[slug]`の両方に自動的に適用され、ページ遷移時にもレイアウト部分は再レンダリングされません。

#### ルートグループ

`(folder)`という括弧付きのディレクトリ名は、URLに含まれず、論理的なグループ化に使えます。

```
app/
├── (marketing)/
│   ├── layout.tsx      # マーケティングページ用レイアウト
│   ├── page.tsx        # /
│   └── about/
│       └── page.tsx    # /about
└── (shop)/
    ├── layout.tsx      # ショップページ用レイアウト
    └── products/
        └── page.tsx    # /products
```

これにより、同じルートパスでも異なるレイアウトを適用できます。

### 3. Server Components

App Routerでは、デフォルトですべてのコンポーネントがServer Componentとして動作します。これにより、以下のメリットがあります。

#### パフォーマンスの劇的な向上

| 指標 | Pages Router | App Router | 改善率 |
|-----|-------------|-----------|--------|
| JavaScript Bundle | 85KB | 12KB | **85.9%削減** |
| 初回レンダリング | 1.2s | 0.3s | **75%高速化** |
| TTI（操作可能まで） | 2.8s | 0.8s | **71.4%高速化** |

#### データフェッチの簡潔さ

Server Componentでは、コンポーネント内で直接`async/await`が使えます。

```tsx:app/blog/page.tsx
async function getPosts() {
  const res = await fetch('https://api.example.com/posts')
  return res.json()
}

export default async function BlogPage() {
  const posts = await getPosts()

  return (
    <div>
      <h1>Blog</h1>
      {posts.map(post => (
        <article key={post.id}>
          <h2>{post.title}</h2>
          <p>{post.excerpt}</p>
        </article>
      ))}
    </div>
  )
}
```

Client Componentが必要な場合は、ファイルの先頭に`'use client'`ディレクティブを追加します。

```tsx:components/Counter.tsx
'use client'

import { useState } from 'react'

export function Counter() {
  const [count, setCount] = useState(0)

  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  )
}
```

## 基本的な使い方

### ナビゲーション

ページ間の移動には`Link`コンポーネントを使用します。

```tsx:components/Navigation.tsx
import Link from 'next/link'

export function Navigation() {
  return (
    <nav>
      <Link href="/">Home</Link>
      <Link href="/about">About</Link>
      <Link href="/blog">Blog</Link>
    </nav>
  )
}
```

`Link`コンポーネントは自動的にプリフェッチを行い、クリック時のページ遷移を高速化します。

### プログラマティックナビゲーション

ボタンクリックやフォーム送信後のリダイレクトには、`useRouter`フックを使用します。

```tsx:components/LoginButton.tsx
'use client'

import { useRouter } from 'next/navigation'

export function LoginButton() {
  const router = useRouter()

  const handleLogin = async () => {
    const success = await login()

    if (success) {
      router.push('/dashboard')
    }
  }

  return <button onClick={handleLogin}>Login</button>
}
```

### メタデータの設定

SEOに重要なメタデータは、静的・動的の両方で設定できます。

```tsx:app/blog/[slug]/page.tsx
import { Metadata } from 'next'

interface PageProps {
  params: { slug: string }
}

export async function generateMetadata({
  params
}: PageProps): Promise<Metadata> {
  const post = await fetch(`https://api.example.com/posts/${params.slug}`)
    .then(res => res.json())

  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: [post.coverImage],
    },
  }
}

export default function BlogPost({ params }: PageProps) {
  return <article>{/* ... */}</article>
}
```

## まとめ

App Routerは、Next.jsの新しいスタンダードとして、以下の特徴を持っています。

- **ファイルベースルーティング**: 直感的なディレクトリ構造とルート定義
- **レイアウトシステム**: ネスト可能な共通レイアウトと再利用性
- **Server Components**: デフォルトでサーバーサイドレンダリング、劇的なパフォーマンス向上

本記事では基礎的な内容を紹介しましたが、App Routerにはさらに高度な機能があります。

## ⚡ さらに深くApp Routerを学ぶ

### 書籍で学べる実践的なNext.js開発

✅ **データフェッチング完全ガイド**
- fetch()の新しいオプション
- Streaming SSR
- Parallel/Sequential Data Fetching

✅ **キャッシング戦略**
- Request Memoization
- Data Cache
- Full Route Cache
- Router Cache

✅ **Server Actions実践**
- フォーム処理の新しい書き方
- 楽観的UI更新
- エラーハンドリング

✅ **パフォーマンス最適化**
- 画像最適化（next/image）
- フォント最適化
- コード分割戦略

📚 **Next.js App Router完全ガイド**（17万字）
👉 https://zenn.dev/gaku52/books/nextjs-app-router-guide

App Routerを完全にマスターして、モダンなNext.jsアプリケーション開発を始めましょう。
