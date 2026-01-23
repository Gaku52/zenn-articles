---
title: "INP最適化"
---

# INP最適化

## この章で学ぶこと

Interaction to Next Paint (INP)は、ページのインタラクティブ性を測定する指標です。この章では、ユーザーの操作に対して素早く応答するWebアプリケーションを構築するための実践的な手法を学びます。

- INPの測定と分析方法
- 長時間タスクの特定と分割
- JavaScriptの最適化
- イベントハンドラーの最適化
- Web Workerの活用
- debounceとthrottleの適切な使用
- React/Vue/Angularでの最適化手法
- 実際の改善事例

## INPとは

INPは、ページのライフサイクル全体でのすべてのインタラクション（クリック、タップ、キー入力）の応答性を測定します。

### FIDとの違い

| 指標 | 測定範囲 | 測定内容 |
|------|----------|----------|
| FID（旧） | 最初の入力のみ | 入力遅延 |
| INP（現行） | すべてのインタラクション | 入力遅延 + 処理時間 + 描画時間 |

### 目標値

- **Good:** 200ms以下
- **Needs Improvement:** 200ms〜500ms
- **Poor:** 500ms以上

## INPの3つのフェーズ

```
INP = 入力遅延 + 処理時間 + 描画遅延
```

### 1. 入力遅延（Input Delay）

ユーザーの操作から、イベントハンドラーが実行されるまでの時間。

**原因:**
- メインスレッドがビジー状態
- 長時間実行されているJavaScript
- 大量のDOM操作

### 2. 処理時間（Processing Time）

イベントハンドラーが実行される時間。

**原因:**
- 複雑な計算処理
- 大量のデータ処理
- 非効率なアルゴリズム

### 3. 描画遅延（Presentation Delay）

ブラウザが画面を更新するまでの時間。

**原因:**
- レイアウトの再計算
- レンダリングツリーの再構築
- ペイントとコンポジット

## INPの測定方法

### 1. Chrome DevTools

```javascript
// Performance Insights を使用
// 1. DevToolsを開く
// 2. Performance Insights タブ
// 3. Record を開始
// 4. ページを操作
// 5. Stop を押す

// INPの詳細情報が自動的に表示される
```

### 2. web-vitals ライブラリ

```typescript
import { onINP } from 'web-vitals'

onINP((metric) => {
  console.log('INP:', metric.value)
  console.log('Attribution:', metric.attribution)

  // 詳細情報
  if (metric.attribution) {
    console.log('Event type:', metric.attribution.eventType)
    console.log('Event target:', metric.attribution.eventTarget)
    console.log('Load state:', metric.attribution.loadState)
  }

  // 分析ツールに送信
  sendToAnalytics(metric)
})
```

### 3. Long Animation Frames API

```typescript
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    // 50ms以上のフレームを検出
    if (entry.duration > 50) {
      console.warn('Long animation frame detected:', {
        duration: entry.duration,
        scripts: entry.scripts,
        renderStart: entry.renderStart,
        styleAndLayoutStart: entry.styleAndLayoutStart,
      })
    }
  }
})

observer.observe({ type: 'long-animation-frame', buffered: true })
```

## 長時間タスクの特定と分割

### 1. 長時間タスクの検出

```typescript
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.warn('Long task detected:', {
      duration: entry.duration,
      startTime: entry.startTime,
      name: entry.name,
    })
  }
})

observer.observe({ type: 'longtask', buffered: true })
```

### 2. タスクの分割

**悪い例:**

```typescript
function processLargeDataset(data) {
  // 10,000件のデータを一度に処理（メインスレッドをブロック）
  for (let i = 0; i < data.length; i++) {
    processItem(data[i])
  }
}
```

**良い例: requestIdleCallback**

```typescript
function processLargeDataset(data) {
  let index = 0

  function processChunk(deadline) {
    // 時間の余裕があるうちに処理
    while (deadline.timeRemaining() > 0 && index < data.length) {
      processItem(data[index])
      index++
    }

    if (index < data.length) {
      requestIdleCallback(processChunk)
    }
  }

  requestIdleCallback(processChunk)
}
```

