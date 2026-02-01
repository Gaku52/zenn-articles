---
title: "useEffect完全ガイド"
---

# Chapter 09: useEffect完全ガイド

副作用を扱う`useEffect`フックの正しい使い方を、実践的なパターンと共に学習します。

## データフェッチパターン

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

## 依存配列の完全理解

### アンチパターン1: 依存配列の欠落

```typescript
// 問題
function SearchResults({ query }: { query: string }) {
  const [results, setResults] = useState([])

  useEffect(() => {
    fetch(`/api/search?q=${query}`)
      .then(res => res.json())
      .then(setResults)
  }, []) // queryが依存に必要(ESLint警告)

  // queryが変わっても検索されない
}

// 解決
function SearchResults({ query }: { query: string }) {
  const [results, setResults] = useState([])

  useEffect(() => {
    fetch(`/api/search?q=${query}`)
      .then(res => res.json())
      .then(setResults)
  }, [query]) // 正しい依存配列
}
```

### アンチパターン2: オブジェクトの依存

```typescript
// 問題: 無限ループ
function DataDisplay() {
  const config = { url: '/api/users', method: 'GET' } // 毎レンダリングで新しいオブジェクト

  useEffect(() => {
    fetch(config.url, { method: config.method })
      .then(res => res.json())
      .then(console.log)
  }, [config]) // configが毎回変わる → 無限ループ
}

// 解決策1: useMemoで安定化
function DataDisplay() {
  const config = useMemo(() => ({
    url: '/api/users',
    method: 'GET' as const
  }), [])

  useEffect(() => {
    fetch(config.url, { method: config.method })
      .then(res => res.json())
      .then(console.log)
  }, [config])
}

// 解決策2: プリミティブ値のみ依存
function DataDisplay() {
  const url = '/api/users'
  const method = 'GET'

  useEffect(() => {
    fetch(url, { method })
      .then(res => res.json())
      .then(console.log)
  }, [url, method])
}

// 解決策3: 依存なし(定数の場合)
function DataDisplay() {
  useEffect(() => {
    fetch('/api/users', { method: 'GET' })
      .then(res => res.json())
      .then(console.log)
  }, [])
}
```

### アンチパターン3: 関数の依存

```typescript
// 問題
function UserList() {
  const fetchUsers = () => {
    return fetch('/api/users').then(res => res.json())
  }

  useEffect(() => {
    fetchUsers().then(console.log)
  }, [fetchUsers]) // 毎レンダリングで新しい関数 → 無限ループ
}

// 解決策1: useCallbackで安定化
function UserList() {
  const fetchUsers = useCallback(() => {
    return fetch('/api/users').then(res => res.json())
  }, [])

  useEffect(() => {
    fetchUsers().then(console.log)
  }, [fetchUsers])
}

// 解決策2: useEffect内で定義
function UserList() {
  useEffect(() => {
    const fetchUsers = () => {
      return fetch('/api/users').then(res => res.json())
    }

    fetchUsers().then(console.log)
  }, [])
}
```

## クリーンアップパターン

### WebSocket接続

```typescript
function ChatRoom({ roomId }: { roomId: string }) {
  const [messages, setMessages] = useState<string[]>([])

  useEffect(() => {
    const ws = new WebSocket(`ws://localhost:8080/rooms/${roomId}`)

    ws.onopen = () => {
      console.log('Connected')
    }

    ws.onmessage = (event) => {
      setMessages(prev => [...prev, event.data])
    }

    ws.onerror = (error) => {
      console.error('WebSocket error:', error)
    }

    // クリーンアップ: 接続を閉じる
    return () => {
      ws.close()
      console.log('Disconnected')
    }
  }, [roomId]) // roomIdが変わったら再接続

  return (
    <ul>
      {messages.map((msg, i) => (
        <li key={i}>{msg}</li>
      ))}
    </ul>
  )
}
```

### タイマー

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

    // クリーンアップ: タイマーを停止
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

### イベントリスナー

```typescript
function WindowSize() {
  const [size, setSize] = useState({
    width: window.innerWidth,
    height: window.innerHeight
  })

  useEffect(() => {
    const handleResize = () => {
      setSize({
        width: window.innerWidth,
        height: window.innerHeight
      })
    }

    window.addEventListener('resize', handleResize)

    // クリーンアップ: リスナーを削除
    return () => {
      window.removeEventListener('resize', handleResize)
    }
  }, [])

  return (
    <div>
      Window size: {size.width} x {size.height}
    </div>
  )
}
```

### Subscription(RxJSなど)

```typescript
import { interval } from 'rxjs'

function ObservableExample() {
  const [count, setCount] = useState(0)

  useEffect(() => {
    const subscription = interval(1000).subscribe(value => {
      setCount(value)
    })

    // クリーンアップ: サブスクリプション解除
    return () => {
      subscription.unsubscribe()
    }
  }, [])

  return <div>Count: {count}</div>
}
```

## useEffect vs useLayoutEffect

```typescript
// useEffect: 画面描画後に実行(通常はこちら)
function NormalEffect() {
  useEffect(() => {
    console.log('Runs after paint')
  })
}

// useLayoutEffect: 画面描画前に実行(DOM測定など)
function LayoutEffect() {
  const [height, setHeight] = useState(0)
  const divRef = useRef<HTMLDivElement>(null)

  useLayoutEffect(() => {
    if (divRef.current) {
      // DOM測定は描画前に行う
      setHeight(divRef.current.offsetHeight)
    }
  })

  return (
    <>
      <div ref={divRef}>Content</div>
      <p>Height: {height}px</p>
    </>
  )
}
```

## まとめ

この章では、`useEffect`の正しい使い方を学習しました。

**重要ポイント:**

- データフェッチ時はクリーンアップフラグを使用
- 依存配列は完全に指定する(ESLintの警告に従う)
- オブジェクトや関数の依存に注意
- クリーンアップ関数で副作用を正しく解除
- WebSocket、タイマー、イベントリスナーなど、適切なクリーンアップパターンを使い分ける

`useEffect`を正しく理解することで、副作用を安全に扱い、メモリリークや無限ループを防ぐことができます。次の章では、再利用可能なロジックを作成するカスタムフックについて学習します。
