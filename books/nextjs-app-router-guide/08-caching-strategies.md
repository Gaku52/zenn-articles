---
title: "キャッシング戦略 - パフォーマンス最適化の核心"
---

# キャッシング戦略

本章では、Next.jsの強力なキャッシング機能を完全に理解し、最適なパフォーマンスを実現します。

## Next.jsのキャッシング階層

Next.jsは4つのレイヤーでキャッシングを行います：

| レイヤー | 場所 | 目的 | 持続期間 |
|---------|------|------|---------|
| **Request Memoization** | サーバー | 同一リクエスト内の重複排除 | リクエスト中のみ |
| **Data Cache** | サーバー | fetchの結果をキャッシュ | 永続的（revalidateまで） |
| **Full Route Cache** | サーバー | レンダリング結果をキャッシュ | 永続的（ビルド時） |
| **Router Cache** | クライアント | ページ遷移の高速化 | セッション中 |

## Request Memoization

### 自動的な重複排除

```tsx:app/page.tsx
async function getUser() {
  console.log('Fetching user...') // 1回だけ実行される
  const res = await fetch('https://api.example.com/user')
  return res.json()
}

export default async function Page() {
  const user1 = await getUser() // 実行
  const user2 = await getUser() // メモ化（キャッシュから取得）
  const user3 = await getUser() // メモ化（キャッシュから取得）

  // すべて同じデータ、fetchは1回のみ
  return <div>{user1.name}</div>
}
```

**特徴:**
- 同一リクエスト内で自動的に重複排除
- fetch、Prismaクエリなど全てに適用
- パフォーマンス向上とコスト削減

### コンポーネント間での重複排除

```tsx:app/dashboard/page.tsx
// 複数コンポーネントで同じデータ取得
async function getStats() {
  console.log('Fetching stats...') // 1回だけ実行
  return await prisma.stats.findMany()
}

async function StatsCard() {
  const stats = await getStats()
  return <div>Total: {stats.length}</div>
}

async function StatsChart() {
  const stats = await getStats() // メモ化（同じデータ）
  return <div>Chart</div>
}

export default function Dashboard() {
  return (
    <div>
      <StatsCard />
      <StatsChart />
    </div>
  )
}
```

## Data Cache

### デフォルトの永続的キャッシュ

```tsx
async function getData() {
  // デフォルトで永続的にキャッシュ
  const res = await fetch('https://api.example.com/data')
  return res.json()
}
```

**ビルド時の挙動:**
```
ビルド → データ取得 → キャッシュ保存 → 以降はキャッシュから返す
```

### cache: 'no-store' - キャッシュしない

```tsx:app/realtime/page.tsx
async function getRealtimeData() {
  const res = await fetch('https://api.example.com/live', {
    cache: 'no-store' // 毎回最新データを取得
  })
  return res.json()
}

export default async function RealtimePage() {
  const data = await getRealtimeData()

  return (
    <div>
      <h1>リアルタイムデータ</h1>
      <p>最終更新: {new Date().toLocaleString()}</p>
      <pre>{JSON.stringify(data, null, 2)}</pre>
    </div>
  )
}
```

**使用例:**
- 株価、為替レート
- リアルタイムチャット
- ライブスコア

### next.revalidate - 時間ベースリバリデーション

```tsx:app/news/page.tsx
async function getNews() {
  const res = await fetch('https://api.example.com/news', {
    next: { revalidate: 60 } // 60秒ごとに再検証
  })
  return res.json()
}

export default async function NewsPage() {
  const news = await getNews()

  return (
    <div>
      <h1>最新ニュース</h1>
      {news.map((article: any) => (
        <article key={article.id}>
          <h2>{article.title}</h2>
          <p>{article.summary}</p>
        </article>
      ))}
    </div>
  )
}
```

**動作:**
```
0秒: キャッシュ作成
1-59秒: キャッシュから返す
60秒: バックグラウンドで再検証 + 古いキャッシュを返す
61秒: 新しいキャッシュから返す
```

### revalidate値の推奨設定

| コンテンツタイプ | 推奨値 | 例 |
|---------------|--------|-----|
| 静的コンテンツ | なし（永続） | 会社概要、利用規約 |
| 準リアルタイム | 60-300秒 | ニュース、ブログ |
| 定期更新 | 3600秒（1時間） | 天気予報、統計 |
| 日次更新 | 86400秒（24時間） | 為替レート、市況 |
| リアルタイム | no-store | 株価、ライブスコア |

## Full Route Cache

### 静的レンダリング（デフォルト）

