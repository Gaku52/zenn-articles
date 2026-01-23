---
title: "CLS最適化"
---

# CLS最適化

## この章で学ぶこと

Cumulative Layout Shift (CLS) は、ページの視覚的な安定性を測定する指標です。予期しないレイアウトシフトはユーザー体験を大きく損なうため、CLSの最適化は非常に重要です。

- CLSの計算方法と測定
- 画像・動画のサイズ指定
- フォント読み込み時のレイアウトシフト対策
- 動的コンテンツの挿入方法
- 広告・埋め込みコンテンツの最適化
- アニメーションの適切な実装
- 実際の改善事例

## CLSの計算方法

```
CLS = 影響を受けたフレームの影響度 × 移動距離
```

### Impact Fraction（影響度）

ビューポート内で移動した要素が占める面積の割合。

```
Impact Fraction = 移動前と移動後の領域の和 / ビューポート面積
```

### Distance Fraction（移動距離）

要素が移動した距離の割合。

```
Distance Fraction = 移動距離 / ビューポート高さ
```

### 目標値

- **Good:** 0.1以下
- **Needs Improvement:** 0.1〜0.25
- **Poor:** 0.25以上

## CLSの測定方法

### 1. Chrome DevTools

```javascript
// Layout Shift のイベントを監視
let clsScore = 0

const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (!entry.hadRecentInput) {
      clsScore += entry.value
      console.log('CLS:', clsScore)
      console.log('Shifted elements:', entry.sources)
    }
  }
})

observer.observe({ type: 'layout-shift', buffered: true })
```

### 2. web-vitals ライブラリ

```typescript
import { onCLS } from 'web-vitals'

onCLS((metric) => {
  console.log('CLS:', metric.value)

  // 詳細情報
  if (metric.attribution) {
    console.log('Largest shift value:', metric.attribution.largestShiftValue)
    console.log('Largest shift time:', metric.attribution.largestShiftTime)
    console.log('Largest shift target:', metric.attribution.largestShiftTarget)
    console.log('Load state:', metric.attribution.loadState)
  }
})
```

### 3. Lighthouse

```bash
lighthouse https://example.com --only-categories=performance --view
```

## 画像・動画のサイズ指定

### 1. 画像のwidth/height属性

**悪い例:**

```html
<!-- サイズ指定なし → CLS発生 -->
<img src="product.jpg" alt="Product">
```

**良い例:**

```html
<!-- サイズ指定あり → CLSなし -->
<img
  src="product.jpg"
  alt="Product"
  width="800"
  height="600"
  style="max-width: 100%; height: auto;"
>
```

### 2. aspect-ratio の使用

```html
<img
  src="product.jpg"
  alt="Product"
  width="800"
  height="600"
  style="aspect-ratio: 800/600; width: 100%; height: auto;"
>
```

```css
/* CSSでも指定可能 */
.product-image {
  aspect-ratio: 16 / 9;
  width: 100%;
  height: auto;
}
```

### 3. Next.js Image コンポーネント

```typescript
import Image from 'next/image'

export default function ProductImage() {
  return (
    <Image
      src="/product.jpg"
      alt="Product"
      width={800}
      height={600}
      style={{
        maxWidth: '100%',
        height: 'auto',
      }}
    />
  )
}
```

### 4. レスポンシブ画像のサイズ指定

```html
<picture>
  <source
    media="(max-width: 640px)"
    srcset="image-small.jpg"
    width="400"
    height="300"
  >
  <source
    media="(max-width: 1024px)"
    srcset="image-medium.jpg"
    width="800"
    height="600"
  >
  <img
    src="image-large.jpg"
    alt="Responsive image"
    width="1200"
    height="900"
    style="max-width: 100%; height: auto;"
  >
</picture>
```

### 5. 動画のサイズ指定

```html
<!-- 悪い例 -->
<video src="video.mp4"></video>

<!-- 良い例 -->
<video
  src="video.mp4"
  width="1280"
  height="720"
  style="aspect-ratio: 16/9; max-width: 100%; height: auto;"
  poster="thumbnail.jpg"
></video>
```

