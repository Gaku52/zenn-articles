---
title: "CSS最適化"
---

# CSS最適化

CSSの最適化は、レンダリングパフォーマンスに直接影響します。

## クリティカルCSSのインライン化

```html
<head>
  <style>
    /* ファーストビューのスタイルのみ */
    body { margin: 0; }
    .hero { min-height: 100vh; }
  </style>
  <link rel="preload" href="styles.css" as="style" onload="this.rel='stylesheet'">
</head>
```

## CSS-in-JSの最適化

### Emotion

```typescript
import { css } from '@emotion/react'

const buttonStyle = css`
  padding: 10px 20px;
  background: blue;
`

// ビルド時に静的CSS抽出
<button css={buttonStyle}>Click</button>
```

### Tailwind CSS

```javascript
// tailwind.config.js
module.exports = {
  content: ['./src/**/*.{js,ts,jsx,tsx}'],
  theme: {},
  plugins: [],
}
```

**結果:** 未使用クラスを削除、3.5MB → 8KB

## セレクタの最適化

```css
/* 遅い: 複雑なセレクタ */
.container .sidebar ul li a.active { color: blue; }

/* 速い: シンプルなセレクタ */
.sidebar-link-active { color: blue; }
```

## will-changeの適切な使用

```css
.animating-element {
  will-change: transform;
}

.animating-element.done {
  will-change: auto; /* アニメーション後は削除 */
}
```

## 改善事例

**Before:** 全CSSを読み込み、850KB、FCP 2.4秒
**After:** クリティカルCSS + Tailwind、12KB + 遅延読み込み、FCP 0.9秒（63%改善）