```tsx:app/about/page.tsx
// 静的にレンダリングされる
export default function AboutPage() {
  return (
    <div>
      <h1>About Us</h1>
      <p>会社概要ページ</p>
    </div>
  )
}
```

**ビルド時:**
```
next build
→ /about: HTML生成 → キャッシュ
→ 以降のリクエストは静的HTMLを返す
```

### 動的レンダリングへの切り替え

以下のいずれかを使用すると自動的に動的レンダリングになります：

```tsx:app/profile/page.tsx
import { cookies, headers } from 'next/headers'

export default async function ProfilePage() {
  // これらを使うと動的レンダリングに
  const cookieStore = cookies()
  const headersList = headers()

  return <div>Profile</div>
}
```

**動的関数:**
- `cookies()`
- `headers()`
- `searchParams`（ページProps）
- `fetch(..., { cache: 'no-store' })`

### export const dynamic

```tsx:app/api/route.ts
// ページ全体を動的にする
export const dynamic = 'force-dynamic'

export async function GET() {
  return Response.json({ time: new Date().toISOString() })
}
```

**オプション:**
- `'auto'` - デフォルト（自動判定）
- `'force-dynamic'` - 常に動的
- `'force-static'` - 常に静的
- `'error'` - 静的のみ（動的関数使用でエラー）

## Router Cache（クライアントサイド）

### 自動プリフェッチとキャッシュ

```tsx:components/Navigation.tsx
import Link from 'next/link'

export function Navigation() {
  return (
    <nav>
      {/* ビューポートに入ると自動プリフェッチ */}
      <Link href="/about">About</Link>
      <Link href="/blog">Blog</Link>
      <Link href="/contact">Contact</Link>
    </nav>
  )
}
```

**動作:**
1. Linkがビューポートに表示
2. バックグラウンドでプリフェッチ
3. Router Cacheに保存
4. クリック時に即座に表示

### キャッシュ持続時間

| ルートタイプ | キャッシュ時間 |
|------------|-------------|
| 静的ルート | 5分 |
| 動的ルート | 30秒 |

### プリフェッチの制御

```tsx
import Link from 'next/link'

export function Links() {
  return (
    <div>
      {/* デフォルト: プリフェッチあり */}
      <Link href="/about">About</Link>

      {/* プリフェッチ無効 */}
      <Link href="/heavy-page" prefetch={false}>
        Heavy Page
      </Link>
    </div>
  )
}
```

## オンデマンドリバリデーション

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

  // /postsページのキャッシュを即座に無効化
  revalidatePath('/posts')
  // または特定のパスパターン
  revalidatePath('/posts/[slug]', 'page')
}
```

**オプション:**
```tsx
revalidatePath('/blog') // /blogのみ
revalidatePath('/blog', 'page') // /blog ページ
revalidatePath('/blog', 'layout') // /blog レイアウト配下すべて
```

### revalidateTag

```tsx:app/posts/page.tsx
async function getPosts() {
  const res = await fetch('https://api.example.com/posts', {
    next: {
      revalidate: 3600,
      tags: ['posts', 'content'] // 複数タグ付け可能
    }
  })
  return res.json()
}
```

```tsx:app/actions.ts
'use server'

import { revalidateTag } from 'next/cache'

export async function publishPost() {
  // 投稿処理...

  // 'posts'タグのすべてのキャッシュを無効化
  revalidateTag('posts')
}

export async function updateSiteContent() {
  // 'content'タグのすべてのキャッシュを無効化
  revalidateTag('content')
}
```

### Webhookによる自動リバリデーション

```tsx:app/api/revalidate/route.ts
import { revalidatePath } from 'next/cache'
import { NextRequest } from 'next/server'

export async function POST(request: NextRequest) {
  const secret = request.nextUrl.searchParams.get('secret')

  // シークレットトークン検証
  if (secret !== process.env.REVALIDATE_SECRET) {
    return Response.json({ message: 'Invalid secret' }, { status: 401 })
  }

  const body = await request.json()
  const path = body.path

  try {
    revalidatePath(path)
    return Response.json({ revalidated: true, now: Date.now() })
  } catch (err) {
    return Response.json({ message: 'Error revalidating' }, { status: 500 })
  }
}
```

**使用例（CMSからのWebhook）:**
```bash
# 記事公開時にCMSから呼び出し
curl -X POST 'https://yoursite.com/api/revalidate?secret=YOUR_SECRET' \
  -H 'Content-Type: application/json' \
  -d '{"path": "/blog/new-post"}'
```

## キャッシング戦略の実践

### 戦略1: ハイブリッド（静的 + 動的）

```tsx:app/blog/[slug]/page.tsx
// 静的生成（ビルド時）
export async function generateStaticParams() {
  const posts = await prisma.post.findMany({
    select: { slug: true }
  })

  return posts.map(post => ({
    slug: post.slug,
  }))
}

