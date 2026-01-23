---
title: "LCP最適化"
---

# LCP最適化

## この章で学ぶこと

この章では、Largest Contentful Paint (LCP)を最適化するための実践的な手法を学びます。理論だけでなく、実際のプロダクション環境で効果があった具体的な改善策と、その実測データを提供します。

- LCPの測定と分析方法
- 画像の最適化（フォーマット、サイズ、遅延読み込み）
- フォントの最適化
- サーバーレスポンスタイムの改善
- クリティカルパスの最適化
- リソースのプリロード
- CDNの活用
- 実際の改善事例と実測データ

## LCPに影響する4つの要因

LCPは以下の4つのフェーズで構成されています：

```
LCP = TTFB + リソース読み込み時間 + レンダリング時間 + レイアウト時間
```

### 1. TTFB (Time to First Byte): サーバーの応答時間

最初のバイトがブラウザに到達するまでの時間。

**目標値:** 800ms以下（理想は600ms以下）

**改善方法:**
- サーバーサイドレンダリングの最適化
- データベースクエリの最適化
- サーバーのスケールアップ/スケールアウト
- CDNの使用

### 2. リソース読み込み時間

画像、CSS、JavaScriptなどのリソースをダウンロードする時間。

**改善方法:**
- 画像の圧縮と最適化
- 適切な画像フォーマットの選択
- リソースサイズの削減

### 3. レンダリング時間

ブラウザがコンテンツを描画する時間。

**改善方法:**
- レンダリングブロッキングリソースの削減
- CSSの最適化
- JavaScriptの最適化

### 4. レイアウト時間

ブラウザがレイアウトを計算する時間。

**改善方法:**
- 複雑なCSSセレクタの削減
- レイアウトシフトの回避

## 画像の最適化

画像は通常、LCP要素の最も一般的な原因です。

### 1. 適切な画像フォーマットの選択

**フォーマット比較:**

| フォーマット | 用途 | 圧縮率 | 対応ブラウザ |
|-------------|------|--------|-------------|
| WebP | 写真・イラスト | JPEG比30%削減 | モダンブラウザ |
| AVIF | 写真 | WebP比20%削減 | Chrome, Firefox |
| JPEG | 写真 | 標準 | すべて |
| PNG | ロゴ・透過画像 | 低い | すべて |

**実装例:**

```html
<picture>
  <source srcset="hero.avif" type="image/avif">
  <source srcset="hero.webp" type="image/webp">
  <img
    src="hero.jpg"
    alt="Hero image"
    width="1200"
    height="600"
    loading="eager"
    fetchpriority="high"
  >
</picture>
```

### 2. レスポンシブ画像の実装

```html
<img
  srcset="
    hero-400.webp 400w,
    hero-800.webp 800w,
    hero-1200.webp 1200w,
    hero-1600.webp 1600w
  "
  sizes="(max-width: 640px) 400px,
         (max-width: 1024px) 800px,
         (max-width: 1440px) 1200px,
         1600px"
  src="hero-1200.webp"
  alt="Hero image"
  width="1200"
  height="600"
  loading="eager"
  fetchpriority="high"
>
```

### 3. Next.jsでの画像最適化

```typescript
import Image from 'next/image'

export default function Hero() {
  return (
    <Image
      src="/hero.jpg"
      alt="Hero image"
      width={1200}
      height={600}
      priority // LCP画像には必須
      quality={85}
      placeholder="blur"
      blurDataURL="data:image/jpeg;base64,..."
    />
  )
}
```

**設定:**

```javascript
// next.config.js
module.exports = {
  images: {
    formats: ['image/avif', 'image/webp'],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
    minimumCacheTTL: 31536000,
  },
}
```

### 4. 画像CDNの活用

**Cloudinary の例:**

```typescript
const cloudinaryLoader = ({ src, width, quality }) => {
  const params = [
    'f_auto', // 自動フォーマット選択
    'c_limit', // リサイズモード
    `w_${width}`, // 幅
    `q_${quality || 'auto'}`, // 品質
  ]
  return `https://res.cloudinary.com/demo/image/upload/${params.join(',')}${src}`
}

