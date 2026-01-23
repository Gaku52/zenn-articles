---
title: "フォント読み込み戦略"
---

# フォント読み込み戦略

Webフォントは視覚的な品質を向上させますが、パフォーマンスに大きな影響を与えます。この章では効率的なフォント読み込み戦略を学びます。

## font-display戦略

```css
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
  font-display: swap; /* 推奨 */
}
```

| 値 | 動作 | 用途 |
|----|------|------|
| swap | すぐに代替表示 | 一般的な用途 |
| optional | 100ms待機後判断 | パフォーマンス最優先 |
| fallback | 短時間待機 | バランス型 |
| block | 3秒待機 | 非推奨 |

## フォントのプリロード

```html
<link
  rel="preload"
  href="/fonts/Inter-Variable.woff2"
  as="font"
  type="font/woff2"
  crossorigin
/>
```

## Next.jsでの最適化

```typescript
import { Inter } from 'next/font/google'

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  adjustFontFallback: true,
})

export default function RootLayout({ children }) {
  return (
    <html className={inter.className}>
      <body>{children}</body>
    </html>
  )
}
```

## サブセット化

```bash
# pyftsubset で必要な文字だけ抽出
pyftsubset font.otf \
  --text-file=chars.txt \
  --flavor=woff2 \
  --output-file=font-subset.woff2
```

## 可変フォント

```css
@font-face {
  font-family: 'Inter Variable';
  src: url('/fonts/Inter-Variable.woff2') format('woff2');
  font-weight: 100 900;
  font-display: swap;
}

h1 { font-weight: 700; }
p { font-weight: 400; }
```

**メリット:** 1ファイルで全ウェイトをカバー、520KB → 85KB

## 改善事例

**Before:** Google Fonts CDN、520KB、LCP 2.8秒
**After:** セルフホスト + 可変フォント + サブセット、85KB、LCP 1.4秒（50%改善）
