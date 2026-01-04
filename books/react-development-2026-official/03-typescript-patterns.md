---
title: "TypeScript × React 実践パターン"
---

# Chapter 3: React × TypeScript パターン完全ガイド

> 型安全なReactアプリケーション開発のための実践的パターン集

## この章で学べること

この章では、TypeScriptとReactを組み合わせた実践的なパターンを学びます。

- ✅ Propsの型定義のベストプラクティス
- ✅ ジェネリクスを使った再利用可能なコンポーネント
- ✅ イベントハンドラの型安全な実装
- ✅ 高度な型テクニック（Discriminated Union, Conditional Types）

**前提知識**: TypeScriptの基礎、Chapter 01-02の内容

**所要時間**: 50-70分

---

## 1. コンポーネントの型定義

### React.FC vs 通常の関数

```typescript
// ❌ React.FC（非推奨 - 暗黙的なchildrenが問題）
const Component: React.FC<Props> = ({ name }) => {
  return <div>{name}</div>
}

// ✅ 通常の関数（推奨）
interface Props {
  name: string
}

function Component({ name }: Props) {
  return <div>{name}</div>
}

// ✅ アロー関数（推奨）
const Component = ({ name }: Props) => {
  return <div>{name}</div>
}
```typescript

**React.FCを使わない理由**:
- 暗黙的に`children`を含む（型安全性が低下）
- ジェネリクスとの相性が悪い
- デフォルトPropsとの互換性問題

### Props の基本的な型定義

```typescript
// プリミティブ型
interface BasicProps {
  title: string
  count: number
  isActive: boolean
}

// オプショナル
interface OptionalProps {
  title: string
  subtitle?: string // オプショナル
}

// ユニオン型
interface UnionProps {
  variant: 'primary' | 'secondary' | 'danger'
  size: 'sm' | 'md' | 'lg'
}

// オブジェクト型
interface User {
  id: string
  name: string
  email: string
}

interface ObjectProps {
  user: User
  onUpdate: (user: User) => void
}
```typescript

### HTML属性の継承

```typescript
// HTMLButtonElement の属性を継承
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary'
}

function Button({ variant = 'primary', children, ...props }: ButtonProps) {
  return (
    <button className={variant} {...props}>
      {children}
    </button>
  )
}

// 使用例（すべてのbutton属性が使える）
<Button
  variant="primary"
  onClick={() => console.log('clicked')}
  disabled
  type="submit"
  aria-label="Submit button"
>
  Submit
</Button>
```typescript

---

## 2. Props の高度な型パターン

### Discriminated Union（条件付きProps）

```typescript
// ❌ 問題：variantによってpropsが変わるが、型で表現できていない
interface BadButtonProps {
  variant: 'link' | 'button'
  href?: string // linkの時のみ必要
  onClick?: () => void // buttonの時のみ必要
}

// ✅ 解決：Discriminated Union
type ButtonProps =
  | {
      variant: 'link'
      href: string
      onClick?: never // buttonの時は使えない
    }
  | {
      variant: 'button'
      onClick: () => void
      href?: never // linkの時は使えない
    }

function Button(props: ButtonProps) {
  if (props.variant === 'link') {
    // TypeScriptがprops.hrefの存在を保証
    return <a href={props.href}>Link</a>
  }

  // TypeScriptがprops.onClickの存在を保証
  return <button onClick={props.onClick}>Button</button>
}

// 使用例
<Button variant="link" href="/home" /> // ✅ OK
<Button variant="button" onClick={() => {}} /> // ✅ OK
<Button variant="link" onClick={() => {}} /> // ❌ 型エラー
<Button variant="button" href="/home" /> // ❌ 型エラー
```typescript

### Utility Types（Omit, Pick, Partial）

```typescript
// 元の型
interface FullUser {
  id: string
  name: string
  email: string
  password: string
  createdAt: Date
}

// Omit: パスワードを除外した型
type PublicUser = Omit<FullUser, 'password'>

// Pick: 必要なプロパティのみ抽出
type UserSummary = Pick<FullUser, 'id' | 'name'>

// Partial: 全てのプロパティをオプショナルに
type PartialUser = Partial<FullUser>

// 使用例
interface FormProps {
  initialValues?: Partial<FullUser>
  onSubmit: (data: FullUser) => void
}
```typescript