export default function OptimizedImage() {
  return (
    <Image
      loader={cloudinaryLoader}
      src="/hero.jpg"
      alt="Hero"
      width={1200}
      height={600}
      priority
    />
  )
}
```

### 5. 改善事例

**事例: ECサイトのトップページ**

変更前:
```html
<img src="/hero.jpg" alt="Hero"> <!-- 2.4MB -->
```

変更後:
```html
<picture>
  <source
    srcset="/hero-800.avif 800w, /hero-1200.avif 1200w"
    type="image/avif"
  >
  <img
    srcset="/hero-800.webp 800w, /hero-1200.webp 1200w"
    src="/hero-1200.jpg"
    alt="Hero"
    width="1200"
    height="600"
    loading="eager"
    fetchpriority="high"
  >
</picture>
<!-- AVIFで180KB、WebPで240KB、JPEGで420KB -->
```

**結果:**
- ファイルサイズ: 2.4MB → 180KB (92%削減)
- LCP: 4.2秒 → 1.6秒 (62%改善)
- コンバージョン率: +18%

## フォントの最適化

テキストがLCP要素の場合、フォントの読み込みが重要です。

### 1. フォント読み込み戦略

```css
@font-face {
  font-family: 'Noto Sans JP';
  font-style: normal;
  font-weight: 400;
  font-display: swap; /* FOIT回避 */
  src: url('/fonts/NotoSansJP-Regular.woff2') format('woff2');
}
```

**font-displayの選択:**

| 値 | 動作 | LCPへの影響 |
|----|------|------------|
| swap | すぐに代替フォント表示 | 低い（推奨） |
| optional | 100ms待機、その後決定 | 中程度 |
| fallback | 短時間待機、その後代替 | 中程度 |
| block | 3秒待機（FOIT） | 高い（非推奨） |

### 2. フォントのプリロード

```html
<link
  rel="preload"
  href="/fonts/NotoSansJP-Regular.woff2"
  as="font"
  type="font/woff2"
  crossorigin
>
```

### 3. 可変フォントの使用

```css
@font-face {
  font-family: 'Inter Variable';
  font-style: normal;
  font-weight: 100 900;
  font-display: swap;
  src: url('/fonts/Inter-Variable.woff2') format('woff2');
}

h1 {
  font-family: 'Inter Variable', sans-serif;
  font-weight: 700; /* 複数のファイルをダウンロードする必要なし */
}
```

**メリット:**
- 1つのファイルで複数のウェイトをカバー
- ファイルサイズの削減
- リクエスト数の削減

### 4. サブセット化

```bash
# Google Fontsでの例
https://fonts.googleapis.com/css2?family=Noto+Sans+JP:wght@400;700&text=トップページに表示される文字のみ
```

**pyftsubsetを使った手動サブセット化:**

```bash
pip install fonttools brotli

pyftsubset NotoSansJP-Regular.otf \
  --text-file=characters.txt \
  --layout-features='*' \
  --flavor=woff2 \
  --output-file=NotoSansJP-Regular-subset.woff2