// ISR（Incremental Static Regeneration）
export const revalidate = 3600 // 1時間

async function getPost(slug: string) {
  return await prisma.post.findUnique({
    where: { slug },
    include: { author: true },
  })
}

export default async function BlogPost({ params }: { params: { slug: string } }) {
  const post = await getPost(params.slug)

  return (
    <article>
      <h1>{post.title}</h1>
      <p>by {post.author.name}</p>
      <div>{post.content}</div>
    </article>
  )
}
```

**効果:**
- ビルド時: 全記事のHTMLを生成
- 1時間ごと: バックグラウンドで更新
- 新記事追加: 初回アクセス時に生成 + キャッシュ

### 戦略2: セグメントごとのキャッシング

```tsx:app/dashboard/page.tsx
import { Suspense } from 'react'

// 静的部分（キャッシュされる）
async function StaticStats() {
  const stats = await fetch('https://api.example.com/stats', {
    next: { revalidate: 3600 }
  }).then(res => res.json())

  return <div>Total Users: {stats.totalUsers}</div>
}

// 動的部分（キャッシュされない）
async function LiveData() {
  const live = await fetch('https://api.example.com/live', {
    cache: 'no-store'
  }).then(res => res.json())

  return <div>Online Now: {live.onlineUsers}</div>
}

export default function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>

      {/* 静的データ: 1時間キャッシュ */}
      <Suspense fallback={<div>Loading stats...</div>}>
        <StaticStats />
      </Suspense>

      {/* 動的データ: 常に最新 */}
      <Suspense fallback={<div>Loading live...</div>}>
        <LiveData />
      </Suspense>
    </div>
  )
}
```

## パフォーマンス測定

### キャッシュ効果の測定

```tsx:app/benchmark/page.tsx
async function getBenchmarkData() {
  const start = Date.now()

  const res = await fetch('https://api.example.com/data', {
    next: { revalidate: 60 }
  })
  const data = await res.json()

  const duration = Date.now() - start

  return { data, duration }
}

export default async function BenchmarkPage() {
  const { data, duration } = await getBenchmarkData()

  return (
    <div>
      <h1>ベンチマーク</h1>
      <p>取得時間: {duration}ms</p>
      <p>データ件数: {data.length}</p>
    </div>
  )
}
```

**測定結果（例）:**
| 状態 | 応答時間 |
|-----|---------|
| キャッシュなし | 850ms |
| キャッシュあり | 2ms |
| **改善率** | **99.8%高速化** |

## よくある間違い

### ❌ 間違い1: 動的データに永続キャッシュ

```tsx
// ❌ 悪い例: ユーザー固有データをキャッシュ
async function getUserData(userId: string) {
  const res = await fetch(`/api/users/${userId}`)
  // デフォルトで永続キャッシュされる
  return res.json()
}
```

```tsx
// ✅ 良い例: ユーザーごとに最新データ
async function getUserData(userId: string) {
  const res = await fetch(`/api/users/${userId}`, {
    cache: 'no-store'
  })
  return res.json()
}
```

### ❌ 間違い2: 過度なrevalidate

```tsx
// ❌ 悪い例: 5秒ごとに再検証（負荷が高い）
const res = await fetch('...', {
  next: { revalidate: 5 }
})
```

```tsx
// ✅ 良い例: 適切な間隔
const res = await fetch('...', {
  next: { revalidate: 300 } // 5分
})
```

### ❌ 間違い3: キャッシュの無効化忘れ

```tsx
// ❌ 悪い例: データ更新後にキャッシュ無効化しない
'use server'

export async function updatePost(id: string, data: any) {
  await prisma.post.update({
    where: { id },
    data,
  })
  // キャッシュが古いまま！
}
```

```tsx
// ✅ 良い例: 更新後にキャッシュ無効化
'use server'

import { revalidatePath } from 'next/cache'

export async function updatePost(id: string, data: any) {
  await prisma.post.update({
    where: { id },
    data,
  })

  revalidatePath('/posts')
  revalidatePath(`/posts/${id}`)
}
```

## まとめ

本章で学んだこと:

✅ Next.jsの4層キャッシング階層
✅ Request Memoizationによる重複排除
✅ Data Cacheと時間ベースリバリデーション
✅ Full Route Cacheと静的/動的レンダリング
✅ Router Cacheとプリフェッチ
✅ オンデマンドリバリデーション
✅ 実践的なキャッシング戦略

次章では、Server Actionsによるフォーム処理を学びます。
