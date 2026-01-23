---
title: "実践事例 Part 1: ECサイトの改善"
---

# 実践事例 Part 1: ECサイトの改善

典型的なECサイトを想定した最適化シナリオを詳しく解説します。

## 想定シナリオ

**想定サイト:** ファッションECサイト（仮想プロジェクト）
**想定規模:** 月間PV 500万程度
**想定課題:** 低いコンバージョン率、高い直帰率

## 初期状態の分析

### パフォーマンス指標

| 指標 | 値 | 目標 |
|------|-----|------|
| LCP | 4.8秒 | 2.5秒以下 |
| INP | 420ms | 200ms以下 |
| CLS | 0.31 | 0.1以下 |
| FCP | 2.8秒 | 1.8秒以下 |
| TTFB | 980ms | 800ms以下 |

### ビジネス指標

- 直帰率: 68%
- コンバージョン率: 1.2%
- 平均セッション時間: 1分20秒

## Phase 1: 画像最適化（Week 1-2）

### 実施内容

```typescript
// Before: 未最適化JPEG
<img src="product.jpg" alt="Product"> // 450KB

// After: Next.js Image + AVIF
<Image
  src="/product.jpg"
  alt="Product"
  width={600}
  height={600}
  sizes="(max-width: 768px) 100vw, 600px"
  quality={85}
  priority={index === 0}
/>
// 45KB (90%削減)
```

### 結果

- 画像サイズ: 9MB → 1.2MB (87%削減)
- LCP: 4.8秒 → 2.3秒 (52%改善)
- **コンバージョン率: 1.2% → 1.8% (+50%)**

## Phase 2: バンドルサイズ削減（Week 3-4）

### 実施内容

```typescript
// 1. lodashの最適化
// Before: 72KB
import _ from 'lodash'

// After: 3KB
import uniq from 'lodash/uniq'

// 2. moment.js → dayjs
// Before: 71KB
import moment from 'moment'

// After: 3KB
import dayjs from 'dayjs'

// 3. コード分割
const CheckoutModal = dynamic(() => import('./CheckoutModal'), {
  loading: () => <Skeleton />,
  ssr: false,
})
```

### 結果

- バンドルサイズ: 1.2MB → 320KB (73%削減)
- FCP: 2.8秒 → 1.2秒 (57%改善)
- TTI: 5.2秒 → 2.4秒(54%改善)

## Phase 3: ISR導入（Week 5-6）

### 実施内容

```typescript
// Before: CSR
function ProductPage() {
  const [product, setProduct] = useState(null)

  useEffect(() => {
    fetch(`/api/product/${id}`)
      .then(res => res.json())
      .then(setProduct)
  }, [id])

  if (!product) return <Loading />
  return <Product data={product} />
}

// After: ISR
export async function getStaticProps({ params }) {
  const product = await getProduct(params.id)

  return {
    props: { product },
    revalidate: 300, // 5分ごとに再生成
  }
}

export default function ProductPage({ product }) {
  return <Product data={product} />
}
```

### 結果

- TTFB: 980ms → 210ms (79%改善)
- FCP: 1.2秒 → 0.6秒 (50%改善)
- SEO: 検索順位が大幅に向上

## Phase 4: CLS改善（Week 7）

### 実施内容

```typescript
// 1. 画像サイズ指定
<Image
  src={product.image}
  alt={product.name}
  width={600}
  height={600}
  style={{ aspectRatio: '1/1' }}
/>

// 2. Skeleton UI
{loading ? (
  <ProductSkeleton />
) : (
  <ProductList products={products} />
)}

// 3. フォント最適化
import { Inter } from 'next/font/google'

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  adjustFontFallback: true,
})
```

### 結果

- CLS: 0.31 → 0.06 (81%改善)
- 誤クリック率: 18% → 3%削減

## Phase 5: INP改善（Week 8）

### 実施内容

```typescript
// 1. 仮想スクロール
import { useVirtualizer } from '@tanstack/react-virtual'

function ProductList({ products }) {
  const virtualizer = useVirtualizer({
    count: products.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 300,
  })

  return (
    <div ref={parentRef}>
      {virtualizer.getVirtualItems().map(item => (
        <ProductCard
          key={item.key}
          product={products[item.index]}
        />
      ))}
    </div>
  )
}

// 2. debounce
const debouncedSearch = useMemo(
  () => debounce((query) => search(query), 300),
  []
)

// 3. React.memo
const ProductCard = React.memo(({ product }) => {
  return <div>{product.name}</div>
})
```

### 結果

- INP: 420ms → 150ms (64%改善)
- 検索体験が大幅に改善

## 最終結果

### パフォーマンス指標

| 指標 | Before | After | 改善率 |
|------|--------|-------|--------|
| LCP | 4.8秒 | 1.4秒 | 71% |
| INP | 420ms | 150ms | 64% |
| CLS | 0.31 | 0.06 | 81% |
| FCP | 2.8秒 | 0.6秒 | 79% |
| TTFB | 980ms | 210ms | 79% |

### ビジネス指標

| 指標 | Before | After | 改善率 |
|------|--------|-------|--------|
| 直帰率 | 68% | 42% | 38%改善 |
| コンバージョン率 | 1.2% | 2.1% | 75%向上 |
| 平均セッション時間 | 1分20秒 | 2分45秒 | 106%向上 |
| 平均注文額 | ¥8,200 | ¥9,400 | 15%向上 |

### ROI

- 月間売上: +32% (約¥48M → ¥63M)
- 開発コスト: 約¥5M
- ROI: 3ヶ月で回収

## 学んだこと

1. **画像最適化が最も効果的**
   - 即座に大きな改善
   - 実装も比較的簡単

2. **ISRで初回表示を高速化**
   - SSRよりサーバー負荷が低い
   - SEO対応も万全

3. **段階的な改善が重要**
   - 一度にすべて変更しない
   - 各フェーズで効果測定

4. **ビジネス指標との相関**
   - パフォーマンス向上→CV率向上
   - データで効果を証明

## 継続的改善

現在も以下を継続実施:

```yaml
# CI/CD統合
- Lighthouse CI: PRごと
- Bundle size check: PRごと
- RUM: リアルタイム監視
- 週次パフォーマンスレビュー
```

次の章では、SaaSダッシュボードの最適化事例を紹介します。