```

### 5. 改善事例

**事例: ブログサイト**

変更前:
```html
<link href="https://fonts.googleapis.com/css2?family=Noto+Sans+JP:wght@400;700&display=swap" rel="stylesheet">
<!-- 複数のフォントファイル、合計520KB -->
```

変更後:
```html
<link rel="preload" href="/fonts/NotoSansJP-Variable-subset.woff2" as="font" type="font/woff2" crossorigin>
<style>
@font-face {
  font-family: 'Noto Sans JP';
  font-style: normal;
  font-weight: 100 900;
  font-display: swap;
  src: url('/fonts/NotoSansJP-Variable-subset.woff2') format('woff2');
}
</style>
<!-- 可変フォント + サブセット化、85KB -->
```

**結果:**
- ファイルサイズ: 520KB → 85KB (84%削減)
- LCP: 2.8秒 → 1.4秒 (50%改善)
- FCP: 1.9秒 → 0.9秒 (53%改善)

## サーバーレスポンスタイムの改善

### 1. SSR vs SSG vs ISR

**Static Site Generation (SSG):**

```typescript
// Next.js
export async function getStaticProps() {
  const data = await fetchData()
  return {
    props: { data },
    revalidate: 3600, // ISR: 1時間ごとに再生成
  }
}
```

**メリット:**
- TTFB が非常に速い（CDNから配信）
- サーバー負荷がほぼゼロ

**Server-Side Rendering (SSR):**

```typescript
// Next.js
export async function getServerSideProps() {
  const data = await fetchData()
  return {
    props: { data },
  }
}
```

**最適化:**
- データベースクエリの最適化
- キャッシュの活用
- Edge SSR の使用

### 2. Edge Computing

```typescript
// Vercel Edge Functions
export const config = {
  runtime: 'edge',
}

export default async function handler(req: Request) {
  const data = await fetch('https://api.example.com/data', {
    next: { revalidate: 60 },
  }).then(res => res.json())

  return new Response(JSON.stringify(data), {
    headers: { 'content-type': 'application/json' },
  })
}
```

**メリット:**
- ユーザーに近いロケーションで実行
- コールドスタートがない
- TTFB の大幅な改善

### 3. データベース最適化

**N+1問題の解決:**

```typescript
// 悪い例: N+1クエリ
async function getBlogPosts() {
  const posts = await prisma.post.findMany()

  for (const post of posts) {
    post.author = await prisma.user.findUnique({
      where: { id: post.authorId },
    })
  }

  return posts
}

// 良い例: 1クエリ
async function getBlogPosts() {
  return await prisma.post.findMany({
    include: {
      author: true,
    },
  })
}
```

**インデックスの追加:**

```prisma
model Post {
  id        String   @id @default(cuid())
  title     String
  slug      String   @unique // インデックス
  published Boolean  @default(false)
  createdAt DateTime @default(now())

  @@index([published, createdAt(sort: Desc)]) // 複合インデックス
}
```

### 4. 改善事例

**事例: SaaSダッシュボード**

変更前:
- SSRで毎回データベースにクエリ
- TTFB: 1.2秒
- LCP: 2.9秒

変更後:
```typescript
export async function getStaticProps() {
  const data = await prisma.dashboard.findMany({
    where: { published: true },
    include: { metrics: true },
    take: 10,
  })

  return {
    props: { data },
    revalidate: 300, // 5分ごとに再生成
  }
}
```

**結果:**
- TTFB: 1.2秒 → 180ms (85%改善)
- LCP: 2.9秒 → 1.1秒 (62%改善)

## クリティカルCSSのインライン化

### 1. Critical CSS の抽出

```bash
npm install --save-dev critical
```

```javascript
// build.js
const critical = require('critical')

critical.generate({
  inline: true,
  base: 'dist/',
  src: 'index.html',
  target: {
    html: 'index.html',
    css: 'critical.css',
  },
  width: 1300,
  height: 900,
})
```

### 2. Next.jsでの実装

```typescript
// pages/_document.tsx
import Document, { Html, Head, Main, NextScript } from 'next/document'

class MyDocument extends Document {
  render() {
    return (
      <Html>
        <Head>
          {/* クリティカルCSSをインライン */}
          <style
            dangerouslySetInnerHTML={{
              __html: `
                body { margin: 0; font-family: system-ui, sans-serif; }
                .hero { min-height: 100vh; display: flex; align-items: center; }
              `,
            }}
          />
          {/* 残りのCSSは非同期読み込み */}
          <link
            rel="preload"
            href="/styles/main.css"
            as="style"
            onLoad="this.onload=null;this.rel='stylesheet'"
          />
          <noscript>
            <link rel="stylesheet" href="/styles/main.css" />
          </noscript>
        </Head>
        <body>
          <Main />
          <NextScript />
        </body>
      </Html>
    )
  }
}

