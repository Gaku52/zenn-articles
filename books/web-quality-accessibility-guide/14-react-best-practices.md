---
title: "Reactベストプラクティス - 品質の高いコンポーネント"
---

# Reactベストプラクティス - 品質の高いコンポーネント

## Reactの基本原則

React開発における品質とアクセシビリティのベストプラクティスを解説します。

## コンポーネント設計

### 1. Single Responsibility Principle

コンポーネントは単一の責任を持つべきです。

```tsx
// ✅ 良い例: 責任が明確
function UserAvatar({ user }: { user: User }) {
  return (
    <img
      src={user.avatar}
      alt={`${user.name}のアバター`}
      className="w-10 h-10 rounded-full"
    />
  )
}

function UserProfile({ user }: { user: User }) {
  return (
    <div>
      <UserAvatar user={user} />
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  )
}

// ❌ 悪い例: 責任が多すぎる
function UserProfileWithEverything({ userId }: { userId: string }) {
  const [user, setUser] = useState(null)
  const [posts, setPosts] = useState([])
  const [comments, setComments] = useState([])
  // ... 複雑すぎる
}
```

### 2. Props Destructuring

```tsx
// ✅ 良い例
function Button({ label, onClick, disabled }: ButtonProps) {
  return (
    <button onClick={onClick} disabled={disabled}>
      {label}
    </button>
  )
}

// ❌ 悪い例
function Button(props: ButtonProps) {
  return (
    <button onClick={props.onClick} disabled={props.disabled}>
      {props.label}
    </button>
  )
}
```

## Hooksのベストプラクティス

### useState

```tsx
// ✅ 良い例: 関連する状態をまとめる
interface FormState {
  name: string
  email: string
  errors: Record<string, string>
}

const [form, setForm] = useState<FormState>({
  name: '',
  email: '',
  errors: {},
})

// ❌ 悪い例: 分散した状態
const [name, setName] = useState('')
const [email, setEmail] = useState('')
const [errors, setErrors] = useState({})
```

### useEffect

```tsx
// ✅ 良い例: 依存配列を正しく指定
useEffect(() => {
  fetchData(userId)
}, [userId])

// ❌ 悪い例: 空の依存配列
useEffect(() => {
  fetchData(userId) // userIdが変わっても再実行されない
}, [])
```

### カスタムフック

```tsx
// ✅ 良い例: ロジックを再利用可能に
function useUser(userId: string) {
  const [user, setUser] = useState<User | null>(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<Error | null>(null)

  useEffect(() => {
    fetchUser(userId)
      .then(setUser)
      .catch(setError)
      .finally(() => setLoading(false))
  }, [userId])

  return { user, loading, error }
}

// 使用
function UserProfile({ userId }: { userId: string }) {
  const { user, loading, error } = useUser(userId)

  if (loading) return <div>読み込み中...</div>
  if (error) return <div>エラー: {error.message}</div>
  if (!user) return null

  return <div>{user.name}</div>
}
```

## アクセシビリティのベストプラクティス

### フォーム

```tsx
function ContactForm() {
  const [formData, setFormData] = useState({ name: '', email: '' })
  const [errors, setErrors] = useState<Record<string, string>>({})

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <label htmlFor="name">
          氏名
          <span className="text-red-600" aria-label="必須">*</span>
        </label>
        <input
          id="name"
          type="text"
          value={formData.name}
          onChange={(e) => setFormData({ ...formData, name: e.target.value })}
          required
          aria-required="true"
          aria-invalid={errors.name ? 'true' : 'false'}
          aria-describedby={errors.name ? 'name-error' : undefined}
        />
        {errors.name && (
          <p id="name-error" className="text-red-600">
            {errors.name}
          </p>
        )}
      </div>
      <button type="submit">送信</button>
    </form>
  )
}
```

### ボタン

```tsx
// ✅ 良い例: アイコンボタンにラベル
<button aria-label="メニューを開く">
  <MenuIcon />
</button>

// ✅ 良い例: 無効状態
<button disabled={loading} aria-disabled={loading}>
  {loading ? '処理中...' : '送信'}
</button>
```

## パフォーマンス最適化

### React.memo

```tsx
// ✅ 良い例: 不要な再レンダリングを防ぐ
const UserItem = React.memo(({ user }: { user: User }) => {
  return (
    <div>
      <h3>{user.name}</h3>
      <p>{user.email}</p>
    </div>
  )
})
```

### useMemo、useCallback

```tsx
function UserList({ users }: { users: User[] }) {
  // ✅ 良い例: 重い計算をメモ化
  const sortedUsers = useMemo(() => {
    return users.sort((a, b) => a.name.localeCompare(b.name))
  }, [users])

  // ✅ 良い例: コールバックをメモ化
  const handleClick = useCallback((userId: string) => {
    console.log(userId)
  }, [])

  return (
    <ul>
      {sortedUsers.map(user => (
        <li key={user.id} onClick={() => handleClick(user.id)}>
          {user.name}
        </li>
      ))}
    </ul>
  )
}
```

## まとめ

Reactのベストプラクティスは、コンポーネント設計、Hooks、アクセシビリティ、パフォーマンスのバランスです。

### チェックリスト

- コンポーネントが単一責任
- Hooksの依存配列が正しい
- アクセシビリティ対応（ラベル、ARIA）
- パフォーマンス最適化（memo、useMemo）

## 次のステップ

次章では、Vueのベストプラクティスについて解説します。
