---
title: "パフォーマンス最適化の実践"
---

# Chapter 4: React パフォーマンス最適化 完全ガイド

> 想定される効果に基づいた実践的なパフォーマンス改善手法

## この章で学べること

この章では、Reactアプリケーションのパフォーマンス最適化を実践的に学びます。

- ✅ React.memoとuseMemo/useCallbackの正しい使い分け
- ✅ Code Splitting（コード分割）による初期ロード高速化
- ✅ 仮想化（Virtualization）で大量データを高速表示
- ✅ バンドルサイズ削減と依存関係の最適化

**前提知識**: Chapter 01-03の内容、Reactの基礎

**所要時間**: 60-80分

---

## 目次

1. [パフォーマンス計測の基礎](#1-パフォーマンス計測の基礎)
   - React DevTools Profiler
   - Core Web Vitals
2. [React.memo による再レンダリング最適化](#2-reactmemo-による再レンダリング最適化)
   - 基本的な使い方
   - カスタム比較関数
   - いつ使うべきか
3. [useMemo と useCallback](#3-usememo-と-usecallback)
   - useMemoの使い方
   - useCallbackの使い方
   - 使い分けのガイドライン
4. [Code Splitting（コード分割）](#4-code-splittingコード分割)
   - React.lazyとSuspense
   - ルートベースの分割
   - コンポーネントベースの分割
5. [仮想化（Virtualization）](#5-仮想化virtualization)
   - react-windowの使い方
   - 大量データの最適化
6. [実践：パフォーマンス改善ケーススタディ](#6-実践パフォーマンス改善ケーススタディ)

---

## 1. パフォーマンス計測の基礎

### React DevTools Profiler

```typescript
// Profilerコンポーネントで計測
import { Profiler, ProfilerOnRenderCallback } from 'react'

const onRenderCallback: ProfilerOnRenderCallback = (
  id, // Profilerのid
  phase, // "mount"（初回）または "update"（更新）
  actualDuration, // レンダリングにかかった時間
  baseDuration, // メモ化なしでかかる推定時間
  startTime, // レンダリング開始時刻
  commitTime, // コミット時刻
  interactions // このレンダリングに関連するinteractions
) => {
  console.log(`${id} (${phase}):`, {
    actualDuration,
    baseDuration
  })
}

function App() {
  return (
    <Profiler id="App" onRender={onRenderCallback}>
      <YourComponent />
    </Profiler>
  )
}
```typescript

### カスタムフックでパフォーマンス計測

```typescript
// レンダリング回数の計測
function useRenderCount(componentName: string) {
  const renderCount = useRef(0)

  useEffect(() => {
    renderCount.current += 1
    console.log(`${componentName} rendered ${renderCount.current} times`)
  })

  return renderCount.current
}

// 使用例
function ExpensiveComponent() {
  const renderCount = useRenderCount('ExpensiveComponent')

  return <div>Rendered {renderCount} times</div>
}

// レンダリング時間の計測
function useRenderTime(componentName: string) {
  const startTime = useRef(performance.now())

  useEffect(() => {
    const endTime = performance.now()
    const duration = endTime - startTime.current
    console.log(`${componentName} render time: ${duration.toFixed(2)}ms`)
    startTime.current = endTime
  })
}
```typescript

---

## 2. React.memo による再レンダリング防止

### 基本的な使い方

```typescript
// ❌ メモ化なし
function ListItem({ item }: { item: Item }) {
  console.log('ListItem rendered')
  return <li>{item.name}</li>
}

function List({ items }: { items: Item[] }) {
  const [filter, setFilter] = useState('')

  return (
    <>
      <input value={filter} onChange={e => setFilter(e.target.value)} />
      <ul>
        {items.map(item => (
          <ListItem key={item.id} item={item} />
        ))}
      </ul>
    </>
  )
}

// 問題：filterが変わるたびに全てのListItemが再レンダリング

// ✅ React.memoで最適化
const ListItem = memo(({ item }: { item: Item }) => {
  console.log('ListItem rendered')
  return <li>{item.name}</li>
})

// 結果：filterが変わってもListItemは再レンダリングされない
```typescript

### React.memoを使うべきとき・使わないべきとき

```typescript
// ✅ 使うべき：重いコンポーネント
const ExpensiveChart = memo(({ data }: { data: number[] }) => {
  // 複雑な計算やレンダリング
  const processedData = complexCalculation(data)
  return <Chart data={processedData} />
})

// ✅ 使うべき：大量のアイテム
const TodoItem = memo(({ todo }: { todo: Todo }) => {
  return <li>{todo.text}</li>
})

function TodoList({ todos }: { todos: Todo[] }) {
  return (
    <ul>
      {todos.map(todo => (
        <TodoItem key={todo.id} todo={todo} />
      ))}
    </ul>
  )
}

// ❌ 使わないべき：単純なコンポーネント
const SimpleText = memo(({ text }: { text: string }) => {
  return <p>{text}</p>
})
// メモ化のオーバーヘッドの方が大きい

// ❌ 使わないべき：Propsが毎回変わるコンポーネント
const AlwaysChanging = memo(({ timestamp }: { timestamp: number }) => {
  return <p>{timestamp}</p>
})
// timestampが毎回変わるのでメモ化の意味がない
```typescript

---

## 3. useMemo/useCallback の使い分け

### useMemo: 計算結果のメモ化

```typescript
// ❌ 毎レンダリングで計算
function Component({ items }: { items: number[] }) {
  const sum = items.reduce((acc, item) => acc + item, 0)
  const average = sum / items.length

  return <div>Average: {average}</div>
}

// ✅ useMemoで計算結果をキャッシュ
function Component({ items }: { items: number[] }) {
  const average = useMemo(() => {
    const sum = items.reduce((acc, item) => acc + item, 0)
    return sum / items.length
  }, [items])

  return <div>Average: {average}</div>
}

// 複雑な計算の例
function DataVisualization({ data }: { data: DataPoint[] }) {
  // フィルタリング、ソート、集計を含む重い処理
  const processedData = useMemo(() => {
    console.log('Processing data...')
    return data
      .filter(point => point.value > 0)
      .sort((a, b) => b.value - a.value)
      .slice(0, 100)
      .map(point => ({
        ...point,
        normalized: point.value / Math.max(...data.map(d => d.value))
      }))
  }, [data])

  return <Chart data={processedData} />
}
```typescript

### useCallback: 関数のメモ化

```typescript
// ❌ 毎レンダリングで新しい関数
function Parent() {
  const [count, setCount] = useState(0)

  const handleClick = () => {
    console.log('Clicked')
  }

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <ExpensiveChild onClick={handleClick} />
    </>
  )
}

const ExpensiveChild = memo(({ onClick }: { onClick: () => void }) => {
  console.log('Child rendered')
  return <button onClick={onClick}>Child Button</button>
})

// 問題：countが変わるたびにhandleClickが新しくなり、Childが再レンダリング

// ✅ useCallbackで関数をメモ化
function Parent() {
  const [count, setCount] = useState(0)

  const handleClick = useCallback(() => {
    console.log('Clicked')
  }, []) // 依存配列が空なので関数は常に同じ

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <ExpensiveChild onClick={handleClick} />
    </>
  )
}

// 結果：countが変わってもChildは再レンダリングされない
```typescript

### useMemo vs useCallback の使い分け

```typescript
// useMemo: 値のメモ化
const expensiveValue = useMemo(() => computeExpensiveValue(a, b), [a, b])

// useCallback: 関数のメモ化
const memoizedCallback = useCallback(() => {
  doSomething(a, b)
}, [a, b])

// 実は同じ（useCallbackはuseMemoのシンタックスシュガー）
const memoizedCallback = useMemo(() => {
  return () => {
    doSomething(a, b)
  }
}, [a, b])

// 実用例：オブジェクトのメモ化
// ❌ 毎回新しいオブジェクト
function Component() {
  const config = { url: '/api', timeout: 5000 }
  // configは毎レンダリングで新しいオブジェクト
}

// ✅ useMemoでメモ化
function Component() {
  const config = useMemo(() => ({
    url: '/api',
    timeout: 5000
  }), [])
  // configは常に同じオブジェクト
}
```typescript

---

## 4. Code Splitting（コード分割）

### React.lazy と Suspense

```typescript
// ❌ 全てを事前にインポート（初期バンドルが大きい）
import HeavyComponent from './HeavyComponent'
import AnotherHeavyComponent from './AnotherHeavyComponent'

function App() {
  return (
    <>
      <HeavyComponent />
      <AnotherHeavyComponent />
    </>
  )
}

// ✅ 動的インポート
const HeavyComponent = lazy(() => import('./HeavyComponent'))
const AnotherHeavyComponent = lazy(() => import('./AnotherHeavyComponent'))

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <HeavyComponent />
      <AnotherHeavyComponent />
    </Suspense>
  )
}
```typescript

### Route-based Code Splitting

```typescript
import { lazy, Suspense } from 'react'
import { BrowserRouter, Routes, Route } from 'react-router-dom'

// 各ルートを個別にロード
const Home = lazy(() => import('./pages/Home'))
const About = lazy(() => import('./pages/About'))
const Dashboard = lazy(() => import('./pages/Dashboard'))
const Settings = lazy(() => import('./pages/Settings'))

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<PageLoader />}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/about" element={<About />} />
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/settings" element={<Settings />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  )
}

// 結果：
// - 初期バンドル: 50KB → 15KB（70%削減）
// - 各ルート: 必要な時のみロード
// - FCP（First Contentful Paint）: 1.2s → 0.4s（3倍高速化）
```typescript

### Component-based Code Splitting

```typescript
// 重いモーダルを動的ロード
const HeavyModal = lazy(() => import('./HeavyModal'))

function App() {
  const [isModalOpen, setModalOpen] = useState(false)

  return (
    <>
      <button onClick={() => setModalOpen(true)}>Open Modal</button>
      {isModalOpen && (
        <Suspense fallback={<div>Loading...</div>}>
          <HeavyModal onClose={() => setModalOpen(false)} />
        </Suspense>
      )}
    </>
  )
}

// チャートライブラリを動的ロード
const Chart = lazy(() => import('react-chartjs-2').then(module => ({
  default: module.Line
})))

function Dashboard() {
  return (
    <Suspense fallback={<ChartSkeleton />}>
      <Chart data={chartData} />
    </Suspense>
  )
}
```typescript

---

## 5. 仮想化（Virtualization）

### react-window を使った仮想化

```typescript
import { FixedSizeList } from 'react-window'

interface Item {
  id: string
  name: string
}

// ❌ 1万個のアイテムを全て表示（遅い）
function BadList({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  )
}

// ✅ 仮想化（表示されている部分のみレンダリング）
function VirtualizedList({ items }: { items: Item[] }) {
  const Row = ({ index, style }: { index: number; style: React.CSSProperties }) => (
    <div style={style}>
      {items[index].name}
    </div>
  )

  return (
    <FixedSizeList
      height={600}
      itemCount={items.length}
      itemSize={35}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  )
}

// 結果（10,000アイテム）:
// - 悪い例: 初期レンダリング 2.5秒、メモリ使用量 150MB
// - 良い例: 初期レンダリング 0.05秒、メモリ使用量 5MB
// - パフォーマンス改善: 50倍
```typescript

### 可変高さの仮想化

```typescript
import { VariableSizeList } from 'react-window'

interface Message {
  id: string
  text: string
  author: string
}

function VirtualizedChat({ messages }: { messages: Message[] }) {
  // 各アイテムの高さを計算
  const getItemSize = (index: number) => {
    const message = messages[index]
    // テキストの長さに基づいて高さを推定
    const lines = Math.ceil(message.text.length / 50)
    return 20 + lines * 20
  }

  const Row = ({ index, style }: { index: number; style: React.CSSProperties }) => {
    const message = messages[index]
    return (
      <div style={style}>
        <strong>{message.author}</strong>
        <p>{message.text}</p>
      </div>
    )
  }

  return (
    <VariableSizeList
      height={600}
      itemCount={messages.length}
      itemSize={getItemSize}
      width="100%"
    >
      {Row}
    </VariableSizeList>
  )
}
```typescript

---

## 6. レンダリング最適化

### 条件付きレンダリングの最適化

```typescript
// ❌ 不要なコンポーネントもマウント
function BadConditional({ show }: { show: boolean }) {
  return (
    <div>
      <HeavyComponent style={{ display: show ? 'block' : 'none' }} />
    </div>
  )
}

// ✅ 条件によってマウント/アンマウント
function GoodConditional({ show }: { show: boolean }) {
  return (
    <div>
      {show && <HeavyComponent />}
    </div>
  )
}

// ✅ より良い：early return
function BetterConditional({ show }: { show: boolean }) {
  if (!show) return null
  return <HeavyComponent />
}
```typescript

### Key の最適化

```typescript
// ❌ インデックスをkeyに使用
function BadList({ items }: { items: string[] }) {
  return (
    <ul>
      {items.map((item, index) => (
        <li key={index}>{item}</li>
      ))}
    </ul>
  )
}

// 問題：アイテムの順序が変わると全て再レンダリング

// ✅ 一意のIDをkeyに使用
function GoodList({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  )
}
```typescript

---

## 7. バンドルサイズ削減

### Tree Shaking

```typescript
// ❌ デフォルトエクスポートをインポート（全体がバンドルされる）
import _ from 'lodash'
const result = _.debounce(fn, 300)

// ✅ 名前付きエクスポートをインポート（必要な部分のみ）
import debounce from 'lodash/debounce'
const result = debounce(fn, 300)

// さらに良い：lodash-es（ES Modules版）
import { debounce } from 'lodash-es'
```typescript

### 依存関係の見直し

```typescript
// ❌ 重いライブラリ（moment.js: 288KB）
import moment from 'moment'
const date = moment().format('YYYY-MM-DD')

// ✅ 軽量な代替（date-fns: 78KB）
import { format } from 'date-fns'
const date = format(new Date(), 'yyyy-MM-dd')

// さらに良い：ネイティブAPI（0KB）
const date = new Date().toISOString().split('T')[0]
```typescript

### Dynamic Import でライブラリを遅延ロード

```typescript
// QRコード生成ライブラリを動的ロード
function QRCodeGenerator({ value }: { value: string }) {
  const [QRCode, setQRCode] = useState<any>(null)

  useEffect(() => {
    import('qrcode.react').then(module => {
      setQRCode(() => module.QRCodeCanvas)
    })
  }, [])

  if (!QRCode) return <div>Loading...</div>

  return <QRCode value={value} />
}
```typescript

---

## 8. 想定される効果と改善事例

### ケーススタディ1: ECサイトの商品一覧

**問題**:
- 1000個の商品を表示
- 初期ロード時間: 3.2秒
- スクロールでカクつき

**改善施策**:
1. ✅ react-windowで仮想化
2. ✅ 画像の遅延ロード
3. ✅ React.memoで商品カードをメモ化

**結果**:
- 初期ロード時間: 3.2s → 0.6s（5.3倍高速化）
- スクロールFPS: 30fps → 60fps（滑らか）
- メモリ使用量: 250MB → 45MB（82%削減）

### ケーススタディ2: チャットアプリ

**問題**:
- 5000件のメッセージ履歴
- 新しいメッセージ受信時に全体が再レンダリング
- スクロールが重い

**改善施策**:
1. ✅ VariableSizeListで可変高さ仮想化
2. ✅ メッセージコンポーネントをmemo化
3. ✅ useCallbackでイベントハンドラを最適化

**結果**:
- 初期ロード: 4.5s → 0.3s（15倍高速化）
- 新規メッセージ受信時の再レンダリング: 全件 → 1件のみ
- メモリ使用量: 180MB → 30MB（83%削減）

---

## パフォーマンス最適化チェックリスト

### 必須項目

- [ ] React DevTools Profilerで計測済み
- [ ] 大量データにはreact-windowを使用
- [ ] Route-based Code Splittingを実装
- [ ] lodashなど重いライブラリを見直し
- [ ] keyにindexを使用していない

### 推奨項目

- [ ] React.memoで重いコンポーネントを最適化
- [ ] useMemo/useCallbackで不要な再計算を防止
- [ ] 画像の遅延ロード実装
- [ ] バンドルサイズを定期的に確認
- [ ] Lighthouseスコア90以上

### 避けるべきパターン

- ❌ 全てのコンポーネントをmemo化
- ❌ 計測せずに最適化
- ❌ 早すぎる最適化
- ❌ パフォーマンス問題を無視

---

## まとめ

この章では、Reactアプリケーションのパフォーマンス最適化を学びました。

**重要ポイント**:
- ✅ 計測してから最適化する（推測で最適化しない）
- ✅ React.memo/useMemo/useCallbackを正しく使い分ける
- ✅ Code Splittingで初期ロードを高速化
- ✅ 仮想化で大量データを効率的に表示
- ✅ バンドルサイズを定期的に見直す

**次章予告**: Chapter 5では、Next.js App Routerの実践的な使い方を学びます。

---

**参考リンク**:
- [React Performance](https://react.dev/learn/render-and-commit)
- [react-window](https://github.com/bvaughn/react-window)
- [Web.dev Performance](https://web.dev/performance/)
