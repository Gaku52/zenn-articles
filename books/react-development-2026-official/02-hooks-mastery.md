---
title: "React Hooks 完全マスター"
---

# Chapter 2: React Hooks 完全マスターガイド（TypeScript版）

> 実務で遭遇した100以上のパターンから厳選した実践的ガイド
> 対象: React 18+ / TypeScript 5+

## この章で学べること

この章では、React Hooksを完全にマスターします。

- ✅ useState/useEffect以外の重要なHooks
- ✅ useRef/useCallback/useMemoの使い分け
- ✅ カスタムフックの設計パターン
- ✅ 実務で即使える実践例10個

**前提知識**: Chapter 01の内容を理解していること

**所要時間**: 60-80分

---

## 1. Hooks の基礎（TypeScript視点）

### なぜHooksか

**Class Component の問題点**:
- ライフサイクルメソッドが複雑
- ロジックの再利用が困難
- thisのバインディング問題
- コンポーネントが肥大化しやすい

**Hooks による解決**:
- ✅ 関数コンポーネントで状態管理可能
- ✅ ロジックの再利用（カスタムフック）
- ✅ thisなし、シンプルな構文
- ✅ 関心事ごとにロジックを分離

### TypeScriptとHooksの相性

```typescript
// ❌ JavaScriptでは型安全性なし
const [user, setUser] = useState(null)
setUser({ name: 'John' }) // エラーにならない（危険）

// ✅ TypeScriptで完全な型安全性
interface User {
  id: string
  name: string
  email: string
}

const [user, setUser] = useState<User | null>(null)
setUser({ name: 'John' }) // 型エラー！idとemailが必要
```

### Hooks のルール

#### ルール1: トップレベルでのみ呼び出す

```typescript
// ❌ 条件分岐内でHooks（絶対NG）
function BadComponent({ condition }: { condition: boolean }) {
  if (condition) {
    const [value, setValue] = useState(0) // エラー！
  }
  return <div>{value}</div>
}

// ✅ トップレベルで呼び出す
function GoodComponent({ condition }: { condition: boolean }) {
  const [value, setValue] = useState(0)

  if (condition) {
    return <div>{value}</div>
  }
  return null
}
```

#### ルール2: React関数内でのみ呼び出す

```typescript
// ❌ 通常の関数内でHooks
function helperFunction() {
  const [count, setCount] = useState(0) // エラー！
  return count
}

// ✅ カスタムフック内でHooks
function useCounter() {
  const [count, setCount] = useState(0)
  return { count, setCount }
}
```

### ESLint設定（必須）

```json
{
  "extends": [
    "plugin:react-hooks/recommended"
  ],
  "rules": {
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn"
  }
}
```

---

## 2. useState 完全ガイド

### 基本パターン

#### プリミティブ型

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

#### オブジェクト型

```typescript
interface User {
  id: string
  name: string
  email: string
  age?: number // オプショナル
}

// 初期値null（ログイン前など）
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

#### 配列型

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

### 複雑な状態管理（Discriminated Union）

```typescript
// ❌ 悪い例：複数のuseState
function BadDataFetching() {
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<Error | null>(null)
  const [data, setData] = useState<User[] | null>(null)

  // 問題：loadingとdataが同時にtrueになる可能性
  // 状態の整合性が保証されない
}

// ✅ 良い例：Discriminated Union（タグ付きユニオン）
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

### Functional Update（競合状態の回避）

```typescript
// ❌ 問題：古い値を参照
function Counter() {
  const [count, setCount] = useState(0)

  const incrementTwice = () => {
    setCount(count + 1) // 0 + 1 = 1
    setCount(count + 1) // 0 + 1 = 1（期待は2）
  }

  return <button onClick={incrementTwice}>{count}</button>
}

// ✅ 解決：関数形式のupdater
function Counter() {
  const [count, setCount] = useState(0)

  const incrementTwice = () => {
    setCount(prev => prev + 1) // 0 + 1 = 1
    setCount(prev => prev + 1) // 1 + 1 = 2（正しい）
  }

  return <button onClick={incrementTwice}>{count}</button>
}
```

---

## 3. useEffect 完全ガイド

### データフェッチパターン