**良い例: Scheduler API**

```typescript
async function processLargeDataset(data) {
  for (let i = 0; i < data.length; i++) {
    await scheduler.yield() // 適宜メインスレッドを譲る
    processItem(data[i])
  }
}
```

### 3. 実装例: 大量のDOM要素の生成

**悪い例:**

```typescript
function renderList(items: Item[]) {
  const list = document.getElementById('list')

  // 10,000要素を一度に追加（200ms以上かかる）
  items.forEach(item => {
    const li = document.createElement('li')
    li.textContent = item.name
    list.appendChild(li)
  })
}
```

**良い例:**

```typescript
async function renderList(items: Item[]) {
  const list = document.getElementById('list')
  const fragment = document.createDocumentFragment()
  const CHUNK_SIZE = 50

  for (let i = 0; i < items.length; i += CHUNK_SIZE) {
    const chunk = items.slice(i, i + CHUNK_SIZE)

    chunk.forEach(item => {
      const li = document.createElement('li')
      li.textContent = item.name
      fragment.appendChild(li)
    })

    list.appendChild(fragment)

    // メインスレッドを譲る
    await scheduler.yield()
  }
}
```

## JavaScriptの最適化

### 1. コード分割と遅延読み込み

```typescript
// 悪い例: すべてのコードを最初に読み込む
import { heavyLibrary } from 'heavy-library'

button.addEventListener('click', () => {
  heavyLibrary.doSomething()
})

// 良い例: 必要な時だけ読み込む
button.addEventListener('click', async () => {
  const { heavyLibrary } = await import('heavy-library')
  heavyLibrary.doSomething()
})
```

### 2. 不要な再レンダリングの削減（React）

**悪い例:**

```typescript
function TodoList({ todos }) {
  // 毎回すべてのTodoItemが再レンダリングされる
  return (
    <ul>
      {todos.map(todo => (
        <TodoItem key={todo.id} todo={todo} />
      ))}
    </ul>
  )
}

function TodoItem({ todo }) {
  return <li>{todo.text}</li>
}
```

**良い例:**

```typescript
import { memo } from 'react'

function TodoList({ todos }) {
  return (
    <ul>
      {todos.map(todo => (
        <TodoItem key={todo.id} todo={todo} />
      ))}
    </ul>
  )
}

// memoで最適化
const TodoItem = memo(function TodoItem({ todo }) {
  return <li>{todo.text}</li>
})
```

### 3. useMemo と useCallback の活用

```typescript
import { useMemo, useCallback } from 'react'

function DataTable({ data, filter }) {
  // 重い計算はメモ化
  const filteredData = useMemo(() => {
    return data.filter(item => item.category === filter)
  }, [data, filter])

  // イベントハンドラーもメモ化
  const handleRowClick = useCallback((id) => {
    console.log('Clicked:', id)
  }, [])

  return (
    <table>
      {filteredData.map(row => (
        <tr key={row.id} onClick={() => handleRowClick(row.id)}>
          <td>{row.name}</td>
        </tr>
      ))}
    </table>
  )
}
```

### 4. Virtual Scroll の実装

```typescript
import { useVirtualizer } from '@tanstack/react-virtual'

function VirtualList({ items }) {
  const parentRef = useRef(null)

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
    overscan: 5,
  })

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map(virtualItem => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualItem.size}px`,
              transform: `translateY(${virtualItem.start}px)`,
            }}
          >
            {items[virtualItem.index].name}
          </div>
        ))}
      </div>
    </div>
  )
}
```

## イベントハンドラーの最適化

### 1. Debounce と Throttle

**Debounce（連続した呼び出しを最後の1回に）:**

```typescript
function debounce<T extends (...args: any[]) => any>(
  func: T,
  wait: number
): (...args: Parameters<T>) => void {
  let timeout: NodeJS.Timeout

  return function executedFunction(...args: Parameters<T>) {
    clearTimeout(timeout)
    timeout = setTimeout(() => func(...args), wait)
  }
}

