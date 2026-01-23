---
title: "PreloadとPrefetch"
---

# PreloadとPrefetch

リソースヒントを活用して、必要なリソースを先読みしてパフォーマンスを向上させます。

## Preload

**現在のページで必要なリソース**を優先的に読み込み。

```html
<!-- LCP画像 -->
<link rel="preload" as="image" href="hero.jpg" fetchpriority="high">

<!-- フォント -->
<link rel="preload" as="font" type="font/woff2" href="font.woff2" crossorigin>

<!-- CSS -->
<link rel="preload" as="style" href="critical.css">

<!-- JavaScript -->
<link rel="preload" as="script" href="app.js">
```

## Prefetch

**次のページで必要になる**リソースをアイドル時に読み込み。

```html
<!-- 次のページで使うリソース -->
<link rel="prefetch" href="/next-page.html">
<link rel="prefetch" href="/next-page.js">
```

## Preconnect

**サードパーティドメイン**への接続を事前確立。

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
```

## DNS Prefetch

**DNSルックアップ**のみ先に実行。

```html
<link rel="dns-prefetch" href="https://analytics.example.com">
```

## Next.jsでの実装

```typescript
import Link from 'next/link'

// 自動でprefetch
<Link href="/about" prefetch={true}>
  About
</Link>

// マウスホバー時のprefetch
<Link
  href="/product"
  onMouseEnter={() => {
    router.prefetch('/product')
  }}
>
  Product
</Link>
```

## Reactでのカスタム実装

```typescript
function usePreload(href: string) {
  useEffect(() => {
    const link = document.createElement('link')
    link.rel = 'prefetch'
    link.href = href
    document.head.appendChild(link)

    return () => {
      document.head.removeChild(link)
    }
  }, [href])
}

// 使用例
function ProductCard({ nextPageUrl }) {
  usePreload(nextPageUrl)

  return <div onMouseEnter={() => usePreload(nextPageUrl)}>
    Product
  </div>
}
```

## 優先順位

| リソースヒント | タイミング | 優先度 | 用途 |
|--------------|----------|--------|------|
| preload | 即座 | 高 | 現在のページ |
| prefetch | アイドル時 | 低 | 次のページ |
| preconnect | 即座 | 中 | サードパーティ |
| dns-prefetch | 即座 | 低 | DNS解決 |

## 改善事例

**Before:** ナビゲーション後の読み込み 2.1秒
**After:** prefetch導入、ナビゲーション後 0.3秒（86%改善）