```typescript
interface User {
  id: string
  name: string
  email: string
}

function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<Error | null>(null)

  useEffect(() => {
    // クリーンアップフラグ
    let cancelled = false

    const fetchUser = async () => {
      try {
        setLoading(true)
        setError(null)

        const response = await fetch(`/api/users/${userId}`)

        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`)
        }

        const data = await response.json()

        // アンマウント済みなら状態更新しない
        if (!cancelled) {
          setUser(data)
        }
      } catch (err) {
        if (!cancelled) {
          setError(err as Error)
        }
      } finally {
        if (!cancelled) {
          setLoading(false)
        }
      }
    }

    fetchUser()

    // クリーンアップ関数
    return () => {
      cancelled = true
    }
  }, [userId]) // userIdが変わったら再フェッチ

  if (loading) return <Spinner />
  if (error) return <ErrorMessage error={error} />
  if (!user) return null

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  )
}
```

### 依存配列の完全理解

#### アンチパターン1: 依存配列の欠落

```typescript
// ❌ 問題
function SearchResults({ query }: { query: string }) {
  const [results, setResults] = useState([])

  useEffect(() => {
    fetch(`/api/search?q=${query}`)
      .then(res => res.json())
      .then(setResults)
  }, []) // queryが依存に必要（ESLint警告）

  // queryが変わっても検索されない！
}

// ✅ 解決
function SearchResults({ query }: { query: string }) {
  const [results, setResults] = useState([])

  useEffect(() => {
    fetch(`/api/search?q=${query}`)
      .then(res => res.json())
      .then(setResults)
  }, [query]) // 正しい依存配列
}
```

### クリーンアップパターン

#### タイマー

```typescript
function CountdownTimer({ seconds }: { seconds: number }) {
  const [timeLeft, setTimeLeft] = useState(seconds)

  useEffect(() => {
    const timer = setInterval(() => {
      setTimeLeft(prev => {
        if (prev <= 1) {
          return 0
        }
        return prev - 1
      })
    }, 1000)

    // クリーンアップ：タイマーを停止
    return () => {
      clearInterval(timer)
    }
  }, []) // 初回のみ

  useEffect(() => {
    if (timeLeft === 0) {
      alert('Time is up!')
    }
  }, [timeLeft])

  return <div>{timeLeft} seconds remaining</div>
}
```

---

## 4. useRef 完全ガイド

### DOM参照

```typescript
// 基本的なDOM参照
function AutoFocusInput() {
  const inputRef = useRef<HTMLInputElement>(null)

  useEffect(() => {
    // Optional chainingで安全にアクセス
    inputRef.current?.focus()
  }, [])

  return <input ref={inputRef} type="text" />
}
```

### 前の値の保持

```typescript
// 前回のpropsやstateを保持
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T>()

  useEffect(() => {
    ref.current = value
  }, [value])

  return ref.current
}

// 使用例：変更を検出
function Counter() {
  const [count, setCount] = useState(0)
  const prevCount = usePrevious(count)

  const difference = prevCount !== undefined ? count - prevCount : 0

  return (
    <div>
      <p>Current: {count}</p>
      <p>Previous: {prevCount ?? 'N/A'}</p>
      <p>Difference: {difference}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </div>
  )
}
```

---

## 5. カスタムフック 完全ガイド

### useFetch（型安全版）

```typescript
type FetchState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error }

interface UseFetchOptions {
  immediate?: boolean
}

