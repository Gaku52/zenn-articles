---
title: "useEffect完全理解、依存配列の罠"
emoji: "⚛️"
type: "tech"
topics: ["react", "hooks", "useeffect", "frontend"]
published: false
---

## はじめに

Reactを使い始めて最初に躓くのが、おそらく`useEffect`でしょう。私も駆け出しの頃、こんな経験をしました。

検索機能を実装していたとき、検索クエリが変わっても検索結果が更新されない。依存配列に`query`を入れたら今度は無限ループが発生。やっと動いたと思ったら、ページ遷移後もタイマーが動き続けてメモリリークが発生...。

`useEffect`は強力ですが、正しく理解しないと思わぬバグを引き起こします。この記事では、React開発者が陥りがちな**依存配列の3つの罠**と、その正しい解決方法を解説します。

## よくある3つの間違い

### 1. 依存配列の漏れ - 古いデータが表示され続ける

最も多いのが、依存している値を配列に含めないパターンです。

```typescript
// ❌ 悪い例：queryが変わっても検索されない
function SearchResults({ query }: { query: string }) {
  const [results, setResults] = useState<string[]>([])

  useEffect(() => {
    fetch(`/api/search?q=${query}`)
      .then(res => res.json())
      .then(setResults)
  }, []) // queryが依存配列に含まれていない！

  return (
    <ul>
      {results.map((result, i) => (
        <li key={i}>{result}</li>
      ))}
    </ul>
  )
}
```

**何が起きているか：**
- 初回レンダリング時のみ検索が実行される
- `query`が変わっても`useEffect`は再実行されない
- 古い検索結果が表示され続ける

**正しい実装：**

```typescript
// ✅ 良い例：queryが変わったら再検索
function SearchResults({ query }: { query: string }) {
  const [results, setResults] = useState<string[]>([])

  useEffect(() => {
    fetch(`/api/search?q=${query}`)
      .then(res => res.json())
      .then(setResults)
  }, [query]) // queryを依存配列に追加

  return (
    <ul>
      {results.map((result, i) => (
        <li key={i}>{result}</li>
      ))}
    </ul>
  )
}
```

**防ぐ方法：**
ESLintの`exhaustive-deps`ルールを必ずエラーにしましょう。

```json
{
  "extends": ["plugin:react-hooks/recommended"],
  "rules": {
    "react-hooks/exhaustive-deps": "error"
  }
}
```

### 2. 無限ループ - オブジェクトや配列を依存配列に入れる

オブジェクトや配列は、毎レンダリングで新しい参照が作られるため、依存配列に入れると無限ループが発生します。

```typescript
// ❌ 悪い例：無限ループ発生
function DataDisplay() {
  const config = { url: '/api/users', method: 'GET' }

  useEffect(() => {
    fetch(config.url, { method: config.method })
      .then(res => res.json())
      .then(console.log)
  }, [config]) // configが毎回新しいオブジェクト → 無限ループ
}
```

**何が起きているか：**
1. レンダリング → 新しい`config`オブジェクトが作成
2. `useEffect`が実行される（依存配列の`config`が変わったため）
3. 状態更新 → 再レンダリング
4. 1に戻る（無限ループ）

**解決策1：プリミティブ値のみ依存する**

```typescript
// ✅ 良い例：プリミティブ値は参照が安定
function DataDisplay() {
  const url = '/api/users'
  const method = 'GET'

  useEffect(() => {
    fetch(url, { method })
      .then(res => res.json())
      .then(console.log)
  }, [url, method]) // 文字列は安定した参照
}
```

**解決策2：useEffect内で定義する**

```typescript
// ✅ より良い例：定数ならuseEffect内で定義
function DataDisplay() {
  useEffect(() => {
    const config = { url: '/api/users', method: 'GET' }

    fetch(config.url, { method: config.method })
      .then(res => res.json())
      .then(console.log)
  }, []) // 依存なし
}
```

**解決策3：useMemoで安定化する**

```typescript
// ✅ 動的な値の場合：useMemoで安定化
function DataDisplay({ page }: { page: number }) {
  const config = useMemo(() => ({
    url: `/api/users?page=${page}`,
    method: 'GET' as const
  }), [page]) // pageが変わったときのみ再生成

  useEffect(() => {
    fetch(config.url, { method: config.method })
      .then(res => res.json())
      .then(console.log)
  }, [config])
}
```

### 3. クリーンアップ忘れ - メモリリークとバグの温床

`useEffect`が返す関数（クリーンアップ関数）を忘れると、メモリリークや予期しないバグが発生します。

```typescript
// ❌ 悪い例：タイマーがクリアされない
function BadTimer() {
  const [count, setCount] = useState(0)

  useEffect(() => {
    setInterval(() => {
      setCount(c => c + 1)
    }, 1000)
    // クリーンアップがない！
  }, [])

  return <div>{count}</div>
}
```

**何が起きているか：**
- コンポーネントがアンマウントされても、タイマーは動き続ける
- アンマウント済みのコンポーネントに対して`setCount`が呼ばれる
- メモリリークが発生

**正しい実装：**

