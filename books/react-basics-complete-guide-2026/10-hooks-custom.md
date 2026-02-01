---
title: "カスタムフック完全ガイド"
---

# Chapter 10: カスタムフック完全ガイド

再利用可能なロジックを作成するカスタムフックの実装方法を、実践的なパターンと共に学習します。

## useFetch(型安全版)

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

## useLocalStorage(型安全版)

```typescript
function useLocalStorage<T>(
  key: string,
  initialValue: T
): [T, (value: T | ((prev: T) => T)) => void, () => void] {
  // 初期値の取得(Lazy Initialization)
  const [storedValue, setStoredValue] = useState<T>(() => {
    if (typeof window === 'undefined') {
      return initialValue
    }

    try {
      const item = window.localStorage.getItem(key)
      return item ? JSON.parse(item) : initialValue
    } catch (error) {
      console.error(`Error reading localStorage key "${key}":`, error)
      return initialValue
    }
  })

  // 値の設定
  const setValue = (value: T | ((prev: T) => T)) => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value
      setStoredValue(valueToStore)

      if (typeof window !== 'undefined') {
        window.localStorage.setItem(key, JSON.stringify(valueToStore))
      }
    } catch (error) {
      console.error(`Error setting localStorage key "${key}":`, error)
    }
  }

  // 値の削除
  const removeValue = () => {
    try {
      setStoredValue(initialValue)

      if (typeof window !== 'undefined') {
        window.localStorage.removeItem(key)
      }
    } catch (error) {
      console.error(`Error removing localStorage key "${key}":`, error)
    }
  }

  return [storedValue, setValue, removeValue]
}

// 使用例
interface Theme {
  mode: 'light' | 'dark'
  primaryColor: string
}

function ThemeSettings() {
  const [theme, setTheme, resetTheme] = useLocalStorage<Theme>('theme', {
    mode: 'light',
    primaryColor: '#3b82f6'
  })

  const toggleMode = () => {
    setTheme(prev => ({
      ...prev,
      mode: prev.mode === 'light' ? 'dark' : 'light'
    }))
  }

  return (
    <div>
      <p>Current mode: {theme.mode}</p>
      <button onClick={toggleMode}>Toggle Mode</button>
      <button onClick={resetTheme}>Reset to Default</button>
    </div>
  )
}
```

## useDebounce(パフォーマンス最適化)

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

// 使用例: リアルタイム検索
function SearchInput() {
  const [searchTerm, setSearchTerm] = useState('')
  const debouncedSearchTerm = useDebounce(searchTerm, 500)
  const [results, setResults] = useState<string[]>([])

  useEffect(() => {
    if (debouncedSearchTerm) {
      // API呼び出しは500ms後のみ(入力中は呼ばれない)
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

## useThrottle(スクロールイベント用)

```typescript
function useThrottle<T>(value: T, delay: number = 500): T {
  const [throttledValue, setThrottledValue] = useState<T>(value)
  const lastRan = useRef(Date.now())

  useEffect(() => {
    const handler = setTimeout(() => {
      if (Date.now() - lastRan.current >= delay) {
        setThrottledValue(value)
        lastRan.current = Date.now()
      }
    }, delay - (Date.now() - lastRan.current))

    return () => {
      clearTimeout(handler)
    }
  }, [value, delay])

  return throttledValue
}

// 使用例: スクロール位置の追跡
function ScrollTracker() {
  const [scrollY, setScrollY] = useState(0)
  const throttledScrollY = useThrottle(scrollY, 100)

  useEffect(() => {
    const handleScroll = () => {
      setScrollY(window.scrollY)
    }

    window.addEventListener('scroll', handleScroll)
    return () => window.removeEventListener('scroll', handleScroll)
  }, [])

  // 100msごとにのみ更新される
  return <div>Scroll position: {throttledScrollY}px</div>
}
```

## useToggle(便利なヘルパー)

```typescript
function useToggle(
  initialValue: boolean = false
): [boolean, () => void, (value: boolean) => void] {
  const [value, setValue] = useState(initialValue)

  const toggle = useCallback(() => {
    setValue(prev => !prev)
  }, [])

  const setExplicit = useCallback((newValue: boolean) => {
    setValue(newValue)
  }, [])

  return [value, toggle, setExplicit]
}

// 使用例
function Modal() {
  const [isOpen, toggle, setIsOpen] = useToggle(false)

  return (
    <>
      <button onClick={toggle}>Toggle Modal</button>
      <button onClick={() => setIsOpen(true)}>Open Modal</button>
      {isOpen && (
        <div className="modal">
          <p>Modal Content</p>
          <button onClick={toggle}>Close</button>
        </div>
      )}
    </>
  )
}
```

## useAsync(汎用非同期処理)

```typescript
type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error }

function useAsync<T>(
  asyncFunction: () => Promise<T>,
  immediate: boolean = true
) {
  const [state, setState] = useState<AsyncState<T>>({ status: 'idle' })

  const execute = useCallback(async () => {
    setState({ status: 'loading' })

    try {
      const data = await asyncFunction()
      setState({ status: 'success', data })
      return data
    } catch (error) {
      setState({ status: 'error', error: error as Error })
      throw error
    }
  }, [asyncFunction])

  useEffect(() => {
    if (immediate) {
      execute()
    }
  }, [immediate, execute])

  return { ...state, execute }
}

// 使用例
function UserProfile({ userId }: { userId: string }) {
  const fetchUser = useCallback(
    () => fetch(`/api/users/${userId}`).then(res => res.json()),
    [userId]
  )

  const { status, data: user, error, execute } = useAsync<User>(fetchUser)

  if (status === 'loading') return <Spinner />
  if (status === 'error') return <ErrorMessage error={error} />
  if (status === 'success') {
    return (
      <>
        <h1>{user.name}</h1>
        <button onClick={execute}>Refresh</button>
      </>
    )
  }
  return null
}
```

## まとめ

この章では、カスタムフックの実装方法を学習しました。

**重要ポイント:**

- カスタムフックは`use`プレフィックスで命名する
- ジェネリクスを活用して型安全性を確保
- 再利用可能なロジックをカプセル化
- 適切なクリーンアップ処理を実装
- useFetch、useLocalStorage、useDebounce、useThrottleなど、実用的なパターンを習得

カスタムフックを活用することで、コンポーネント間でロジックを共有し、コードの保守性と可読性を向上させることができます。これでReact Hooksの基礎は完璧です。次の章では、さらなる学習へのステップを紹介します。