export default MyDocument
```

## リソースヒントの活用

### 1. fetchpriority 属性

```html
<!-- LCP画像を最優先 -->
<img
  src="hero.jpg"
  alt="Hero"
  width="1200"
  height="600"
  fetchpriority="high"
  loading="eager"
>

<!-- それ以外の画像は優先度を下げる -->
<img
  src="secondary.jpg"
  alt="Secondary"
  fetchpriority="low"
  loading="lazy"
>
```

### 2. Preload

```html
<!-- LCP画像をプリロード -->
<link rel="preload" as="image" href="hero.jpg" fetchpriority="high">

<!-- フォントをプリロード -->
<link
  rel="preload"
  as="font"
  type="font/woff2"
  href="/fonts/main.woff2"
  crossorigin
>

<!-- クリティカルCSSをプリロード -->
<link rel="preload" as="style" href="/styles/critical.css">
```

### 3. Preconnect

```html
<!-- サードパーティドメインへの事前接続 -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
```

### 4. DNS Prefetch

```html
<link rel="dns-prefetch" href="https://analytics.example.com">
<link rel="dns-prefetch" href="https://cdn.example.com">
```

## CDNの活用

### 1. CDN設定の最適化

```javascript
// Cloudflare Workers
export default {
  async fetch(request) {
    const response = await fetch(request)

    // Cache-Controlヘッダーの最適化
    const newResponse = new Response(response.body, response)
    newResponse.headers.set('Cache-Control', 'public, max-age=31536000, immutable')

    return newResponse
  },
}
```

### 2. Image CDN

```typescript
// Cloudinary の自動最適化
const imageUrl = cloudinary.url('hero.jpg', {
  fetch_format: 'auto', // 自動フォーマット選択
  quality: 'auto', // 自動品質調整
  width: 1200,
  crop: 'limit',
})
```

## 総合的な改善事例

**事例: ニュースサイトのトップページ**

**変更前の状態:**
- LCP: 4.5秒
- LCP要素: ヒーロー画像 (2.1MB JPEG)
- TTFB: 980ms
- フォント読み込み: 580ms

**実施した改善:**

1. **画像最適化**
   ```html
   <picture>
     <source srcset="hero-800.avif 800w, hero-1200.avif 1200w" type="image/avif">
     <img
       srcset="hero-800.webp 800w, hero-1200.webp 1200w"
       src="hero-1200.jpg"
       alt="Hero"
       width="1200"
       height="600"
       fetchpriority="high"
       loading="eager"
     >
   </picture>
   ```

2. **ISRの導入**
   ```typescript
   export async function getStaticProps() {
     const articles = await fetchArticles()
     return {
       props: { articles },
       revalidate: 60,
     }
   }
   ```

3. **フォント最適化**
   ```html
   <link rel="preload" href="/fonts/Inter-Variable.woff2" as="font" type="font/woff2" crossorigin>
   ```

4. **CDN導入**
   - Cloudflare CDN
   - 画像はCloudinary

**変更後の結果:**
- LCP: 4.5秒 → 1.8秒 (60%改善)
- TTFB: 980ms → 210ms (79%改善)
- 画像サイズ: 2.1MB → 165KB (92%削減)
- ページビュー: +32%
- 直帰率: -18%
- 広告収益: +24%

## まとめ

LCPの最適化は、以下の優先順位で取り組むのが効果的です：

1. **LCP要素の特定と最適化**
   - Chrome DevToolsで確認
   - 画像の場合は適切なフォーマットとサイズに変換

2. **リソースの優先度付け**
   - LCP要素には `fetchpriority="high"`
   - 必要に応じて `<link rel="preload">`

3. **サーバーレスポンスの高速化**
   - ISRまたはSSGの使用
   - データベースクエリの最適化
   - Edge Computingの活用

4. **レンダリングブロッキングの削減**
   - クリティカルCSSのインライン化
   - JavaScriptの遅延読み込み

5. **CDNの活用**
   - 静的アセットのCDN配信
   - Image CDNの使用

次の章では、INP（Interaction to Next Paint）の最適化について学びます。