## フォント読み込み時のレイアウトシフト対策

### 1. font-display: swap の使用

```css
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
  font-display: swap; /* FOIT を回避 */
}
```

### 2. サイズ調整（size-adjust）

```css
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
  font-display: swap;
  /* フォールバックフォントとサイズを合わせる */
  ascent-override: 90%;
  descent-override: 22%;
  line-gap-override: 0%;
  size-adjust: 107%;
}
```

### 3. Fallback Font の最適化

```typescript
// Fontaine を使用して自動調整
import { FontaineTransform } from 'fontaine'

export default {
  plugins: [
    FontaineTransform.vite({
      fonts: [
        {
          family: 'Inter',
          src: '/fonts/Inter-Variable.woff2',
          fallbacks: ['system-ui', 'sans-serif'],
        },
      ],
    }),
  ],
}
```

### 4. next/font の使用

```typescript
import { Inter } from 'next/font/google'

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  adjustFontFallback: true, // 自動でフォールバックを調整
})

export default function RootLayout({ children }) {
  return (
    <html lang="en" className={inter.className}>
      <body>{children}</body>
    </html>
  )
}
```

### 5. 改善事例

**変更前:**
```css
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
  font-display: block; /* 3秒待機 → CLS発生 */
}
```
- CLS: 0.28
- フォント読み込みまで空白

**変更後:**
```css
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
  font-display: swap;
  ascent-override: 90%;
  descent-override: 22%;
  size-adjust: 107%;
}
```
- CLS: 0.03 (90%改善)
- すぐにフォールバックフォントで表示

## 動的コンテンツの挿入

### 1. スペースの予約

**悪い例:**

```typescript
function Banner() {
  const [show, setShow] = useState(false)

  useEffect(() => {
    setTimeout(() => setShow(true), 1000)
  }, [])

  // バナーが突然表示される → CLS発生
  if (!show) return null

  return <div className="banner">Important message!</div>
}
```

**良い例:**

```typescript
function Banner() {
  const [show, setShow] = useState(false)

  useEffect(() => {
    setTimeout(() => setShow(true), 1000)
  }, [])

  // 最初からスペースを確保
  return (
    <div
      className="banner"
      style={{
        minHeight: '60px',
        opacity: show ? 1 : 0,
        visibility: show ? 'visible' : 'hidden',
      }}
    >
      {show && 'Important message!'}
    </div>
  )
}
```

### 2. Skeleton UI の使用

```typescript
function ProductList() {
  const { data, loading } = useProducts()

  if (loading) {
    return (
      <div>
        {[...Array(6)].map((_, i) => (
          <ProductSkeleton key={i} />
        ))}
      </div>
    )
  }

  return (
    <div>
      {data.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  )
}

function ProductSkeleton() {
  return (
    <div className="skeleton" style={{ height: '300px' }}>
      <div className="skeleton-image" style={{ height: '200px' }} />
      <div className="skeleton-title" style={{ height: '24px' }} />
      <div className="skeleton-price" style={{ height: '20px' }} />
    </div>
  )
}
```

### 3. transform の使用（position変更の回避）

**悪い例:**

```css
/* position を変更 → レイアウトシフト */
.modal {
  position: fixed;
  top: 0;
  left: 0;
}

.modal.open {
  top: 50%;
  left: 50%;
}
```

**良い例:**

```css
/* transform を使用 → レイアウトシフトなし */
.modal {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%) scale(0);
  transition: transform 0.3s;
}

.modal.open {
  transform: translate(-50%, -50%) scale(1);
}
```

## 広告・埋め込みコンテンツの最適化

### 1. 広告スロットのサイズ予約