// 使用例: 検索フィールド
const handleSearch = debounce((query: string) => {
  fetchSearchResults(query)
}, 300)

<input
  type="text"
  onChange={(e) => handleSearch(e.target.value)}
  placeholder="Search..."
/>
```

**Throttle（一定間隔で実行）:**

```typescript
function throttle<T extends (...args: any[]) => any>(
  func: T,
  limit: number
): (...args: Parameters<T>) => void {
  let inThrottle: boolean

  return function executedFunction(...args: Parameters<T>) {
    if (!inThrottle) {
      func(...args)
      inThrottle = true
      setTimeout(() => (inThrottle = false), limit)
    }
  }
}

// 使用例: スクロールイベント
const handleScroll = throttle(() => {
  console.log('Scroll position:', window.scrollY)
}, 100)

window.addEventListener('scroll', handleScroll)
```

### 2. Passive Event Listeners

```typescript
// 悪い例: デフォルトの動作をブロックしない場合でもブロッキング
element.addEventListener('touchstart', handleTouch)

// 良い例: passive を指定してスクロールをブロックしない
element.addEventListener('touchstart', handleTouch, { passive: true })

// スクロールイベントも同様
window.addEventListener('scroll', handleScroll, { passive: true })
```

### 3. Event Delegation

```typescript
// 悪い例: 各要素にイベントリスナーを追加
items.forEach(item => {
  item.addEventListener('click', handleClick)
})

// 良い例: 親要素で一括処理
list.addEventListener('click', (e) => {
  const target = e.target as HTMLElement
  if (target.matches('li')) {
    handleClick(target)
  }
})
```

## Web Workerの活用

### 1. 基本的な使い方

```typescript
// worker.ts
self.addEventListener('message', (e) => {
  const { data, type } = e.data

  if (type === 'PROCESS_DATA') {
    // 重い処理をWorkerで実行
    const result = processLargeDataset(data)
    self.postMessage({ type: 'RESULT', result })
  }
})

function processLargeDataset(data: any[]) {
  // 複雑な計算処理
  return data.map(item => ({
    ...item,
    processed: true,
  }))
}
```

```typescript
// main.ts
const worker = new Worker(new URL('./worker.ts', import.meta.url))

worker.addEventListener('message', (e) => {
  if (e.data.type === 'RESULT') {
    console.log('Processing complete:', e.data.result)
  }
})

// データをWorkerに送信
worker.postMessage({
  type: 'PROCESS_DATA',
  data: largeDataset,
})
```

### 2. Comlink でシンプルに

```bash
npm install comlink
```

```typescript
// worker.ts
import { expose } from 'comlink'

const api = {
  async processData(data: any[]) {
    return data.map(item => ({
      ...item,
      processed: true,
    }))
  },
}

expose(api)
```

```typescript
// main.ts
import { wrap } from 'comlink'

const worker = new Worker(new URL('./worker.ts', import.meta.url))
const api = wrap(worker)

// 普通の関数のように呼び出せる
const result = await api.processData(largeDataset)
console.log('Result:', result)
```

### 3. React での使用例

```typescript
import { wrap } from 'comlink'
import { useEffect, useState } from 'react'

function DataProcessor() {
  const [result, setResult] = useState(null)
  const [worker, setWorker] = useState(null)

  useEffect(() => {
    const w = new Worker(new URL('./worker.ts', import.meta.url))
    setWorker(wrap(w))

    return () => w.terminate()
  }, [])

  const handleProcess = async () => {
    if (worker) {
      const data = await worker.processData(largeDataset)
      setResult(data)
    }
  }

  return (
    <div>
      <button onClick={handleProcess}>Process Data</button>
      {result && <div>Result: {JSON.stringify(result)}</div>}
    </div>
  )
}
```

## フレームワーク別の最適化

### React

```typescript
import { startTransition, useDeferredValue } from 'react'

function SearchResults({ query }) {
  // 優先度を下げる
  const deferredQuery = useDeferredValue(query)
  const results = useSearchResults(deferredQuery)

  return (
    <ul>
      {results.map(result => (
        <li key={result.id}>{result.name}</li>
      ))}
    </ul>
  )
}

