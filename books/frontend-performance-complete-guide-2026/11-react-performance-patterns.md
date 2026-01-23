---
title: "Reactパフォーマンスパターン"
---

# Reactパフォーマンスパターン

## この章で学ぶこと

Reactアプリケーションのパフォーマンス最適化のためのパターンと実践的なテクニックを学びます。

## React.memo

不要な再レンダリングを防ぐ基本的な最適化。

```typescript
const ExpensiveComponent = React.memo(function ExpensiveComponent({ data }) {
  return <div>{/* 複雑なレンダリング */}</div>
})

// Props比較関数でカスタマイズ
const MemoizedComponent = React.memo(
  Component,
  (prevProps, nextProps) => {
    return prevProps.id === nextProps.id
  }
)
```

## useMemo と useCallback

### useMemo

高コストな計算結果をメモ化。

```typescript
function DataTable({ data, filter }) {
  // フィルタリング結果をメモ化
  const filteredData = useMemo(() => {
    return data.filter(item => item.category === filter)
  }, [data, filter])

  return <Table data={filteredData} />
}
```

### useCallback

関数をメモ化して子コンポーネントの再レンダリングを防ぐ。

```typescript
function Parent() {
  const [count, setCount] = useState(0)

  // 関数をメモ化
  const handleClick = useCallback(() => {
    setCount(c => c + 1)
  }, [])

  return <Child onClick={handleClick} />
}

const Child = React.memo(({ onClick }) => {
  return <button onClick={onClick}>Click</button>
})
```

## 状態管理の最適化

### Context の分割

```typescript
// 悪い例: 1つのContextにすべての状態
const AppContext = createContext({ user, theme, data })

// 良い例: 用途別に分割
const UserContext = createContext(user)
const ThemeContext = createContext(theme)
const DataContext = createContext(data)
```

### useReducer + Context

```typescript
const StateContext = createContext(null)
const DispatchContext = createContext(null)

function Provider({ children }) {
  const [state, dispatch] = useReducer(reducer, initialState)

  return (
    <StateContext.Provider value={state}>
      <DispatchContext.Provider value={dispatch}>
        {children}
      </DispatchContext.Provider>
    </StateContext.Provider>
  )
}
```

### Zustand での最適化

```typescript
import create from 'zustand'

// 必要な状態のみ購読
const useStore = create((set) => ({
  count: 0,
  user: null,
  increment: () => set((state) => ({ count: state.count + 1 })),
}))

function Counter() {
  // countのみ購読（userが変わっても再レンダリングなし）
  const count = useStore((state) => state.count)
  const increment = useStore((state) => state.increment)

  return <button onClick={increment}>{count}</button>
}
```

## リスト最適化

### key の適切な使用

```typescript
// 悪い例: インデックスをkeyに
{items.map((item, index) => (
  <Item key={index} data={item} />
))}

// 良い例: 一意のIDをkeyに
{items.map(item => (
  <Item key={item.id} data={item} />
))}
```

### 仮想化

```typescript
import { useVirtualizer } from '@tanstack/react-virtual'

function VirtualList({ items }) {
  const parentRef = useRef(null)

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
  })

  return (
    <div ref={parentRef} style={{ height: '400px', overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize() }}>
        {virtualizer.getVirtualItems().map(virtualItem => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
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

## Concurrent Features

### startTransition

```typescript
import { startTransition, useState } from 'react'

function SearchResults() {
  const [query, setQuery] = useState('')
  const [results, setResults] = useState([])

  const handleChange = (e) => {
    // 入力は即座に反映
    setQuery(e.target.value)

    // 検索は低優先度
    startTransition(() => {
      const filtered = filterResults(e.target.value)
      setResults(filtered)
    })
  }

  return (
    <>
      <input value={query} onChange={handleChange} />
      <ResultsList results={results} />
    </>
  )
}
```

### useDeferredValue

```typescript
function SearchPage({ query }) {
  const deferredQuery = useDeferredValue(query)
  const results = useMemo(
    () => searchDatabase(deferredQuery),
    [deferredQuery]
  )

  return <Results items={results} />
}
```

## 改善事例

### 事例: Todoアプリ

**Before:**
- 100件のTodoで再レンダリング: 850ms
- 入力時にラグ

**After:**

```typescript
// React.memo + useCallback
const TodoItem = React.memo(({ todo, onToggle }) => (
  <li onClick={() => onToggle(todo.id)}>
    {todo.text}
  </li>
))

function TodoList({ todos }) {
  const handleToggle = useCallback((id) => {
    setTodos(todos =>
      todos.map(t => t.id === id ? { ...t, done: !t.done } : t)
    )
  }, [])

  return todos.map(todo => (
    <TodoItem key={todo.id} todo={todo} onToggle={handleToggle} />
  ))
}
```

**結果:**
- 再レンダリング: 850ms → 45ms (95%改善)
- 入力がスムーズに

## まとめ

Reactパフォーマンス最適化のポイント:

1. **React.memo** で不要な再レンダリング防止
2. **useMemo/useCallback** で値・関数をメモ化
3. **Context分割** で再レンダリング範囲を限定
4. **仮想化** で大量リストを最適化
5. **Concurrent Features** で優先度制御

次の章では、Virtual DOM最適化について学びます。