```html
<!-- 悪い例: サイズ未指定 -->
<div id="ad-slot"></div>

<!-- 良い例: 最小サイズを確保 -->
<div
  id="ad-slot"
  style="min-height: 250px; display: flex; align-items: center; justify-content: center;"
>
  <div class="ad-loading">Loading...</div>
</div>
```

### 2. Google AdSense の最適化

```html
<ins
  class="adsbygoogle"
  style="display: inline-block; min-height: 280px; width: 336px;"
  data-ad-client="ca-pub-xxxxx"
  data-ad-slot="xxxxx"
></ins>
<script>
  (adsbygoogle = window.adsbygoogle || []).push({})
</script>
```

### 3. YouTube埋め込みの最適化

**悪い例:**

```html
<iframe
  src="https://www.youtube.com/embed/VIDEO_ID"
  frameborder="0"
></iframe>
```

**良い例:**

```html
<div style="aspect-ratio: 16/9; position: relative;">
  <iframe
    src="https://www.youtube.com/embed/VIDEO_ID"
    style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;"
    frameborder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
  ></iframe>
</div>
```

### 4. React での実装

```typescript
function YouTubeEmbed({ videoId }: { videoId: string }) {
  return (
    <div
      style={{
        aspectRatio: '16/9',
        position: 'relative',
        width: '100%',
      }}
    >
      <iframe
        src={`https://www.youtube.com/embed/${videoId}`}
        style={{
          position: 'absolute',
          top: 0,
          left: 0,
          width: '100%',
          height: '100%',
          border: 0,
        }}
        allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
        allowFullScreen
      />
    </div>
  )
}
```

### 5. Twitter埋め込みの最適化

```typescript
import { useEffect, useRef } from 'react'

function TwitterEmbed({ tweetId }: { tweetId: string }) {
  const containerRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    // 最小サイズを確保してからスクリプトを読み込み
    if (containerRef.current) {
      containerRef.current.style.minHeight = '400px'
    }

    const script = document.createElement('script')
    script.src = 'https://platform.twitter.com/widgets.js'
    script.async = true
    document.body.appendChild(script)

    return () => {
      document.body.removeChild(script)
    }
  }, [])

  return (
    <div ref={containerRef}>
      <blockquote className="twitter-tweet">
        <a href={`https://twitter.com/x/status/${tweetId}`}>Loading tweet...</a>
      </blockquote>
    </div>
  )
}
```

## アニメーションの適切な実装

### 1. transform と opacity のみ使用

**悪い例（レイアウトシフト発生）:**

```css
.box {
  transition: width 0.3s, height 0.3s, top 0.3s, left 0.3s;
}

.box:hover {
  width: 300px;
  height: 300px;
  top: 50px;
  left: 50px;
}
```

**良い例（レイアウトシフトなし）:**

```css
.box {
  transition: transform 0.3s, opacity 0.3s;
}

.box:hover {
  transform: scale(1.2) translate(10px, 10px);
  opacity: 0.8;
}
```

### 2. will-change の適切な使用

```css
.animated-element {
  /* アニメーション直前に指定 */
  will-change: transform;
}

.animated-element.animating {
  transform: translateX(100px);
}

.animated-element.done {
  /* アニメーション後は削除 */
  will-change: auto;
}
```

### 3. content-visibility の使用

```css
.long-content {
  /* オフスクリーンのコンテンツをスキップ */
  content-visibility: auto;
  contain-intrinsic-size: 0 500px; /* 推定サイズを指定 */
}
```

## 実際の改善事例

### 事例1: ニュースサイト

**問題:**
- CLS: 0.42
- 広告が読み込まれるたびにコンテンツが移動
- 画像のサイズ未指定

**実施した改善:**

```html
<!-- 1. 画像サイズの指定 -->
<img
  src="article.jpg"
  alt="Article"
  width="800"
  height="600"
  style="aspect-ratio: 800/600; width: 100%; height: auto;"
>

<!-- 2. 広告スロットのサイズ予約 -->
<div class="ad-container" style="min-height: 250px;">
  <div id="ad-slot"></div>
</div>