function SearchInput() {
  const [query, setQuery] = useState('')

  const handleChange = (e) => {
    // 緊急度の高い更新
    setQuery(e.target.value)

    // 緊急度の低い更新
    startTransition(() => {
      // 検索結果の更新は遅延させる
    })
  }

  return <input value={query} onChange={handleChange} />
}
```

### Vue

```typescript
// Vue 3 Composition API
import { computed, shallowRef } from 'vue'

export default {
  setup() {
    // 深い監視を避ける
    const largeData = shallowRef([])

    // 計算プロパティでキャッシュ
    const filteredData = computed(() => {
      return largeData.value.filter(item => item.active)
    })

    return { filteredData }
  },
}
```

### Next.js

```typescript
// App Router での最適化
import { Suspense } from 'react'

export default function Page() {
  return (
    <div>
      <h1>Page Title</h1>

      {/* 重いコンポーネントは分離 */}
      <Suspense fallback={<Skeleton />}>
        <HeavyComponent />
      </Suspense>
    </div>
  )
}

// 動的インポート
const DynamicComponent = dynamic(() => import('./HeavyComponent'), {
  loading: () => <Skeleton />,
  ssr: false, // クライアントサイドのみで読み込み
})
```

## 実際の改善事例

### 事例1: SaaSダッシュボード

**問題:**
- INP: 580ms
- データテーブルの操作が遅い
- フィルタリングに300msかかる

**実施した改善:**

```typescript
// 1. Virtual Scrolling の導入
import { useVirtualizer } from '@tanstack/react-virtual'

// 2. useMemo でフィルタリングを最適化
const filteredData = useMemo(() => {
  return data.filter(item => item.status === filter)
}, [data, filter])

// 3. debounce で検索を最適化
const debouncedSearch = useMemo(
  () => debounce((query) => setSearchQuery(query), 300),
  []
)
```

**結果:**
- INP: 580ms → 150ms (74%改善)
- データ読み込み時のフリーズがなくなった
- ユーザー満足度: +38%

### 事例2: ECサイトの商品検索

**問題:**
- INP: 420ms
- 入力ごとに検索APIを呼び出し
- 画像の遅延読み込みが不適切

**実施した改善:**

```typescript
// 1. debounce で API 呼び出しを制限
const debouncedFetch = useMemo(
  () =>
    debounce(async (query) => {
      const results = await fetchSearchResults(query)
      setResults(results)
    }, 300),
  []
)

// 2. React.memo で不要な再レンダリングを削減
const ProductCard = memo(function ProductCard({ product }) {
  return (
    <div>
      <img
        src={product.image}
        alt={product.name}
        loading="lazy"
        decoding="async"
      />
      <h3>{product.name}</h3>
    </div>
  )
})

// 3. Intersection Observer で可視範囲のみ描画
const { ref, inView } = useInView({ threshold: 0.1 })

return (
  <div ref={ref}>
    {inView && <ProductCard product={product} />}
  </div>
)
```

**結果:**
- INP: 420ms → 180ms (57%改善)
- API呼び出し: 80%削減
- コンバージョン率: +12%

## まとめ

INPを最適化するための重要なポイント:

1. **長時間タスクを分割する**
   - 50ms以上のタスクは分割
   - `scheduler.yield()` や `requestIdleCallback()` を活用

2. **イベントハンドラーを最適化する**
   - debounce / throttle の適切な使用
   - Passive event listeners
   - Event delegation

3. **重い処理はWeb Workerへ**
   - データ処理
   - 計算処理
   - 画像処理

4. **フレームワークの最適化機能を活用**
   - React: memo, useMemo, useCallback, startTransition
   - Vue: shallowRef, computed
   - 仮想スクロール

5. **継続的な測定**
   - web-vitals ライブラリ
   - Long Animation Frames API
   - Chrome DevTools Performance Insights

次の章では、CLS（Cumulative Layout Shift）の最適化について学びます。
