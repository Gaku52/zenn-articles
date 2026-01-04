---
title: "レンダリング最適化とCore Web Vitals"
---

# Chapter 6: レンダリング最適化とCore Web Vitals

> ユーザー体験とSEOを劇的に改善する実践ガイド

## この章で学べること

この章では、Webパフォーマンスの核心指標と最適化手法を学びます。

- ✅ Core Web Vitals（LCP、INP、CLS）の完全理解
- ✅ Next.jsのレンダリング戦略（SSG、SSR、ISR）の使い分け
- ✅ 画像・フォント最適化の実践
- ✅ 実測値に基づいた改善手法

**前提知識**: Chapter 01-05の内容、Next.jsの基礎

**所要時間**: 70-90分

**難易度**: ★★★☆☆（中級）

---

## 目次

1. [Core Web Vitalsとは](#1-core-web-vitalsとは)
   - 3つの主要指標
   - なぜ重要か
   - 測定方法
2. [LCP（Largest Contentful Paint）](#2-lcplargest-contentful-paint)
   - LCPとは
   - 画像最適化
   - フォント最適化
3. [INP（Interaction to Next Paint）](#3-inpinteraction-to-next-paint)
   - INPとは
   - JavaScriptの最適化
   - イベントハンドラの改善
4. [CLS（Cumulative Layout Shift）](#4-clscumulative-layout-shift)
   - CLSとは
   - レイアウトシフトの防止
   - 画像・広告の対策
5. [レンダリング戦略](#5-レンダリング戦略)
   - SSG vs SSR vs ISR vs CSR
   - 使い分けフローチャート
   - 実装パターン
6. [実践：パフォーマンス改善事例](#6-実践パフォーマンス改善事例)

---

## 1. Core Web Vitalsとは

### 3つの主要指標

GoogleがWeb体験の品質を測定するために定義した3つの指標：

| 指標 | 説明 | 測定対象 | 目標値 |
|------|------|----------|--------|
| **LCP** | Largest Contentful Paint | 読み込みパフォーマンス | < 2.5秒 |
| **INP** | Interaction to Next Paint | インタラクティブ性 | < 200ms |
| **CLS** | Cumulative Layout Shift | 視覚的安定性 | < 0.1 |

### なぜ重要か

**1. SEOへの影響**

GoogleのランキングシグナルとしてCore Web Vitalsが使用されています。

**2. ビジネスへの影響**

実際の調査データ：
- **Amazon**: ページ速度が1秒遅くなると、売上が1.6%減少
- **Google**: モバイルサイトの読み込みが3秒以上かかると、53%のユーザーが離脱
- **Walmart**: 1秒の改善でコンバージョンが2%向上

**3. ユーザー体験の向上**

```typescript
// 悪いパフォーマンスの影響
const userExperience = {
  LCP_poor: '読み込みが遅い → ユーザーが離脱',
  INP_poor: 'クリックしても反応しない → イライラ',
  CLS_poor: 'レイアウトがずれる → 誤クリック'
}
```

### 測定方法

#### 1. Chrome DevTools（Lighthouse）

```bash
# Lighthouseで計測
1. Chrome DevToolsを開く（F12）
2. Lighthouseタブを選択
3. "Analyze page load"をクリック
```

#### 2. Web Vitals ライブラリ

```bash
npm install web-vitals
```

```typescript
// app/layout.tsx
import { onCLS, onINP, onLCP } from 'web-vitals'

export default function RootLayout({ children }) {
  useEffect(() => {
    onLCP(console.log)  // LCPを計測
    onINP(console.log)  // INPを計測
    onCLS(console.log)  // CLSを計測
  }, [])

  return <html>{children}</html>
}
```

#### 3. PageSpeed Insights

```bash
# URLで計測
https://pagespeed.web.dev/
```

実際のユーザーデータ（Field Data）とLab Dataの両方が表示されます。

---

## 2. LCP（Largest Contentful Paint）

### LCPとは

**定義**: ビューポート内で最も大きなコンテンツ要素がレンダリングされるまでの時間

**LCPの対象要素**:
- `<img>` 要素
- `<video>` 要素のポスター画像
- `url()` によるCSS背景画像
- テキストを含むブロックレベル要素

**目標値**:

| 評価 | LCP |
|------|-----|
| **Good** | < 2.5秒 |
| **Needs Improvement** | 2.5秒 - 4.0秒 |
| **Poor** | > 4.0秒 |

### 画像最適化

#### Next.js Imageコンポーネント

```tsx
// ❌ 悪い例: 最適化なし
<img src="/hero.jpg" alt="Hero" width={1920} height={1080} />

// ✅ 良い例: Next.js Image（自動最適化）
import Image from 'next/image'

export function HeroSection() {
  return (
    <Image
      src="/hero.jpg"
      alt="Hero"
      width={1920}
      height={1080}
      priority  // LCP要素には必須！
      quality={75}
      sizes="100vw"
      placeholder="blur"
      blurDataURL="data:image/jpeg;base64,..."
    />
  )
}
```

**効果**:
- WebP/AVIF形式への自動変換（-30~50%ファイルサイズ）
- レスポンシブ画像の自動生成
- 遅延ローディング（priority以外）
- ぼかしプレースホルダー

#### 画像フォーマットの選択

```typescript
// 画像フォーマット比較（同じ品質）
const imageFormats = {
  JPEG: '100 KB',
  WebP: '70 KB (-30%)',
  AVIF: '50 KB (-50%)'
}
```

**推奨設定**:

```typescript
// next.config.js
module.exports = {
  images: {
    formats: ['image/avif', 'image/webp'],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920, 2048, 3840],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
  },
}
```

### フォント最適化

#### 1. next/font による最適化

```tsx
// app/layout.tsx
import { Inter, Noto_Sans_JP } from 'next/font/google'

// Google Fontsの最適化読み込み
const inter = Inter({
  subsets: ['latin'],
  display: 'swap',  // FOIT（Flash of Invisible Text）を防ぐ
  variable: '--font-inter',
})

const notoSansJP = Noto_Sans_JP({
  subsets: ['latin'],
  weight: ['400', '700'],
  display: 'swap',
  variable: '--font-noto-sans-jp',
})

export default function RootLayout({ children }) {
  return (
    <html className={`${inter.variable} ${notoSansJP.variable}`}>
      <body>{children}</body>
    </html>
  )
}
```

**効果**:
- フォントファイルの自動最適化
- レイアウトシフトの防止（`font-display: swap`）
- セルフホスティングによるプライバシー向上

#### 2. ローカルフォントの最適化

```tsx
import localFont from 'next/font/local'

const myFont = localFont({
  src: './my-font.woff2',
  display: 'swap',
  variable: '--font-my-font',
})
```

---

## 3. INP（Interaction to Next Paint）

### INPとは

**定義**: ユーザーのインタラクション（クリック、タップ、キー入力）から次の描画までの時間

**目標値**:

| 評価 | INP |
|------|-----|
| **Good** | < 200ms |
| **Needs Improvement** | 200ms - 500ms |
| **Poor** | > 500ms |

### JavaScriptの最適化

#### 1. 重い処理を分割

```tsx
// ❌ 悪い例: 1度に全て処理
function SearchResults({ query }: { query: string }) {
  const results = useMemo(() => {
    // 10,000件のデータを一度に処理（ブロッキング）
    return heavyData.filter(item => item.name.includes(query))
  }, [query])

  return <ul>{results.map(r => <li key={r.id}>{r.name}</li>)}</ul>
}

// ✅ 良い例: Web Workerで非同期処理
function SearchResults({ query }: { query: string }) {
  const [results, setResults] = useState([])

  useEffect(() => {
    const worker = new Worker('/search-worker.js')

    worker.postMessage({ query, data: heavyData })

    worker.onmessage = (e) => {
      setResults(e.data)
    }

    return () => worker.terminate()
  }, [query])

  return <ul>{results.map(r => <li key={r.id}>{r.name}</li>)}</ul>
}
```

#### 2. デバウンス・スロットリング

```tsx
// hooks/useDebounce.ts
import { useState, useEffect } from 'react'

export function useDebounce<T>(value: T, delay: number = 300): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value)

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value)
    }, delay)

    return () => clearTimeout(timer)
  }, [value, delay])

  return debouncedValue
}

// 使用例
function SearchInput() {
  const [query, setQuery] = useState('')
  const debouncedQuery = useDebounce(query, 300)

  useEffect(() => {
    if (debouncedQuery) {
      // API呼び出しは300ms後のみ
      fetchResults(debouncedQuery)
    }
  }, [debouncedQuery])

  return (
    <input
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      placeholder="検索..."
    />
  )
}
```

### イベントハンドラの改善

#### 1. useTransition による優先度制御

```tsx
import { useState, useTransition } from 'react'

function FilteredList({ items }: { items: Item[] }) {
  const [filter, setFilter] = useState('')
  const [isPending, startTransition] = useTransition()

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value

    // 入力は即座に反映（高優先度）
    setFilter(value)

    // フィルタリングは低優先度（遅延可能）
    startTransition(() => {
      setFilteredItems(items.filter(item => item.name.includes(value)))
    })
  }

  return (
    <>
      <input value={filter} onChange={handleChange} />
      {isPending && <Spinner />}
      <List items={filteredItems} />
    </>
  )
}
```

---

## 4. CLS（Cumulative Layout Shift）

### CLSとは

**定義**: ページの視覚的安定性を測定する指標。予期しないレイアウトシフトの累積スコア。

**目標値**:

| 評価 | CLS |
|------|-----|
| **Good** | < 0.1 |
| **Needs Improvement** | 0.1 - 0.25 |
| **Poor** | > 0.25 |

### レイアウトシフトの防止

#### 1. 画像・動画のサイズを明示

```tsx
// ❌ 悪い例: サイズ指定なし（CLSが発生）
<img src="/product.jpg" alt="Product" />

// ✅ 良い例: width/heightを明示
<Image
  src="/product.jpg"
  alt="Product"
  width={400}
  height={300}
  style={{ width: '100%', height: 'auto' }}
/>
```

#### 2. フォント読み込み時のシフト防止

```css
/* globals.css */
:root {
  --font-inter: Inter, sans-serif;
}

body {
  font-family: var(--font-inter);
  /* フォールバックフォントと同じサイズ調整 */
  font-size-adjust: 0.5;
}
```

#### 3. 動的コンテンツのスペース確保

```tsx
// ❌ 悪い例: 広告読み込み後にレイアウトシフト
function AdBanner() {
  return <div id="ad-slot"></div>
}

// ✅ 良い例: 事前にスペース確保
function AdBanner() {
  return (
    <div style={{ minHeight: '250px', backgroundColor: '#f0f0f0' }}>
      <div id="ad-slot"></div>
    </div>
  )
}
```

#### 4. Skeleton UIの活用

```tsx
function ProductCard({ productId }: { productId: string }) {
  const { data, loading } = useProduct(productId)

  if (loading) {
    return (
      <div className="product-card">
        {/* レイアウトシフトを防ぐSkeleton */}
        <div className="skeleton-image" style={{ height: '200px' }} />
        <div className="skeleton-title" style={{ height: '24px', width: '80%' }} />
        <div className="skeleton-price" style={{ height: '20px', width: '40%' }} />
      </div>
    )
  }

  return (
    <div className="product-card">
      <Image src={data.image} width={200} height={200} alt={data.name} />
      <h3>{data.name}</h3>
      <p>{data.price}</p>
    </div>
  )
}
```

---

## 5. レンダリング戦略

### SSG vs SSR vs ISR vs CSR

| 戦略 | 実行場所 | 実行タイミング | TTFB | LCP | 用途 |
|------|----------|----------------|------|-----|------|
| **CSR** | クライアント | 実行時 | 80ms | 2,200ms | SPAアプリ |
| **SSR** | サーバー | リクエスト時 | 250ms | 1,200ms | 動的コンテンツ |
| **SSG** | ビルド時 | ビルド時 | 20ms | 500ms | 静的コンテンツ |
| **ISR** | サーバー | 定期的 | 25ms | 520ms | 準静的コンテンツ |

### 使い分けフローチャート

```typescript
function selectStrategy(content: {
  updateFrequency: 'static' | 'hourly' | 'daily' | 'realtime'
  userSpecific: boolean
  seoImportant: boolean
}): 'SSG' | 'ISR' | 'SSR' | 'CSR' {
  // ユーザー固有データ
  if (content.userSpecific) {
    return content.seoImportant ? 'SSR' : 'CSR'
  }

  // 更新頻度に応じて
  switch (content.updateFrequency) {
    case 'static':
      return 'SSG'  // 会社概要、利用規約
    case 'hourly':
    case 'daily':
      return 'ISR'  // ブログ記事、商品詳細
    case 'realtime':
      return 'SSR'  // 株価、チャット
  }
}
```

### 実装パターン

#### 1. SSG（Static Site Generation）

```tsx
// app/about/page.tsx
export default function AboutPage() {
  // ビルド時に生成される静的ページ
  return (
    <div>
      <h1>会社概要</h1>
      <p>私たちは...</p>
    </div>
  )
}
```

#### 2. ISR（Incremental Static Regeneration）

```tsx
// app/blog/[slug]/page.tsx
interface BlogPost {
  title: string
  content: string
  publishedAt: Date
}

async function getPost(slug: string): Promise<BlogPost> {
  const res = await fetch(`https://api.example.com/posts/${slug}`, {
    next: { revalidate: 3600 }  // 1時間ごとに再生成
  })
  return res.json()
}

export default async function BlogPostPage({ params }: { params: { slug: string } }) {
  const post = await getPost(params.slug)

  return (
    <article>
      <h1>{post.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  )
}

// 静的パスの生成
export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts').then(r => r.json())

  return posts.map((post: BlogPost) => ({
    slug: post.slug
  }))
}
```

#### 3. SSR（Server-Side Rendering）

```tsx
// app/dashboard/page.tsx
async function getUserData(userId: string) {
  const res = await fetch(`https://api.example.com/users/${userId}`, {
    cache: 'no-store'  // キャッシュしない（常に最新）
  })
  return res.json()
}

export default async function DashboardPage() {
  const session = await getServerSession()
  const userData = await getUserData(session.user.id)

  return (
    <div>
      <h1>ダッシュボード</h1>
      <UserStats data={userData} />
    </div>
  )
}
```

#### 4. CSR（Client-Side Rendering）

```tsx
// components/RealtimeChart.tsx
'use client'

import { useState, useEffect } from 'react'

export function RealtimeChart() {
  const [data, setData] = useState([])

  useEffect(() => {
    // WebSocketでリアルタイムデータ取得
    const ws = new WebSocket('wss://api.example.com/realtime')

    ws.onmessage = (event) => {
      setData(JSON.parse(event.data))
    }

    return () => ws.close()
  }, [])

  return <Chart data={data} />
}
```

---

## 6. 実践：パフォーマンス改善事例

### Before / After

#### 改善前の状態

```typescript
// 問題点
const issues = {
  LCP: '4.2秒（Poor）',
  INP: '350ms（Needs Improvement）',
  CLS: '0.25（Needs Improvement）',
  原因: [
    '画像が最適化されていない（JPEG、1.5MB）',
    'フォントのブロッキング読み込み',
    '広告のレイアウトシフト',
    '重いJavaScript処理'
  ]
}
```

#### 改善後の状態

```typescript
const improvements = {
  LCP: '1.8秒（Good）- 57%改善',
  INP: '120ms（Good）- 66%改善',
  CLS: '0.05（Good）- 80%改善',
  施策: [
    'Next.js Imageで自動最適化（AVIF、300KB）',
    'next/fontでフォント最適化',
    '広告エリアの固定高さ指定',
    'useTransitionで処理の優先度制御'
  ]
}
```

### 実装コード

```tsx
// app/products/[id]/page.tsx
import Image from 'next/image'
import { Suspense } from 'react'

async function getProduct(id: string) {
  const res = await fetch(`https://api.example.com/products/${id}`, {
    next: { revalidate: 3600 }  // ISR: 1時間
  })
  return res.json()
}

export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await getProduct(params.id)

  return (
    <div>
      {/* LCP改善: priority指定 */}
      <Image
        src={product.image}
        alt={product.name}
        width={600}
        height={600}
        priority
        quality={75}
      />

      <h1>{product.name}</h1>
      <p>{product.description}</p>

      {/* INP改善: Suspenseで遅延ロード */}
      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews productId={params.id} />
      </Suspense>
    </div>
  )
}

// CLS改善: Skeleton UI
function ReviewsSkeleton() {
  return (
    <div style={{ minHeight: '200px' }}>
      <div className="skeleton" />
    </div>
  )
}
```

---

## まとめ

この章では、Core Web Vitalsとレンダリング最適化を学びました。

**重要ポイント**:
- ✅ LCP < 2.5秒、INP < 200ms、CLS < 0.1 を目指す
- ✅ Next.js Image、next/fontで自動最適化
- ✅ SSG/ISR/SSRを適切に使い分ける
- ✅ Skeleton UIでレイアウトシフトを防ぐ
- ✅ useTransitionで処理の優先度を制御

**次章予告**: Chapter 7では、Webアクセシビリティを学びます。

---

**参考リンク**:
- [Web Vitals](https://web.dev/vitals/)
- [Next.js Performance](https://nextjs.org/docs/app/building-your-application/optimizing)
- [PageSpeed Insights](https://pagespeed.web.dev/)