---

## 3. イベントハンドラの型

### 基本的なイベント型

```typescript
// マウスイベント
function Button() {
  const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
    console.log('Button clicked at', e.clientX, e.clientY)
    e.currentTarget.disabled = true
  }

  return <button onClick={handleClick}>Click me</button>
}

// チェンジイベント（input）
function TextInput() {
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    console.log('Value:', e.target.value)
  }

  return <input onChange={handleChange} />
}

// フォームイベント
function Form() {
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault()
    const formData = new FormData(e.currentTarget)
    console.log('Form data:', Object.fromEntries(formData))
  }

  return <form onSubmit={handleSubmit}>{/* ... */}</form>
}

// キーボードイベント
function KeyboardInput() {
  const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
    if (e.key === 'Enter') {
      console.log('Enter pressed')
    }
    if (e.ctrlKey && e.key === 's') {
      e.preventDefault()
      console.log('Ctrl+S pressed')
    }
  }

  return <input onKeyDown={handleKeyDown} />
}
```typescript

### カスタムイベントハンドラの型

```typescript
// カスタムコンポーネントのイベントハンドラ
interface User {
  id: string
  name: string
}

interface UserListProps {
  users: User[]
  onUserSelect: (user: User) => void
  onUserDelete: (userId: string) => void
}

function UserList({ users, onUserSelect, onUserDelete }: UserListProps) {
  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>
          <button onClick={() => onUserSelect(user)}>
            {user.name}
          </button>
          <button onClick={() => onUserDelete(user.id)}>
            Delete
          </button>
        </li>
      ))}
    </ul>
  )
}
```typescript

---

## 4. ジェネリックコンポーネント

### ジェネリックなListコンポーネント

```typescript
interface ListProps<T> {
  items: T[]
  renderItem: (item: T, index: number) => React.ReactNode
  keyExtractor: (item: T) => string
  emptyMessage?: string
}

function List<T>({
  items,
  renderItem,
  keyExtractor,
  emptyMessage = 'No items'
}: ListProps<T>) {
  if (items.length === 0) {
    return <p>{emptyMessage}</p>
  }

  return (
    <ul>
      {items.map((item, index) => (
        <li key={keyExtractor(item)}>
          {renderItem(item, index)}
        </li>
      ))}
    </ul>
  )
}

// 使用例: User型
interface User {
  id: string
  name: string
  email: string
}

function UserList() {
  const users: User[] = [
    { id: '1', name: 'John', email: 'john@example.com' },
    { id: '2', name: 'Jane', email: 'jane@example.com' }
  ]

  return (
    <List<User>
      items={users}
      keyExtractor={(user) => user.id}
      renderItem={(user) => (
        <div>
          <strong>{user.name}</strong>
          <span>{user.email}</span>
        </div>
      )}
    />
  )
}
```typescript

### ジェネリックなSelectコンポーネント

```typescript
interface SelectProps<T> {
  value: T
  options: T[]
  onChange: (value: T) => void
  getLabel: (option: T) => string
  getValue: (option: T) => string
}

function Select<T>({
  value,
  options,
  onChange,
  getLabel,
  getValue
}: SelectProps<T>) {
  const handleChange = (e: React.ChangeEvent<HTMLSelectElement>) => {
    const selectedValue = e.target.value
    const selectedOption = options.find(
      (opt) => getValue(opt) === selectedValue
    )
    if (selectedOption) {
      onChange(selectedOption)
    }
  }

  return (
    <select value={getValue(value)} onChange={handleChange}>
      {options.map((option) => (
        <option key={getValue(option)} value={getValue(option)}>
          {getLabel(option)}
        </option>
      ))}
    </select>
  )
}

// 使用例: オブジェクト型
interface Country {
  code: string
  name: string
}

function CountrySelect() {
  const countries: Country[] = [
    { code: 'JP', name: 'Japan' },
    { code: 'US', name: 'United States' }
  ]

  const [selected, setSelected] = useState(countries[0])

  return (
    <Select<Country>
      value={selected}
      options={countries}
      onChange={setSelected}
      getLabel={(country) => country.name}
      getValue={(country) => country.code}
    />
  )
}
```typescript