<!-- 3. フォントの最適化 -->
<style>
@font-face {
  font-family: 'ArticleFont';
  src: url('/fonts/article.woff2') format('woff2');
  font-display: swap;
  size-adjust: 105%;
}
</style>
```

**結果:**
- CLS: 0.42 → 0.05 (88%改善)
- セッション時間: +41%
- ページビュー/セッション: +35%
- 広告クリック率: +22%

### 事例2: ECサイトの商品ページ

**問題:**
- CLS: 0.31
- 商品画像の遅延読み込みでレイアウトシフト
- レビューセクションの突然の表示

**実施した改善:**

```typescript
// 1. Next.js Image で自動最適化
<Image
  src={product.image}
  alt={product.name}
  width={600}
  height={600}
  priority={index < 3} // ファーストビューのみ
  sizes="(max-width: 768px) 100vw, 600px"
/>

// 2. Skeleton UI の実装
{loading ? (
  <ReviewSkeleton count={3} />
) : (
  <ReviewList reviews={reviews} />
)}

// 3. aspect-ratio でスペース確保
<div style={{ aspectRatio: '1/1', width: '100%' }}>
  <Image src={image} fill alt="Product" />
</div>
```

**結果:**
- CLS: 0.31 → 0.06 (81%改善)
- 直帰率: -12%
- コンバージョン率: +8%

### 事例3: ブログサイト

**問題:**
- CLS: 0.38
- Webフォント読み込み時のテキスト移動
- Twitter埋め込みのレイアウトシフト

**実施した改善:**

```typescript
// 1. next/font で自動最適化
import { Inter } from 'next/font/google'

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  adjustFontFallback: true,
})

// 2. Twitter埋め込みのサイズ予約
<div style={{ minHeight: '500px' }}>
  <TwitterEmbed tweetId={id} />
</div>

// 3. 画像の遅延読み込みを適切に
<Image
  src={post.thumbnail}
  alt={post.title}
  width={800}
  height={400}
  loading={index < 2 ? 'eager' : 'lazy'}
  priority={index === 0}
/>
```

**結果:**
- CLS: 0.38 → 0.04 (89%改善)
- セッション時間: +28%
- ページビュー: +19%

## CLS最適化のチェックリスト

### 画像・動画
- [ ] すべての画像に width/height 属性を指定
- [ ] aspect-ratio を使用してスペースを確保
- [ ] 適切な loading 属性（eager/lazy）を設定
- [ ] 動画にも width/height とポスター画像を指定

### フォント
- [ ] font-display: swap を使用
- [ ] size-adjust でフォールバックを調整
- [ ] next/font や Fontaine で自動最適化
- [ ] クリティカルなテキストはシステムフォントを検討

### 動的コンテンツ
- [ ] コンテンツ挿入前にスペースを確保
- [ ] Skeleton UI を実装
- [ ] transform/opacity のみでアニメーション
- [ ] content-visibility を適切に使用

### 広告・埋め込み
- [ ] 広告スロットの最小サイズを指定
- [ ] iframe に aspect-ratio を使用
- [ ] サードパーティコンテンツの遅延読み込み

### 測定・監視
- [ ] Layout Shift Events を監視
- [ ] web-vitals でCLSを測定
- [ ] Lighthouse で定期的にチェック
- [ ] Real User Monitoring で継続監視

## まとめ

CLSを最適化するための重要なポイント:

1. **すべてのメディアにサイズ指定**
   - width/height 属性
   - aspect-ratio プロパティ

2. **フォント読み込みの最適化**
   - font-display: swap
   - size-adjust での調整

3. **動的コンテンツのスペース確保**
   - Skeleton UI
   - 最小サイズの指定

4. **transform/opacity のみでアニメーション**
   - width/height/top/left は使わない

5. **広告・埋め込みコンテンツの適切な処理**
   - 最小サイズの確保
   - aspect-ratio の使用

次の章では、バンドルサイズの分析と最適化について学びます。
