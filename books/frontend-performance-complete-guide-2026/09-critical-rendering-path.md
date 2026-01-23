---
title: "クリティカルレンダリングパス"
---

# クリティカルレンダリングパス

## この章で学ぶこと

クリティカルレンダリングパスは、ブラウザがHTMLをピクセルに変換するまでの一連のステップです。このパスを最適化することで、初回レンダリングを劇的に高速化できます。

## レンダリングの流れ

```
HTML → DOM → CSSOM → Render Tree → Layout → Paint → Composite
```

### 1. DOM構築

HTMLをパースしてDOM（Document Object Model）を構築します。

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Page</title>
  </head>
  <body>
    <div id="app">
      <h1>Hello</h1>
    </div>
  </body>
</html>
```

### 2. CSSOM構築

CSSをパースしてCSS Object Modelを構築します。

```css
body { font-size: 16px; }
h1 { color: blue; }
```

### 3. Render Tree構築

DOMとCSSOMを組み合わせてRender Treeを作成します。

### 4. Layout（Reflow）

各要素の正確な位置とサイズを計算します。

### 5. Paint

ピクセルを実際に描画します。

### 6. Composite

複数のレイヤーを合成して最終的な画面を生成します。

## レンダリングブロッキングの削減

### CSSの最適化

**悪い例:**

```html
<head>
  <link rel="stylesheet" href="styles.css">
  <!-- CSSがダウンロードされるまでレンダリングブロック -->
</head>
```

**良い例:**

```html
<head>
  <!-- クリティカルCSSをインライン化 -->
  <style>
    body { margin: 0; font-family: sans-serif; }
    .hero { min-height: 100vh; }
  </style>
  <!-- 非クリティカルCSSは非同期読み込み -->
  <link rel="preload" href="styles.css" as="style" onload="this.onload=null;this.rel='stylesheet'">
  <noscript><link rel="stylesheet" href="styles.css"></noscript>
</head>
```

### JavaScriptの最適化

**悪い例:**

```html
<head>
  <script src="app.js"></script>
  <!-- JavaScriptがダウンロード・実行されるまでブロック -->
</head>
```

**良い例:**

```html
<head>
  <!-- defer で非同期ダウンロード、HTML解析後に実行 -->
  <script src="app.js" defer></script>
</head>

<body>
  <!-- または body の最後に配置 -->
  <script src="app.js"></script>
</body>
```

### async vs defer

| 属性 | ダウンロード | 実行タイミング | 順序保証 |
|------|------------|---------------|---------|
| なし | ブロック | 即座 | あり |
| async | 非同期 | ダウンロード完了後すぐ | なし |
| defer | 非同期 | HTML解析完了後 | あり |

```html
<!-- Analytics（順序不要）→ async -->
<script src="analytics.js" async></script>

<!-- アプリケーションコード（順序重要）→ defer -->
<script src="vendor.js" defer></script>
<script src="app.js" defer></script>
```

## クリティカルCSS の抽出

### Critical ツール

```bash
npm install --save-dev critical
```

```javascript
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

### Next.jsでの実装

```typescript
// pages/_document.tsx
import Document, { Html, Head, Main, NextScript } from 'next/document'

class MyDocument extends Document {
  render() {
    return (
      <Html>
        <Head>
          <style
            dangerouslySetInnerHTML={{
              __html: `
                body { margin: 0; }
                .hero { min-height: 100vh; }
              `,
            }}
          />
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

## リソースヒント

### Preconnect

```html
<!-- サードパーティドメインへの事前接続 -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
```

### DNS Prefetch

```html
<link rel="dns-prefetch" href="https://cdn.example.com">
```

### Preload

```html
<!-- 重要なリソースを優先的にダウンロード -->
<link rel="preload" as="image" href="hero.jpg">
<link rel="preload" as="font" type="font/woff2" href="font.woff2" crossorigin>
```

## 改善事例

### 事例: コーポレートサイト

**Before:**
- FCP: 2.4秒
- LCP: 3.8秒
- すべてのCSSをブロッキング読み込み

**After:**

```html
<head>
  <!-- クリティカルCSSをインライン（5KB） -->
  <style>
    /* ファーストビューのスタイルのみ */
  </style>

  <!-- 残りは非同期 -->
  <link rel="preload" href="styles.css" as="style" onload="this.onload=null;this.rel='stylesheet'">

  <!-- フォントをプリロード -->
  <link rel="preload" href="/fonts/main.woff2" as="font" type="font/woff2" crossorigin>

  <!-- JavaScriptはdefer -->
  <script src="app.js" defer></script>
</head>
```

**結果:**
- FCP: 2.4秒 → 0.9秒 (63%改善)
- LCP: 3.8秒 → 1.6秒 (58%改善)

## まとめ

クリティカルレンダリングパス最適化のポイント:

1. **CSSの最適化**
   - クリティカルCSSのインライン化
   - 非クリティカルCSSの非同期読み込み

2. **JavaScriptの最適化**
   - defer/async属性の活用
   - body終了タグ直前に配置

3. **リソースヒント**
   - preconnect
   - preload
   - dns-prefetch

4. **測定と改善**
   - Lighthouse
   - WebPageTest

次の章では、SSR vs CSR vs SSGについて学びます。
