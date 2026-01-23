---
title: "SSR vs CSR vs SSG"
---

# SSR vs CSR vs SSG

## この章で学ぶこと

- 各レンダリング手法の特徴
- 適切な使い分け
- パフォーマンスへの影響
- Next.jsでの実装方法
- ハイブリッド戦略

## レンダリング手法の比較

### CSR (Client-Side Rendering)

**特徴:**
- ブラウザでJavaScriptを実行してレンダリング
- SPAの標準的な手法

**メリット:**
- リッチなインタラクション
- サーバー負荷が低い

**デメリット:**
- 初回表示が遅い
- SEOが困難
- JavaScriptが必須

**実装例:**

```typescript
// React CSR
function App() {
  const [data, setData] = useState(null)

  useEffect(() => {
    fetch('/api/data')
      .then(res => res.json())
      .then(setData)
  }, [])

  if (!data) return <Loading />
  return <div>{data.content}</div>
}
```

### SSR (Server-Side Rendering)

**特徴:**
- サーバーでHTMLを生成
- リクエストごとにレンダリング

**メリット:**
- 初回表示が高速
- SEO対応
- JavaScriptなしでも表示可能

**デメリット:**
- サーバー負荷が高い
- TTFBが長くなる可能性

**実装例:**

```typescript
// Next.js SSR
export async function getServerSideProps() {
  const data = await fetchData()
  return { props: { data } }
}

export default function Page({ data }) {
  return <div>{data.content}</div>
}
```

### SSG (Static Site Generation)

**特徴:**
- ビルド時にHTMLを生成
- 静的ファイルとして配信

**メリット:**
- 最速の表示速度
- SEO対応
- サーバー負荷ゼロ

**デメリット:**
- データ更新に再ビルドが必要
- 動的コンテンツには不向き

**実装例:**

```typescript
// Next.js SSG
export async function getStaticProps() {
  const data = await fetchData()
  return {
    props: { data },
    revalidate: 60, // ISR: 60秒ごとに再生成
  }
}

export default function Page({ data }) {
  return <div>{data.content}</div>
}
```

### ISR (Incremental Static Regeneration)

**特徴:**
- SSGとSSRのハイブリッド
- 静的ページを定期的に再生成

**メリット:**
- SSGの高速性
- データの鮮度を保持

**実装例:**

```typescript
export async function getStaticProps() {
  const data = await fetchData()
  return {
    props: { data },
    revalidate: 300, // 5分ごとに再生成
  }
}
```

## パフォーマンス比較

| 手法 | TTFB | FCP | TTI | SEO |
|------|------|-----|-----|-----|
| CSR | 速い | 遅い | 遅い | △ |
| SSR | 中程度 | 速い | 中程度 | ◯ |
| SSG | 最速 | 最速 | 速い | ◯ |
| ISR | 最速 | 最速 | 速い | ◯ |

## 適切な使い分け

### SSGが適している場合

- ブログ記事
- ドキュメントサイト
- ランディングページ
- 商品カタログ

```typescript
// ブログ記事
export async function getStaticPaths() {
  const posts = await getAllPosts()
  return {
    paths: posts.map(post => ({ params: { slug: post.slug } })),
    fallback: 'blocking',
  }
}

export async function getStaticProps({ params }) {
  const post = await getPost(params.slug)
  return {
    props: { post },
    revalidate: 3600, // 1時間ごとに再生成
  }
}
```

### SSRが適している場合

- ダッシュボード
- ユーザー固有のページ
- リアルタイムデータ

```typescript
// ユーザーダッシュボード
export async function getServerSideProps(context) {
  const session = await getSession(context)
  const userData = await getUserData(session.userId)
  return { props: { userData } }
}
```

### CSRが適している場合

- 管理画面
- インタラクティブなアプリ
- ログイン後のコンテンツ

```typescript
// 管理画面
function AdminDashboard() {
  const { data, loading } = useSWR('/api/admin/stats', fetcher)

  if (loading) return <Loading />
  return <Dashboard data={data} />
}
```

## ハイブリッド戦略

### Next.js App Router

```typescript
// app/page.tsx (SSG)
export default async function HomePage() {
  const posts = await getPosts()
  return <PostList posts={posts} />
}

// app/dashboard/page.tsx (Dynamic)
export const dynamic = 'force-dynamic'

export default async function DashboardPage() {
  const session = await getSession()
  const data = await getUserData(session.userId)
  return <Dashboard data={data} />
}

// app/blog/[slug]/page.tsx (ISR)
export const revalidate = 3600

export default async function BlogPost({ params }) {
  const post = await getPost(params.slug)
  return <Post data={post} />
}
```

### Streaming SSR

```typescript
import { Suspense } from 'react'

export default function Page() {
  return (
    <div>
      <h1>Page Title</h1>
      {/* 即座にレンダリング */}

      <Suspense fallback={<Skeleton />}>
        <SlowComponent />
        {/* 準備できたらストリーミング */}
      </Suspense>
    </div>
  )
}
```

## 実装例

### CSR + SWR

```typescript
import useSWR from 'swr'

function Profile() {
  const { data, error, isLoading } = useSWR('/api/user', fetcher, {
    revalidateOnFocus: true,
    revalidateOnReconnect: true,
  })

  if (isLoading) return <Skeleton />
  if (error) return <Error />
  return <div>{data.name}</div>
}
```

### SSG + CSR

```typescript
// ビルド時に基本データを取得（SSG）
export async function getStaticProps() {
  const initialData = await fetchInitialData()
  return { props: { initialData }, revalidate: 60 }
}

// クライアントで追加データを取得（CSR）
export default function Page({ initialData }) {
  const { data } = useSWR('/api/additional', fetcher, {
    fallbackData: initialData,
  })

  return <Content data={data} />
}
```

## 改善事例

### 事例: ニュースサイト

**Before (CSR):**
- FCP: 2.8秒
- TTI: 4.2秒
- SEO: 検索結果に表示されない

**After (SSG + ISR):**

```typescript
export async function getStaticProps() {
  const articles = await fetchArticles()
  return {
    props: { articles },
    revalidate: 60, // 1分ごとに再生成
  }
}
```

**結果:**
- FCP: 2.8秒 → 0.8秒 (71%改善)
- TTI: 4.2秒 → 1.2秒 (71%改善)
- SEO: Google検索1ページ目に表示
- オーガニックトラフィック: +340%

## まとめ

レンダリング手法の選択ポイント:

1. **SSG/ISR優先**
   - 最も高速
   - SEO対応
   - サーバー負荷ゼロ

2. **SSRは必要な場合のみ**
   - ユーザー固有データ
   - リアルタイム性が重要

3. **CSRは補完的に**
   - インタラクティブ機能
   - ログイン後のコンテンツ

4. **ハイブリッド戦略**
   - ページごとに最適な手法を選択

次の章では、Reactのパフォーマンスパターンについて学びます。