function useFetch<T>(
  url: string,
  options: UseFetchOptions = {}
) {
  const { immediate = true } = options
  const [state, setState] = useState<FetchState<T>>({ status: 'idle' })

  const execute = useCallback(async () => {
    setState({ status: 'loading' })

    try {
      const response = await fetch(url)

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`)
      }

      const data = await response.json()
      setState({ status: 'success', data })
      return data
    } catch (error) {
      const err = error as Error
      setState({ status: 'error', error: err })
      throw err
    }
  }, [url])

  useEffect(() => {
    if (immediate) {
      execute()
    }
  }, [immediate, execute])

  const refetch = execute

  return { ...state, refetch }
}

// 使用例
interface User {
  id: string
  name: string
  email: string
}

function UserList() {
  const { status, data, error, refetch } = useFetch<User[]>('/api/users')

  if (status === 'loading') return <Spinner />
  if (status === 'error') return <ErrorMessage error={error} />
  if (status === 'success') {
    return (
      <>
        <button onClick={refetch}>Refresh</button>
        <ul>
          {data.map(user => (
            <li key={user.id}>{user.name}</li>
          ))}
        </ul>
      </>
    )
  }
  return null
}
```

### useDebounce（パフォーマンス最適化）

```typescript
function useDebounce<T>(value: T, delay: number = 500): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value)

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value)
    }, delay)

    return () => {
      clearTimeout(timer)
    }
  }, [value, delay])

  return debouncedValue
}

// 使用例：リアルタイム検索
function SearchInput() {
  const [searchTerm, setSearchTerm] = useState('')
  const debouncedSearchTerm = useDebounce(searchTerm, 500)
  const [results, setResults] = useState<string[]>([])

  useEffect(() => {
    if (debouncedSearchTerm) {
      // API呼び出しは500ms後のみ（入力中は呼ばれない）
      fetch(`/api/search?q=${debouncedSearchTerm}`)
        .then(res => res.json())
        .then(setResults)
    } else {
      setResults([])
    }
  }, [debouncedSearchTerm])

  return (
    <>
      <input
        type="text"
        value={searchTerm}
        onChange={e => setSearchTerm(e.target.value)}
        placeholder="Search..."
      />
      <ul>
        {results.map(result => (
          <li key={result}>{result}</li>
        ))}
      </ul>
    </>
  )
}
```

---

## 6. 実際の失敗事例 5選

### 1. useEffect無限ループ

**問題**:
```typescript
function UserList() {
  const [users, setUsers] = useState([])

  useEffect(() => {
    fetch('/api/users')
      .then(res => res.json())
      .then(setUsers)
  }, [users]) // usersが依存 → 無限ループ
}
```

**原因**: usersが変わる → useEffectが実行 → usersが変わる → ...

**解決**:
```typescript
useEffect(() => {
  fetch('/api/users')
    .then(res => res.json())
    .then(setUsers)
}, []) // 初回のみ
```

### 2. クリーンアップ忘れによるメモリリーク

**問題**:
```typescript
function Timer() {
  const [count, setCount] = useState(0)

  useEffect(() => {
    const interval = setInterval(() => {
      setCount(c => c + 1)
    }, 1000)
    // クリーンアップなし！
  }, [])
}
```

**原因**: コンポーネントアンマウント後もタイマーが動き続ける

**解決**:
```typescript
useEffect(() => {
  const interval = setInterval(() => {
    setCount(c => c + 1)
  }, 1000)

  return () => clearInterval(interval)
}, [])
```

### 3. 古いクロージャ問題

**問題**:
```typescript
function Counter() {
  const [count, setCount] = useState(0)

  useEffect(() => {
    const interval = setInterval(() => {
      setCount(count + 1) // 常に0 + 1
    }, 1000)

    return () => clearInterval(interval)
  }, [])
}
```

**原因**: useEffectが最初のレンダリング時のcountをキャプチャ

**解決**:
```typescript
useEffect(() => {
  const interval = setInterval(() => {
    setCount(c => c + 1) // 関数形式で最新の値を参照
  }, 1000)

  return () => clearInterval(interval)
}, [])
```

### 4. useCallbackの誤用

**問題**:
```typescript
function Parent() {
  const [count, setCount] = useState(0)

  const handleClick = useCallback(() => {
    console.log(count) // 常に0
  }, []) // countが依存にない

  return <Child onClick={handleClick} />
}
```

**原因**: useCallbackが最初のcountをキャプチャ

**解決**:
```typescript
const handleClick = useCallback(() => {
  console.log(count)
}, [count]) // 正しい依存配列
```

### 5. 型定義の不足

**問題**:
```typescript
const [data, setData] = useState(null) // any型

useEffect(() => {
  fetch('/api/users')
    .then(res => res.json())
    .then(setData) // 型チェックなし
}, [])
```

**解決**:
```typescript
interface User {
  id: string
  name: string
  email: string
}

const [data, setData] = useState<User[] | null>(null)

useEffect(() => {
  fetch('/api/users')
    .then(res => res.json())
    .then((users: User[]) => setData(users))
}, [])
```

---

## まとめ

この章では、React Hooksの実践的な使い方を学びました。

**重要ポイント**:
- ✅ TypeScriptで型安全なHooksを書く
- ✅ useEffectの依存配列を正しく管理する
- ✅ クリーンアップ関数を忘れない
- ✅ カスタムフックでロジックを再利用する
- ✅ よくある失敗パターンを避ける

**次章予告**: Chapter 3では、TypeScriptとReactの実践的なパターンを学びます。

---

**参考リンク**:
- [React Hooks 公式ドキュメント](https://react.dev/reference/react)
- [TypeScript ハンドブック](https://www.typescriptlang.org/docs/handbook/intro.html)
