---
title: "Dynamic Imports"
---

# Dynamic Imports

## この章で学ぶこと

- Dynamic Importsの基礎
- 適切な使用タイミング
- Preloadとの組み合わせ
- エラーハンドリング
- パフォーマンス測定
- 実践例

## Dynamic Importsとは

動的にモジュールを読み込む機能。必要な時だけコードを読み込むことで初回読み込みを高速化します。

### 基本構文

```typescript
// 静的インポート（ビルド時にバンドル）
import { heavyFunction } from './heavy'

// 動的インポート（実行時に読み込み）
const module = await import('./heavy')
module.heavyFunction()
```

## 使用パターン

### 1. ユーザーアクション時

```typescript
// ボタンクリック時にモーダルを読み込み
button.addEventListener('click', async () => {
  const { Modal } = await import('./Modal')
  const modal = new Modal()
  modal.show()
})
```

### 2. 条件付き読み込み

```typescript
// 管理者のみ表示する機能
if (user.isAdmin) {
  const { AdminPanel } = await import('./AdminPanel')
  renderAdminPanel(AdminPanel)
}
```

### 3. ルート分割

```typescript
const router = {
  '/': () => import('./pages/Home'),
  '/about': () => import('./pages/About'),
  '/dashboard': () => import('./pages/Dashboard'),
}

async function navigate(path) {
  const page = await router[path]()
  render(page.default)
}
```

## React での実装

### React.lazy

```typescript
import { lazy, Suspense } from 'react'

const HeavyComponent = lazy(() => import('./HeavyComponent'))

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <HeavyComponent />
    </Suspense>
  )
}
```

### 条件付きレンダリング

```typescript
function Dashboard() {
  const [showChart, setShowChart] = useState(false)
  const Chart = lazy(() => import('./Chart'))

  return (
    <div>
      <button onClick={() => setShowChart(true)}>
        Show Chart
      </button>
      {showChart && (
        <Suspense fallback={<ChartSkeleton />}>
          <Chart data={data} />
        </Suspense>
      )}
    </div>
  )
}
```

## Next.js での実装

### dynamic()

```typescript
import dynamic from 'next/dynamic'

const DynamicComponent = dynamic(() => import('../components/Heavy'), {
  loading: () => <p>Loading...</p>,
  ssr: false,
})

export default function Page() {
  return <DynamicComponent />
}
```

### 名前付きエクスポート

```typescript
const DynamicChart = dynamic(
  () => import('../components/Chart').then(mod => mod.Chart),
  { ssr: false }
)
```

## Preloadとの組み合わせ

### マウスホバー時にプリロード

```typescript
function ProductCard({ product }) {
  const preloadModal = () => {
    import('./ProductModal')
  }

  const handleClick = async () => {
    const { ProductModal } = await import('./ProductModal')
    showModal(<ProductModal product={product} />)
  }

  return (
    <div
      onMouseEnter={preloadModal}
      onClick={handleClick}
    >
      {product.name}
    </div>
  )
}
```

### Intersection Observer

```typescript
function LazySection() {
  const [shouldLoad, setShouldLoad] = useState(false)
  const ref = useRef(null)

  useEffect(() => {
    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) {
        setShouldLoad(true)
      }
    })

    if (ref.current) {
      observer.observe(ref.current)
    }

    return () => observer.disconnect()
  }, [])

  const HeavyComponent = lazy(() => import('./HeavyComponent'))

  return (
    <div ref={ref}>
      {shouldLoad && (
        <Suspense fallback={<Skeleton />}>
          <HeavyComponent />
        </Suspense>
      )}
    </div>
  )
}
```

## エラーハンドリング

```typescript
async function loadModule() {
  try {
    const module = await import('./module')
    return module
  } catch (error) {
    console.error('Failed to load module:', error)
    // フォールバック処理
    return null
  }
}
```

### React Error Boundary

```typescript
class ErrorBoundary extends Component {
  state = { hasError: false }

  static getDerivedStateFromError(error) {
    return { hasError: true }
  }

  render() {
    if (this.state.hasError) {
      return <div>Failed to load component</div>
    }
    return this.props.children
  }
}

function App() {
  const Component = lazy(() => import('./Component'))

  return (
    <ErrorBoundary>
      <Suspense fallback={<Loading />}>
        <Component />
      </Suspense>
    </ErrorBoundary>
  )
}
```

## パフォーマンス測定

```typescript
async function loadWithTiming(path) {
  const start = performance.now()
  const module = await import(path)
  const duration = performance.now() - start

  console.log(`Loaded ${path} in ${duration}ms`)
  return module
}
```

## 実践事例

### 事例1: PDF生成機能

```typescript
// 使用時のみ読み込み（2MB）
const handleExportPDF = async () => {
  const { jsPDF } = await import('jspdf')
  const pdf = new jsPDF()
  pdf.text('Hello', 10, 10)
  pdf.save('document.pdf')
}
```

**結果:**
- 初回バンドル: 2MB削減
- ボタンクリック→PDF生成: 800ms

### 事例2: Chart.js

```typescript
const ChartComponent = dynamic(
  () => import('react-chartjs-2').then(mod => mod.Line),
  { ssr: false, loading: () => <ChartSkeleton /> }
)
```

**結果:**
- 初回バンドル: 242KB削減
- チャート表示: 350ms

## まとめ

Dynamic Importsの活用ポイント:

1. **初回読み込みの最適化**
   - 大きなライブラリは動的読み込み

2. **適切なタイミング**
   - ユーザーアクション時
   - 条件付き機能

3. **Preloadと組み合わせ**
   - ホバー時にプリロード
   - Intersection Observer

4. **エラーハンドリング**
   - try-catch
   - Error Boundary

次の章では、クリティカルレンダリングパスについて学びます。
