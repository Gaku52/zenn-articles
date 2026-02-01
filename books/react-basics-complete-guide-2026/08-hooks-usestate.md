---
title: "useState完全ガイド"
---

# Chapter 08: useState完全ガイド

Reactの状態管理の基礎となる`useState`フックを、基本から応用まで徹底的に学習します。

## 基本パターン

### プリミティブ型

```typescript
// 数値
const [count, setCount] = useState<number>(0)

// 文字列
const [name, setName] = useState<string>('')

// 真偽値
const [isOpen, setIsOpen] = useState<boolean>(false)

// 型推論が効くので型注釈は省略可能
const [count, setCount] = useState(0) // number型と推論される
```

### オブジェクト型

```typescript
interface User {
  id: string
  name: string
  email: string
  age?: number // オプショナル
}

// 初期値null(ログイン前など)
const [user, setUser] = useState<User | null>(null)

// 初期値あり
const [user, setUser] = useState<User>({
  id: '1',
  name: 'John Doe',
  email: 'john@example.com'
})

// 部分的な更新
setUser(prevUser => ({
  ...prevUser!,
  name: 'Jane Doe'
}))
```

### 配列型

```typescript
// プリミティブ配列
const [tags, setTags] = useState<string[]>([])

// オブジェクト配列
interface Todo {
  id: string
  text: string
  completed: boolean
}

const [todos, setTodos] = useState<Todo[]>([])

// 追加
setTodos(prev => [...prev, { id: '1', text: 'New todo', completed: false }])

// 更新
setTodos(prev =>
  prev.map(todo =>
    todo.id === '1' ? { ...todo, completed: true } : todo
  )
)

// 削除
setTodos(prev => prev.filter(todo => todo.id !== '1'))
```

## ジェネリクスでの抽象化

```typescript
// 汎用的なリスト管理フック
function useList<T>(initialValue: T[] = []) {
  const [list, setList] = useState<T[]>(initialValue)

  const add = (item: T) => {
    setList(prev => [...prev, item])
  }

  const remove = (index: number) => {
    setList(prev => prev.filter((_, i) => i !== index))
  }

  const update = (index: number, item: T) => {
    setList(prev =>
      prev.map((current, i) => (i === index ? item : current))
    )
  }

  const clear = () => {
    setList([])
  }

  return { list, add, remove, update, clear }
}

// 使用例
interface Product {
  id: string
  name: string
  price: number
}

function ShoppingCart() {
  const { list: cart, add, remove } = useList<Product>()

  return (
    <div>
      <button onClick={() => add({ id: '1', name: 'Apple', price: 100 })}>
        Add Apple
      </button>
      <ul>
        {cart.map((item, index) => (
          <li key={item.id}>
            {item.name} - ¥{item.price}
            <button onClick={() => remove(index)}>Remove</button>
          </li>
        ))}
      </ul>
    </div>
  )
}
```

## 複雑な状態管理(Discriminated Union)

```typescript
// 悪い例: 複数のuseState
function BadDataFetching() {
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<Error | null>(null)
  const [data, setData] = useState<User[] | null>(null)

  // 問題: loadingとdataが同時にtrueになる可能性
  // 状態の整合性が保証されない
}

// 良い例: Discriminated Union(タグ付きユニオン)
type FetchState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error }

function GoodDataFetching() {
  const [state, setState] = useState<FetchState<User[]>>({ status: 'idle' })

  const fetchUsers = async () => {
    setState({ status: 'loading' })

    try {
      const response = await fetch('/api/users')
      const data = await response.json()
      setState({ status: 'success', data })
    } catch (error) {
      setState({ status: 'error', error: error as Error })
    }
  }

  // TypeScriptが状態を正しく絞り込む
  if (state.status === 'loading') {
    return <Spinner />
  }

  if (state.status === 'error') {
    return <ErrorMessage message={state.error.message} />
  }

  if (state.status === 'success') {
    return (
      <ul>
        {state.data.map(user => (
          <li key={user.id}>{user.name}</li>
        ))}
      </ul>
    )
  }

  return <button onClick={fetchUsers}>Fetch Users</button>
}
```

## Lazy Initialization(パフォーマンス最適化)

```typescript
// 毎レンダリングで関数実行
function ExpensiveComponent() {
  const [value] = useState(expensiveComputation()) // 毎回計算される
  return <div>{value}</div>
}

// 初回のみ実行(関数を渡す)
function OptimizedComponent() {
  const [value] = useState(() => expensiveComputation()) // 初回のみ
  return <div>{value}</div>
}

// 実例: localStorageからの読み込み
function useLocalStorageState<T>(key: string, defaultValue: T) {
  const [state, setState] = useState<T>(() => {
    try {
      const item = localStorage.getItem(key)
      return item ? JSON.parse(item) : defaultValue
    } catch (error) {
      console.error(`Error reading localStorage key "${key}":`, error)
      return defaultValue
    }
  })

  return [state, setState] as const
}
```

## Functional Update(競合状態の回避)

```typescript
// 問題: 古い値を参照
function Counter() {
  const [count, setCount] = useState(0)

  const incrementTwice = () => {
    setCount(count + 1) // 0 + 1 = 1
    setCount(count + 1) // 0 + 1 = 1(期待は2)
  }

  return <button onClick={incrementTwice}>{count}</button>
}

// 解決: 関数形式のupdater
function Counter() {
  const [count, setCount] = useState(0)

  const incrementTwice = () => {
    setCount(prev => prev + 1) // 0 + 1 = 1
    setCount(prev => prev + 1) // 1 + 1 = 2(正しい)
  }

  return <button onClick={incrementTwice}>{count}</button>
}

// 非同期処理での安全性
function AsyncCounter() {
  const [count, setCount] = useState(0)

  const incrementAfterDelay = () => {
    setTimeout(() => {
      setCount(prev => prev + 1) // 常に最新の値を参照
    }, 1000)
  }

  return <button onClick={incrementAfterDelay}>{count}</button>
}
```

## まとめ

この章では、`useState`の基本から応用まで学習しました。

**重要ポイント:**

- TypeScriptの型定義で安全性を確保
- ジェネリクスで再利用可能なパターンを構築
- Discriminated Unionで複雑な状態を管理
- Lazy Initializationでパフォーマンス最適化
- Functional Updateで競合状態を回避

これらのパターンを理解することで、Reactアプリケーションの状態管理を効率的に実装できるようになります。次の章では、副作用を扱う`useEffect`について学習します。