```typescript
// ✅ 良い例：タイマーをクリア
function GoodTimer() {
  const [count, setCount] = useState(0)

  useEffect(() => {
    const timer = setInterval(() => {
      setCount(c => c + 1)
    }, 1000)

    // クリーンアップ関数でタイマーをクリア
    return () => {
      clearInterval(timer)
    }
  }, [])

  return <div>{count}</div>
}
```

**クリーンアップが必要なケース：**
- イベントリスナーの登録（`addEventListener`）
- タイマー（`setInterval`、`setTimeout`）
- WebSocket接続
- Subscription（RxJS等）
- データフェッチのキャンセル

## データフェッチの正しい実装パターン

データフェッチでは、Race Condition（競合状態）に注意が必要です。

**Race Conditionとは：**
複数の非同期処理が競合して、古いデータで上書きされる問題です。

```typescript
// 問題：
// 1. userId='user1'でフェッチ開始（3秒かかる）
// 2. すぐにuserId='user2'に変更してフェッチ開始（1秒で完了）
// 3. user2のデータが表示される
// 4. その後、user1のフェッチが完了して、古いデータで上書き！
```

**解決策：AbortControllerを使う**

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
    const abortController = new AbortController()

    const fetchUser = async () => {
      try {
        setLoading(true)
        setError(null)

        const response = await fetch(`/api/users/${userId}`, {
          signal: abortController.signal // シグナルを渡す
        })

        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`)
        }

        const data = await response.json()
        setUser(data)
      } catch (err) {
        // AbortErrorは無視
        if ((err as Error).name !== 'AbortError') {
          setError(err as Error)
        }
      } finally {
        setLoading(false)
      }
    }

    fetchUser()

    // クリーンアップ：リクエストをキャンセル
    return () => {
      abortController.abort()
    }
  }, [userId]) // userIdが変わったら再フェッチ

  if (loading) return <div>Loading...</div>
  if (error) return <div>Error: {error.message}</div>
  if (!user) return null

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  )
}
```

**このコードのポイント：**
1. `AbortController`でリクエストをキャンセル可能にする
2. クリーンアップ関数で古いリクエストをキャンセル
3. `AbortError`は無視する（正常なキャンセル）
4. 常に最新のデータのみが表示される

## カスタムHookで再利用可能にする

データフェッチのロジックはカスタムHookにまとめると便利です。

```typescript
function useFetch<T>(url: string) {
  const [data, setData] = useState<T | null>(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<Error | null>(null)

  useEffect(() => {
    const abortController = new AbortController()

    const fetchData = async () => {
      try {
        setLoading(true)
        setError(null)

        const response = await fetch(url, {
          signal: abortController.signal
        })

        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`)
        }

        const json = await response.json()
        setData(json)
      } catch (err) {
        if ((err as Error).name !== 'AbortError') {
          setError(err as Error)
        }
      } finally {
        setLoading(false)
      }
    }

    fetchData()

    return () => {
      abortController.abort()
    }
  }, [url])

  return { data, loading, error }
}

// 使用例
function UserProfile({ userId }: { userId: string }) {
  const { data: user, loading, error } = useFetch<User>(
    `/api/users/${userId}`
  )

  if (loading) return <div>Loading...</div>
  if (error) return <div>Error: {error.message}</div>
  if (!user) return null

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  )
}
```

## まとめ

`useEffect`を正しく使うための重要ポイントをまとめます：

**1. 依存配列を正しく設定する**
- ESLintの`exhaustive-deps`ルールを守る
- オブジェクト/配列は`useMemo`で安定化
- または`useEffect`内で定義する

**2. クリーンアップ関数を忘れない**
- イベントリスナーは必ず削除
- タイマーは必ずクリア
- WebSocket/Subscriptionは必ず解除

**3. データフェッチはRace Condition対策を**
- `AbortController`でリクエストをキャンセル
- 古いデータで上書きされるのを防ぐ

**チェックリスト：**
- [ ] 依存配列は正しく設定されているか？（ESLintの警告を無視していないか？）
- [ ] クリーンアップ関数は必要ないか？
- [ ] データフェッチでRace Condition対策をしているか？
- [ ] 無限ループの可能性はないか？
- [ ] メモリリークの可能性はないか？

## ⚛️ さらにHooksを極める

### 書籍で学べる高度なReact開発

✅ **カスタムHooks設計パターン**
- 再利用可能なロジック抽出
- useState + useEffectの組み合わせ
- useReducerとの使い分け

✅ **TypeScript型定義完全版**
- ジェネリクス活用
- 型安全なコンポーネント設計
- Props型推論

✅ **パフォーマンス最適化**
- useMemo/useCallback使い分け
- React.memo実践
- コード分割とLazy Loading

✅ **よくある間違いTOP10**
- 実務でつまずくポイント完全網羅
- Before/Afterコード比較
- デバッグ手法

📚 **React実践テクニック**（21万字）
👉 https://zenn.dev/gaku52/books/react-advanced-techniques

---

**参考リンク：**
- [React公式ドキュメント - useEffect](https://react.dev/reference/react/useEffect)
- [A Complete Guide to useEffect by Dan Abramov](https://overreacted.io/a-complete-guide-to-useeffect/)