---

## 5. 高度な型テクニック

### Conditional Types（条件付き型）

```typescript
// AsyncReturnType
type AsyncReturnType<T> = T extends (...args: any[]) => Promise<infer R>
  ? R
  : never

async function fetchUser() {
  return { id: '1', name: 'John' }
}

type User = AsyncReturnType<typeof fetchUser>
// { id: string; name: string }

// UnwrapArray
type UnwrapArray<T> = T extends Array<infer U> ? U : T

type StringType = UnwrapArray<string[]> // string
type NumberType = UnwrapArray<number> // number
```typescript

### Template Literal Types

```typescript
// イベント名の生成
type EventName = 'click' | 'focus' | 'blur'
type HandlerName = `on${Capitalize<EventName>}`
// 'onClick' | 'onFocus' | 'onBlur'

// 型安全なイベントハンドラ
type Event = 'submit' | 'cancel' | 'save'
type EventHandlers = {
  [K in Event as `on${Capitalize<K>}`]: () => void
}
// { onSubmit: () => void; onCancel: () => void; onSave: () => void }

interface FormProps extends EventHandlers {
  title: string
}

function Form({ title, onSubmit, onCancel, onSave }: FormProps) {
  return (
    <form>
      <h2>{title}</h2>
      <button onClick={onSubmit}>Submit</button>
      <button onClick={onCancel}>Cancel</button>
      <button onClick={onSave}>Save</button>
    </form>
  )
}
```typescript

### Type Guards（型ガード）

```typescript
// カスタム型ガード
interface User {
  type: 'user'
  id: string
  name: string
}

interface Admin {
  type: 'admin'
  id: string
  name: string
  permissions: string[]
}

function isAdmin(person: User | Admin): person is Admin {
  return person.type === 'admin'
}

function greet(person: User | Admin) {
  if (isAdmin(person)) {
    // この中では person は Admin型
    console.log(`Hello Admin ${person.name}`)
    console.log('Permissions:', person.permissions)
  } else {
    // この中では person は User型
    console.log(`Hello ${person.name}`)
  }
}
```typescript

---

## 6. よくある型エラーと解決策

### エラー1: "Type 'undefined' is not assignable to type 'X'"

```typescript
// ❌ 問題
const [user, setUser] = useState<User>() // undefinedが許可されていない

// ✅ 解決
const [user, setUser] = useState<User | undefined>()
// または
const [user, setUser] = useState<User | null>(null)
```typescript

### エラー2: "Property 'X' does not exist on type 'never'"

```typescript
// ❌ 問題
const [data, setData] = useState([])
data.push(item) // 型エラー

// ✅ 解決
const [data, setData] = useState<Item[]>([])
data.push(item) // OK
```typescript

### エラー3: "Object is possibly 'null'"

```typescript
// ❌ 問題
const inputRef = useRef<HTMLInputElement>(null)
inputRef.current.focus() // エラー: possibly 'null'

// ✅ 解決1: Optional Chaining
inputRef.current?.focus()

// ✅ 解決2: Null Check
if (inputRef.current) {
  inputRef.current.focus()
}
```typescript

---

## まとめ

この章では、TypeScriptとReactを組み合わせた実践的なパターンを学びました。

**重要ポイント**:
- ✅ React.FCよりも通常の関数を使用
- ✅ Discriminated Unionで条件付きPropsを型安全に
- ✅ ジェネリクスで再利用可能なコンポーネント
- ✅ Utility Types（Omit, Pick, Partial）を活用
- ✅ Type Guardsで型を絞り込む

**次章予告**: Chapter 4では、Reactアプリケーションのパフォーマンス最適化を学びます。

---

**参考リンク**:
- [TypeScript ハンドブック](https://www.typescriptlang.org/docs/handbook/intro.html)
- [React TypeScript Cheatsheet](https://react-typescript-cheatsheet.netlify.app/)
