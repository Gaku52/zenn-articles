---
title: "コード分割戦略"
---

# コード分割戦略

## この章で学ぶこと

コード分割は、初回読み込み時間を劇的に改善する最も効果的な手法の一つです。この章では、効果的なコード分割の戦略を学びます。

- コード分割の種類と使い分け
- ルートベースの分割
- コンポーネントベースの分割
- 機能ベースの分割
- React.lazyとSuspenseの活用
- Next.jsでの動的インポート
- 分割の粒度の決定方法

## コード分割の種類

### 1. ルートベース分割

各ページごとにバンドルを分割する最も基本的な手法。

```typescript
// React Router
import { lazy, Suspense } from 'react'
import { BrowserRouter, Routes, Route } from 'react-router-dom'

const Home = lazy(() => import('./pages/Home'))
const About = lazy(() => import('./pages/About'))
const Dashboard = lazy(() => import('./pages/Dashboard'))

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<Loading />}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/about" element={<About />} />
          <Route path="/dashboard" element={<Dashboard />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  )
}
```

### 2. コンポーネントベース分割

大きなコンポーネントを遅延読み込み。

```typescript
import { lazy, Suspense } from 'react'

const HeavyChart = lazy(() => import('./components/HeavyChart'))
const VideoPlayer = lazy(() => import('./components/VideoPlayer'))

function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      <Suspense fallback={<ChartSkeleton />}>
        <HeavyChart data={data} />
      </Suspense>
      <Suspense fallback={<VideoSkeleton />}>
        <VideoPlayer src={videoUrl} />
      </Suspense>
    </div>
  )
}
```

### 3. 機能ベース分割

特定の機能全体を遅延読み込み。

```typescript
// モーダルを遅延読み込み
const handleOpenModal = async () => {
  const { EditModal } = await import('./modals/EditModal')
  showModal(<EditModal item={item} />)
}

// PDF生成機能を遅延読み込み
const handleExportPDF = async () => {
  const { generatePDF } = await import('./utils/pdfGenerator')
  const pdf = await generatePDF(data)
  downloadPDF(pdf)
}
```

## React での実装

### React.lazy と Suspense

```typescript
import { lazy, Suspense, useState } from 'react'

// 名前付きエクスポートの遅延読み込み
const AdminPanel = lazy(() =>
  import('./AdminPanel').then(module => ({ default: module.AdminPanel }))
)

function App() {
  const [showAdmin, setShowAdmin] = useState(false)

  return (
    <div>
      <button onClick={() => setShowAdmin(true)}>Show Admin</button>
      {showAdmin && (
        <Suspense fallback={<AdminSkeleton />}>
          <AdminPanel />
        </Suspense>
      )}
    </div>
  )
}
```

### エラーハンドリング

```typescript
import { Component, ReactNode, lazy, Suspense } from 'react'

class ErrorBoundary extends Component<
  { children: ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false }

  static getDerivedStateFromError() {
    return { hasError: true }
  }

  render() {
    if (this.state.hasError) {
      return <div>Failed to load component. Please refresh.</div>
    }
    return this.props.children
  }
}

const LazyComponent = lazy(() => import('./Component'))

function App() {
  return (
    <ErrorBoundary>
      <Suspense fallback={<Loading />}>
        <LazyComponent />
      </Suspense>
    </ErrorBoundary>
  )
}
```

## Next.js での実装

### Dynamic Import

```typescript
import dynamic from 'next/dynamic'

// コンポーネントの動的インポート
const DynamicComponent = dynamic(() => import('../components/Heavy'), {
  loading: () => <p>Loading...</p>,
  ssr: false, // SSRを無効化
})

// 名前付きエクスポート
const DynamicWithNamed = dynamic(
  () => import('../components/Named').then(mod => mod.NamedComponent),
  { ssr: false }
)

export default function Page() {
  return (
    <div>
      <DynamicComponent />
      <DynamicWithNamed />
    </div>
  )
}
```

### Suspenseでの実装（App Router）

```typescript
// app/page.tsx
import { Suspense } from 'react'
import HeavyComponent from './HeavyComponent'

export default function Page() {
  return (
    <Suspense fallback={<Loading />}>
      <HeavyComponent />
    </Suspense>
  )
}
```

## ライブラリの分割

### Chart.js の最適化

```typescript
// 悪い例: すべて読み込み
import Chart from 'chart.js/auto'

// 良い例: 必要なものだけ
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  BarElement,
} from 'chart.js'

ChartJS.register(CategoryScale, LinearScale, BarElement)
```

### Lodash の最適化

```typescript
// 悪い例
import _ from 'lodash'
const result = _.uniq(array)

// 良い例
import uniq from 'lodash/uniq'
const result = uniq(array)
```

## ベンダーバンドルの分割

### Webpack設定

```javascript
// webpack.config.js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          priority: 10,
        },
        common: {
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true,
        },
      },
    },
  },
}
```

## 実際の改善事例

### 事例1: SaaSダッシュボード

**Before:**
- 初回バンドル: 980KB
- LCP: 3.2秒

**After（コード分割）:**

```typescript
// ルート分割
const Dashboard = lazy(() => import('./pages/Dashboard'))
const Analytics = lazy(() => import('./pages/Analytics'))
const Settings = lazy(() => import('./pages/Settings'))

// 重いコンポーネントも分割
const Chart = lazy(() => import('./components/Chart'))
const DataTable = lazy(() => import('./components/DataTable'))
```

**結果:**
- 初回バンドル: 980KB → 240KB (76%削減)
- LCP: 3.2秒 → 1.4秒 (56%改善)
- Time to Interactive: 4.8秒 → 2.1秒 (56%改善)

### 事例2: ECサイト

**Before:**
- 全機能を最初に読み込み
- バンドルサイズ: 1.2MB

**After:**

```typescript
// 決済機能は使う時だけ読み込み
const handleCheckout = async () => {
  const { processPayment } = await import('./payment/processor')
  await processPayment(cart)
}

// PDFレシートも遅延読み込み
const handleDownloadReceipt = async () => {
  const { generateReceipt } = await import('./receipt/generator')
  const pdf = await generateReceipt(order)
  download(pdf)
}
```

**結果:**
- 初回バンドル: 1.2MB → 320KB (73%削減)
- FCP: 2.8秒 → 1.2秒 (57%改善)

## まとめ

効果的なコード分割のポイント:

1. **ルート単位で分割**
   - 各ページは独立したバンドル

2. **大きなコンポーネントを分割**
   - Chart、Map、RichEditor など

3. **使用頻度の低い機能を分割**
   - PDF生成、エクスポート機能

4. **適切なローディング状態**
   - Skeleton UI
   - スピナー

次の章では、Tree Shakingとデッドコード削除について学びます。
