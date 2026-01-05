---
title: "高度な型パターンとジェネリック"
---

# Chapter 5: 高度な型パターンとジェネリック

## この章で学べること

この章では、TypeScriptの高度な型機能を使ってReactコンポーネントの再利用性と型安全性を最大化する方法を学びます。

- ✅ ジェネリックコンポーネントの設計パターン
- ✅ Conditional Types（条件付き型）の実践活用
- ✅ Mapped Types（マップ型）で型を動的生成
- ✅ Template Literal Types で型安全なAPI
- ✅ Type Guards（型ガード）による型の絞り込み
- ✅ infer キーワードで型を抽出
- ✅ 実践例：汎用的なList/Table/Select/Formコンポーネント

**前提知識**: Chapter 4 の内容（基本的な型定義）

**所要時間**: 50-60分

---

## 目次

1. [ジェネリックコンポーネント入門](#1-ジェネリックコンポーネント入門)
2. [ジェネリックなListコンポーネント](#2-ジェネリックなlistコンポーネント)
3. [ジェネリックなSelectコンポーネント](#3-ジェネリックなselectコンポーネント)
4. [ジェネリックなTableコンポーネント](#4-ジェネリックなtableコンポーネント)
5. [ジェネリックなFormコンポーネント](#5-ジェネリックなformコンポーネント)
6. [Conditional Types（条件付き型）](#6-conditional-types条件付き型)
7. [Mapped Types（マップ型）](#7-mapped-typesマップ型)
8. [Template Literal Types](#8-template-literal-types)
9. [Type Guards（型ガード）](#9-type-guards型ガード)
10. [まとめ](#10-まとめ)

---

## 1. ジェネリックコンポーネント入門

### ジェネリックとは？

ジェネリック（Generics）は、型を引数として受け取る仕組みです。これにより、1つのコンポーネント定義で複数の型に対応できます。

```typescript
// ❌ 型ごとにコンポーネントを定義（冗長）
interface UserListProps {
  items: User[]
  renderItem: (item: User) => React.ReactNode
}

interface ProductListProps {
  items: Product[]
  renderItem: (item: Product) => React.ReactNode
}

// ✅ ジェネリックで1つの定義に統一
interface ListProps<T> {
  items: T[]
  renderItem: (item: T) => React.ReactNode
}

// User型でもProduct型でも使える
function UserList() {
  return <List<User> items={users} renderItem={renderUser} />
}

function ProductList() {
  return <List<Product> items={products} renderItem={renderProduct} />
}
```

### ジェネリックのメリット

1. **コードの再利用性**: 1つの実装で複数の型に対応
2. **型安全性**: 型の不整合をコンパイル時に検出
3. **保守性**: 重複コードを削減し、変更を1箇所に集約

---

## 2. ジェネリックなListコンポーネント

### 基本実装

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
    return <p className="empty-message">{emptyMessage}</p>
  }

  return (
    <ul className="list">
      {items.map((item, index) => (
        <li key={keyExtractor(item)}>
          {renderItem(item, index)}
        </li>
      ))}
    </ul>
  )
}
```

### 使用例1: User型

```typescript
interface User {
  id: string
  name: string
  email: string
  role: 'admin' | 'user'
}

function UserList() {
  const users: User[] = [
    { id: '1', name: 'John', email: 'john@example.com', role: 'admin' },
    { id: '2', name: 'Jane', email: 'jane@example.com', role: 'user' }
  ]

  return (
    <List<User>
      items={users}
      keyExtractor={(user) => user.id}
      renderItem={(user, index) => (
        <div className="user-item">
          <strong>
            {index + 1}. {user.name}
          </strong>
          <span className="email">{user.email}</span>
          <span className={`badge badge-${user.role}`}>
            {user.role}
          </span>
        </div>
      )}
      emptyMessage="No users found"
    />
  )
}
```

### 使用例2: Product型

```typescript
interface Product {
  id: string
  name: string
  price: number
  inStock: boolean
}

function ProductList() {
  const products: Product[] = [
    { id: '1', name: 'Apple', price: 100, inStock: true },
    { id: '2', name: 'Banana', price: 50, inStock: false }
  ]

  return (
    <List<Product>
      items={products}
      keyExtractor={(product) => product.id}
      renderItem={(product) => (
        <div className="product-item">
          <h3>{product.name}</h3>
          <p className="price">¥{product.price.toLocaleString()}</p>
          <span className={product.inStock ? 'in-stock' : 'out-of-stock'}>
            {product.inStock ? '在庫あり' : '在庫なし'}
          </span>
        </div>
      )}
      emptyMessage="No products available"
    />
  )
}
```

**型安全性の恩恵**:

```typescript
// ✅ OK: User型のプロパティにアクセス
<List<User>
  items={users}
  renderItem={(user) => <div>{user.name}</div>}
  keyExtractor={(user) => user.id}
/>

// ❌ 型エラー: Product型に'username'プロパティは存在しない
<List<Product>
  items={products}
  renderItem={(product) => <div>{product.name}</div>} // OK
  keyExtractor={(product) => product.username} // ❌ エラー
/>
```

---

## 3. ジェネリックなSelectコンポーネント

### 基本実装

```typescript
interface SelectProps<T> {
  value: T
  options: T[]
  onChange: (value: T) => void
  getLabel: (option: T) => string
  getValue: (option: T) => string
  placeholder?: string
  disabled?: boolean
}

function Select<T>({
  value,
  options,
  onChange,
  getLabel,
  getValue,
  placeholder,
  disabled = false
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
    <select
      value={getValue(value)}
      onChange={handleChange}
      disabled={disabled}
      className="select"
    >
      {placeholder && (
        <option value="" disabled>
          {placeholder}
        </option>
      )}
      {options.map((option) => (
        <option key={getValue(option)} value={getValue(option)}>
          {getLabel(option)}
        </option>
      ))}
    </select>
  )
}
```

### 使用例1: プリミティブ型

```typescript
function FruitSelect() {
  const [selected, setSelected] = useState('apple')
  const fruits = ['apple', 'banana', 'orange', 'grape']

  return (
    <Select<string>
      value={selected}
      options={fruits}
      onChange={setSelected}
      getLabel={(fruit) => fruit.charAt(0).toUpperCase() + fruit.slice(1)}
      getValue={(fruit) => fruit}
      placeholder="Select a fruit"
    />
  )
}
```

### 使用例2: オブジェクト型

```typescript
interface Country {
  code: string
  name: string
  flag: string
}

function CountrySelect() {
  const countries: Country[] = [
    { code: 'JP', name: 'Japan', flag: '🇯🇵' },
    { code: 'US', name: 'United States', flag: '🇺🇸' },
    { code: 'UK', name: 'United Kingdom', flag: '🇬🇧' }
  ]

  const [selected, setSelected] = useState(countries[0])

  return (
    <div>
      <Select<Country>
        value={selected}
        options={countries}
        onChange={setSelected}
        getLabel={(country) => `${country.flag} ${country.name}`}
        getValue={(country) => country.code}
      />
      <p>Selected: {selected.name} ({selected.code})</p>
    </div>
  )
}
```

---

## 4. ジェネリックなTableコンポーネント

### 基本実装

```typescript
interface Column<T> {
  key: string
  header: string
  render: (item: T) => React.ReactNode
  width?: string
  align?: 'left' | 'center' | 'right'
}

interface TableProps<T> {
  data: T[]
  columns: Column<T>[]
  keyExtractor: (item: T) => string
  onRowClick?: (item: T) => void
}

function Table<T>({
  data,
  columns,
  keyExtractor,
  onRowClick
}: TableProps<T>) {
  return (
    <table className="table">
      <thead>
        <tr>
          {columns.map((col) => (
            <th
              key={col.key}
              style={{
                width: col.width,
                textAlign: col.align ?? 'left'
              }}
            >
              {col.header}
            </th>
          ))}
        </tr>
      </thead>
      <tbody>
        {data.map((item) => (
          <tr
            key={keyExtractor(item)}
            onClick={() => onRowClick?.(item)}
            className={onRowClick ? 'clickable' : ''}
          >
            {columns.map((col) => (
              <td
                key={col.key}
                style={{ textAlign: col.align ?? 'left' }}
              >
                {col.render(item)}
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  )
}
```

### 使用例: Userテーブル

```typescript
interface User {
  id: string
  name: string
  email: string
  age: number
  status: 'active' | 'inactive'
}

function UserTable() {
  const users: User[] = [
    { id: '1', name: 'John', email: 'john@example.com', age: 25, status: 'active' },
    { id: '2', name: 'Jane', email: 'jane@example.com', age: 30, status: 'inactive' }
  ]

  const columns: Column<User>[] = [
    {
      key: 'name',
      header: 'Name',
      render: (user) => <strong>{user.name}</strong>,
      width: '200px'
    },
    {
      key: 'email',
      header: 'Email',
      render: (user) => (
        <a href={`mailto:${user.email}`}>{user.email}</a>
      )
    },
    {
      key: 'age',
      header: 'Age',
      render: (user) => `${user.age} years old`,
      align: 'center',
      width: '100px'
    },
    {
      key: 'status',
      header: 'Status',
      render: (user) => (
        <span className={`badge badge-${user.status}`}>
          {user.status}
        </span>
      ),
      align: 'center'
    },
    {
      key: 'actions',
      header: 'Actions',
      render: (user) => (
        <div className="actions">
          <button onClick={() => console.log('Edit', user.id)}>
            Edit
          </button>
          <button onClick={() => console.log('Delete', user.id)}>
            Delete
          </button>
        </div>
      ),
      align: 'right'
    }
  ]

  return (
    <Table<User>
      data={users}
      columns={columns}
      keyExtractor={(user) => user.id}
      onRowClick={(user) => console.log('Row clicked:', user.name)}
    />
  )
}
```

---

## 5. ジェネリックなFormコンポーネント

### 基本実装

```typescript
interface FormField<T> {
  name: keyof T
  label: string
  type: 'text' | 'number' | 'email' | 'password' | 'textarea'
  required?: boolean
  placeholder?: string
  validate?: (value: T[keyof T]) => string | undefined
}

interface FormProps<T> {
  initialValues: T
  fields: FormField<T>[]
  onSubmit: (values: T) => void
  submitLabel?: string
}

function Form<T extends Record<string, any>>({
  initialValues,
  fields,
  onSubmit,
  submitLabel = 'Submit'
}: FormProps<T>) {
  const [values, setValues] = useState<T>(initialValues)
  const [errors, setErrors] = useState<Partial<Record<keyof T, string>>>({})
  const [touched, setTouched] = useState<Partial<Record<keyof T, boolean>>>({})

  const handleChange = (name: keyof T, value: any) => {
    setValues((prev) => ({ ...prev, [name]: value }))

    // バリデーション
    const field = fields.find((f) => f.name === name)
    if (field?.validate) {
      const error = field.validate(value)
      setErrors((prev) => ({ ...prev, [name]: error }))
    }
  }

  const handleBlur = (name: keyof T) => {
    setTouched((prev) => ({ ...prev, [name]: true }))
  }

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault()

    // 全フィールドをバリデーション
    const newErrors: Partial<Record<keyof T, string>> = {}
    fields.forEach((field) => {
      if (field.validate) {
        const error = field.validate(values[field.name])
        if (error) {
          newErrors[field.name] = error
        }
      }
    })

    setErrors(newErrors)

    // エラーがなければsubmit
    if (Object.keys(newErrors).length === 0) {
      onSubmit(values)
    }
  }

  return (
    <form onSubmit={handleSubmit} className="form">
      {fields.map((field) => (
        <div key={String(field.name)} className="form-field">
          <label htmlFor={String(field.name)}>
            {field.label}
            {field.required && <span className="required">*</span>}
          </label>

          {field.type === 'textarea' ? (
            <textarea
              id={String(field.name)}
              value={String(values[field.name] ?? '')}
              onChange={(e) => handleChange(field.name, e.target.value)}
              onBlur={() => handleBlur(field.name)}
              placeholder={field.placeholder}
              required={field.required}
            />
          ) : (
            <input
              id={String(field.name)}
              type={field.type}
              value={String(values[field.name] ?? '')}
              onChange={(e) => {
                const value = field.type === 'number'
                  ? Number(e.target.value)
                  : e.target.value
                handleChange(field.name, value)
              }}
              onBlur={() => handleBlur(field.name)}
              placeholder={field.placeholder}
              required={field.required}
            />
          )}

          {touched[field.name] && errors[field.name] && (
            <span className="error-message">{errors[field.name]}</span>
          )}
        </div>
      ))}

      <button type="submit" className="submit-button">
        {submitLabel}
      </button>
    </form>
  )
}
```

### 使用例: 登録フォーム

```typescript
interface RegisterFormData {
  username: string
  email: string
  age: number
  bio: string
}

function RegisterForm() {
  const fields: FormField<RegisterFormData>[] = [
    {
      name: 'username',
      label: 'Username',
      type: 'text',
      required: true,
      placeholder: 'Enter your username',
      validate: (value) => {
        if (typeof value !== 'string') return 'Invalid username'
        if (value.length < 3) return 'Username must be at least 3 characters'
        if (value.length > 20) return 'Username must be at most 20 characters'
        return undefined
      }
    },
    {
      name: 'email',
      label: 'Email',
      type: 'email',
      required: true,
      placeholder: 'your@email.com',
      validate: (value) => {
        if (typeof value !== 'string') return 'Invalid email'
        if (!value.includes('@')) return 'Invalid email format'
        return undefined
      }
    },
    {
      name: 'age',
      label: 'Age',
      type: 'number',
      required: true,
      validate: (value) => {
        if (typeof value !== 'number') return 'Invalid age'
        if (value < 18) return 'Must be 18 or older'
        if (value > 120) return 'Invalid age'
        return undefined
      }
    },
    {
      name: 'bio',
      label: 'Bio',
      type: 'textarea',
      placeholder: 'Tell us about yourself...',
      validate: (value) => {
        if (typeof value !== 'string') return 'Invalid bio'
        if (value.length > 500) return 'Bio must be at most 500 characters'
        return undefined
      }
    }
  ]

  const handleSubmit = (values: RegisterFormData) => {
    console.log('Form submitted:', values)
    // API call...
  }

  return (
    <Form<RegisterFormData>
      initialValues={{ username: '', email: '', age: 0, bio: '' }}
      fields={fields}
      onSubmit={handleSubmit}
      submitLabel="Register"
    />
  )
}
```

---

## 6. Conditional Types（条件付き型）

### 基本構文

```typescript
type IsString<T> = T extends string ? true : false

type A = IsString<string> // true型
type B = IsString<number> // false型
```

### 実用例1: AsyncReturnType

```typescript
// 非同期関数の戻り値の型を抽出
type AsyncReturnType<T> = T extends (...args: any[]) => Promise<infer R>
  ? R
  : never

async function fetchUser() {
  return { id: '1', name: 'John', email: 'john@example.com' }
}

type User = AsyncReturnType<typeof fetchUser>
// { id: string; name: string; email: string }

// 使用例
function UserComponent() {
  const [user, setUser] = useState<User | null>(null)

  useEffect(() => {
    fetchUser().then(setUser)
  }, [])

  return user ? <div>{user.name}</div> : null
}
```

### 実用例2: UnwrapArray

```typescript
// 配列型から要素の型を抽出
type UnwrapArray<T> = T extends Array<infer U> ? U : T

type StringArray = UnwrapArray<string[]> // string
type Number = UnwrapArray<number> // number

// 使用例：配列のアイテムの型を推論
function useArrayItem<T extends any[]>(
  array: T,
  index: number
): UnwrapArray<T> | undefined {
  return array[index]
}

const users = [
  { id: '1', name: 'John' },
  { id: '2', name: 'Jane' }
]

const user = useArrayItem(users, 0)
// user の型: { id: string; name: string } | undefined
```

### 実用例3: NonNullable

```typescript
// null と undefined を除外
type NonNullable<T> = T extends null | undefined ? never : T

type MaybeString = string | null | undefined
type DefiniteString = NonNullable<MaybeString> // string

// 使用例
interface User {
  id: string
  name: string
  email: string | null
}

type RequiredEmail = NonNullable<User['email']> // string
```

---

## 7. Mapped Types（マップ型）

### 基本パターン

```typescript
// 全てのプロパティをオプショナルに
type Optional<T> = {
  [K in keyof T]?: T[K]
}

interface User {
  id: string
  name: string
  email: string
}

type OptionalUser = Optional<User>
// { id?: string; name?: string; email?: string }
```

### 実用例1: Readonly の深いバージョン

```typescript
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? DeepReadonly<T[K]>
    : T[K]
}

interface NestedData {
  user: {
    name: string
    address: {
      city: string
      country: string
    }
  }
}

type ImmutableData = DeepReadonly<NestedData>

const data: ImmutableData = {
  user: {
    name: 'John',
    address: { city: 'Tokyo', country: 'Japan' }
  }
}

// ❌ 全て読み取り専用
data.user.name = 'Jane' // エラー
data.user.address.city = 'Osaka' // エラー
```

### 実用例2: Nullable

```typescript
// 全てのプロパティを nullable に
type Nullable<T> = {
  [K in keyof T]: T[K] | null
}

interface User {
  id: string
  name: string
  email: string
}

type NullableUser = Nullable<User>
// { id: string | null; name: string | null; email: string | null }

// 使用例：API レスポンス
function parseUserResponse(response: NullableUser): User | null {
  if (!response.id || !response.name || !response.email) {
    return null
  }
  return {
    id: response.id,
    name: response.name,
    email: response.email
  }
}
```

---

## 8. Template Literal Types

### イベントハンドラの自動生成

```typescript
type EventName = 'click' | 'focus' | 'blur' | 'submit'
type HandlerName = `on${Capitalize<EventName>}`
// 'onClick' | 'onFocus' | 'onBlur' | 'onSubmit'

// 実用例
type Event = 'submit' | 'cancel' | 'save' | 'delete'
type EventHandlers = {
  [K in Event as `on${Capitalize<K>}`]: () => void
}
// {
//   onSubmit: () => void
//   onCancel: () => void
//   onSave: () => void
//   onDelete: () => void
// }

interface FormProps extends EventHandlers {
  title: string
}

function Form({ title, onSubmit, onCancel, onSave, onDelete }: FormProps) {
  return (
    <form>
      <h2>{title}</h2>
      <button type="button" onClick={onSubmit}>Submit</button>
      <button type="button" onClick={onCancel}>Cancel</button>
      <button type="button" onClick={onSave}>Save</button>
      <button type="button" onClick={onDelete}>Delete</button>
    </form>
  )
}
```

### CSS プロパティの生成

```typescript
type CSSProperty = 'margin' | 'padding'
type CSSDirection = 'top' | 'right' | 'bottom' | 'left'
type CSSPropertyWithDirection = `${CSSProperty}${Capitalize<CSSDirection>}`
// 'marginTop' | 'marginRight' | 'marginBottom' | 'marginLeft' |
// 'paddingTop' | 'paddingRight' | 'paddingBottom' | 'paddingLeft'

// 使用例
type SpacingProps = {
  [K in CSSPropertyWithDirection]?: number
}

interface BoxProps extends SpacingProps {
  children: React.ReactNode
}

function Box({ children, ...spacing }: BoxProps) {
  return (
    <div style={spacing}>
      {children}
    </div>
  )
}

// 使用例
<Box marginTop={10} paddingLeft={20}>
  Content
</Box>
```

---

## 9. Type Guards（型ガード）

### typeof を使った型ガード

```typescript
function processValue(value: string | number) {
  if (typeof value === 'string') {
    // この中では value は string型
    return value.toUpperCase()
  }
  // この中では value は number型
  return value.toFixed(2)
}
```

### カスタム型ガード

```typescript
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

// カスタム型ガード関数
function isAdmin(person: User | Admin): person is Admin {
  return person.type === 'admin'
}

function UserProfile({ person }: { person: User | Admin }) {
  if (isAdmin(person)) {
    // この中では person は Admin型
    return (
      <div>
        <h2>Admin: {person.name}</h2>
        <ul>
          {person.permissions.map((p) => (
            <li key={p}>{p}</li>
          ))}
        </ul>
      </div>
    )
  }

  // この中では person は User型
  return <div>User: {person.name}</div>
}
```

### null/undefined チェック

```typescript
function processUser(user: User | null | undefined) {
  if (!user) {
    return <div>No user</div>
  }

  // この中では user は User型（null/undefinedではない）
  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  )
}
```

---

## 10. まとめ

この章では、TypeScriptの高度な型機能を使ったReactコンポーネント設計を学びました。

### 重要ポイント

1. **ジェネリックコンポーネント**: 型引数で再利用性を最大化
   - List/Select/Table/Form など汎用コンポーネントに最適

2. **Conditional Types**: `T extends U ? X : Y` で型を条件分岐
   - AsyncReturnType, UnwrapArray など型の変換に活用

3. **Mapped Types**: `[K in keyof T]` で型を動的生成
   - DeepReadonly, Nullable など型の変換に活用

4. **Template Literal Types**: 文字列リテラルで型を構築
   - イベントハンドラ、CSSプロパティなど命名規則のある型に最適

5. **Type Guards**: `is` キーワードで型を絞り込み
   - カスタム型ガード関数で複雑な型判定も型安全に

### 実践での使い所

- **List/Table**: データ表示コンポーネントは必ずジェネリックに
- **Form**: フォームの型定義は `keyof T` で型安全に
- **API レスポンス**: Conditional Types で戻り値の型を自動推論
- **イベントハンドラ**: Template Literal Types で命名を統一

### Next Steps

次の章では、これらの型パターンを活用して Context と Form を型安全に実装します。

- Chapter 6: Context と Form の型安全な実装

---

**執筆時間**: 約55分で習得可能
**文字数**: 約2,200語

この章をマスターすることで、TypeScriptの高度な型機能を駆使した再利用性の高いコンポーネント設計ができるようになります。
