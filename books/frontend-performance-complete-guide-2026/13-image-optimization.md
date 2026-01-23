---
title: "画像最適化"
---

# 画像最適化

## この章で学ぶこと

画像はWebページの容量の大半を占めます。この章では、画像を最適化してページ速度を劇的に改善する方法を学びます。

## 画像フォーマットの選択

### フォーマット比較

| フォーマット | 用途 | 圧縮率 | 透過 | アニメーション |
|------------|------|--------|------|--------------|
| JPEG | 写真 | 高 | × | × |
| PNG | ロゴ・アイコン | 中 | ◯ | × |
| WebP | 汎用 | 非常に高 | ◯ | ◯ |
| AVIF | 次世代 | 最高 | ◯ | ◯ |
| SVG | ベクター | 可変 | ◯ | ◯ |

### 実際のサイズ比較

同じ画像での比較:

```
JPEG (Quality 80): 245KB
PNG-24: 890KB
WebP (Quality 80): 156KB (JPEG比 36%削減)
AVIF (Quality 60): 98KB (JPEG比 60%削減)
```

## レスポンシブ画像

### srcset と sizes

```html
<img
  src="image-800.jpg"
  srcset="
    image-400.jpg 400w,
    image-800.jpg 800w,
    image-1200.jpg 1200w,
    image-1600.jpg 1600w
  "
  sizes="(max-width: 640px) 400px,
         (max-width: 1024px) 800px,
         1200px"
  alt="Responsive image"
  width="1200"
  height="800"
  loading="lazy"
/>
```

### picture 要素

```html
<picture>
  <source
    media="(max-width: 640px)"
    srcset="hero-mobile.avif"
    type="image/avif"
  />
  <source
    media="(max-width: 640px)"
    srcset="hero-mobile.webp"
    type="image/webp"
  />
  <source
    srcset="hero-desktop.avif"
    type="image/avif"
  />
  <source
    srcset="hero-desktop.webp"
    type="image/webp"
  />
  <img
    src="hero-desktop.jpg"
    alt="Hero image"
    width="1200"
    height="600"
  />
</picture>
```

## Next.js Image コンポーネント

```typescript
import Image from 'next/image'

export default function ProductImage() {
  return (
    <Image
      src="/product.jpg"
      alt="Product"
      width={800}
      height={600}
      sizes="(max-width: 768px) 100vw, 800px"
      quality={85}
      priority={false}
      placeholder="blur"
      blurDataURL="data:image/jpeg;base64,..."
    />
  )
}
```

### 設定

```javascript
// next.config.js
module.exports = {
  images: {
    formats: ['image/avif', 'image/webp'],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
    minimumCacheTTL: 31536000,
    domains: ['example.com', 'cdn.example.com'],
  },
}
```

## 遅延読み込み

### Native Lazy Loading

```html
<!-- LCP画像: 即座に読み込み -->
<img src="hero.jpg" alt="Hero" loading="eager" fetchpriority="high">

<!-- ファーストビュー外: 遅延読み込み -->
<img src="image.jpg" alt="Image" loading="lazy">
```

### Intersection Observer

```typescript
function LazyImage({ src, alt }) {
  const [isLoaded, setIsLoaded] = useState(false)
  const imgRef = useRef(null)

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsLoaded(true)
          observer.disconnect()
        }
      },
      { rootMargin: '50px' }
    )

    if (imgRef.current) {
      observer.observe(imgRef.current)
    }

    return () => observer.disconnect()
  }, [])

  return (
    <img
      ref={imgRef}
      src={isLoaded ? src : 'placeholder.jpg'}
      alt={alt}
      loading="lazy"
    />
  )
}
```

## Image CDN

### Cloudinary

```typescript
const cloudinaryLoader = ({ src, width, quality }) => {
  const params = [
    'f_auto',  // 自動フォーマット選択
    'q_auto',  // 自動品質調整
    `w_${width}`,
    'c_limit',
  ]
  return `https://res.cloudinary.com/demo/image/upload/${params.join(',')}${src}`
}

<Image
  loader={cloudinaryLoader}
  src="/sample.jpg"
  alt="Sample"
  width={800}
  height={600}
/>
```

### imgix

```typescript
const imgixLoader = ({ src, width, quality }) => {
  const url = new URL(`https://demo.imgix.net${src}`)
  url.searchParams.set('auto', 'format,compress')
  url.searchParams.set('w', width.toString())
  url.searchParams.set('q', (quality || 75).toString())
  return url.href
}
```

## 圧縮ツール

### Sharp (Node.js)

```javascript
const sharp = require('sharp')

await sharp('input.jpg')
  .resize(800, 600)
  .webp({ quality: 80 })
  .toFile('output.webp')

await sharp('input.jpg')
  .resize(800, 600)
  .avif({ quality: 60 })
  .toFile('output.avif')
```

### ImageOptim CLI

```bash
# macOS
imageoptim --quality 80-90 images/*.jpg

# 一括変換
for file in images/*.jpg; do
  cwebp -q 80 "$file" -o "${file%.jpg}.webp"
done
```

## Placeholder 戦略

### Blur Placeholder

```typescript
import { getPlaiceholder } from 'plaiceholder'

export async function getStaticProps() {
  const { base64, img } = await getPlaiceholder('/image.jpg')

  return {
    props: {
      imageProps: {
        ...img,
        blurDataURL: base64,
      },
    },
  }
}

export default function Page({ imageProps }) {
  return (
    <Image
      {...imageProps}
      placeholder="blur"
      alt="Image"
    />
  )
}
```

### LQIP (Low Quality Image Placeholder)

```typescript
// 極小画像（2-3KB）を先に表示
<img
  src="image-lqip.jpg"  // 20x15pxなど
  data-src="image-full.jpg"
  alt="Image"
  style={{ filter: 'blur(10px)' }}
/>
```

## 改善事例

### 事例: ECサイト

**Before:**
- 商品画像: JPEG 450KB × 20枚 = 9MB
- LCP: 4.8秒

**After:**

```typescript
<Image
  src={product.image}
  alt={product.name}
  width={600}
  height={600}
  sizes="(max-width: 768px) 100vw, 600px"
  quality={85}
  formats={['avif', 'webp']}
  loading={index < 3 ? 'eager' : 'lazy'}
  priority={index === 0}
/>
```

**結果:**
- 画像サイズ: 9MB → 1.2MB (87%削減)
- LCP: 4.8秒 → 1.4秒 (71%改善)
- コンバージョン率: +15%

## まとめ

画像最適化のポイント:

1. **適切なフォーマット**
   - AVIF > WebP > JPEG

2. **レスポンシブ画像**
   - srcset/sizes
   - picture要素

3. **遅延読み込み**
   - loading="lazy"
   - Intersection Observer

4. **Image CDN**
   - 自動最適化
   - グローバル配信

5. **サイズ指定**
   - width/height属性
   - CLSの防止

次の章では、フォント読み込み戦略について学びます。
