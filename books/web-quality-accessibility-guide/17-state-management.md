---
title: "状態管理 - 効率的なデータ管理"
---

# 状態管理 - 効率的なデータ管理

## 状態管理の種類

Reactアプリケーションの状態は、以下の4つに分類されます：

1. **ローカル状態**: コンポーネント内の状態（useState）
2. **グローバル状態**: アプリ全体で共有する状態
3. **サーバー状態**: サーバーから取得するデータ
4. **URL状態**: URLパラメータ、クエリ文字列

## ローカル状態

### useState

```tsx
'use client'

import { useState } from 'react'

export function Counter() {
  const [count, setCount] = useState(0)

  return (
    <div>
      <p>カウント: {count}</p>
      <button onClick={() => setCount(count + 1)}>増やす</button>
    </div>
  )
}
```

### useReducer

複雑な状態管理に適しています。

```tsx
'use client'

import { useReducer } from 'react'

type State = {
  count: number
  step: number
}

type Action =
  | { type: 'INCREMENT' }
  | { type: 'DECREMENT' }
  | { type: 'SET_STEP', payload: number }

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'INCREMENT':
      return { ...state, count: state.count + state.step }
    case 'DECREMENT':
      return { ...state, count: state.count - state.step }
    case 'SET_STEP':
      return { ...state, step: action.payload }
    default:
      return state
  }
}

export function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0, step: 1 })

  return (
    <div>
      <p>カウント: {state.count}</p>
      <button onClick={() => dispatch({ type: 'INCREMENT' })}>+{state.step}</button>
      <button onClick={() => dispatch({ type: 'DECREMENT' })}>-{state.step}</button>
      <input
        type="number"
        value={state.step}
        onChange={(e) => dispatch({ type: 'SET_STEP', payload: Number(e.target.value) })}
      />
    </div>
  )
}
```

## グローバル状態

### Context API

小〜中規模のアプリケーションに適しています。

```tsx
// contexts/ThemeContext.tsx
'use client'

import { createContext, useContext, useState, ReactNode } from 'react'

type Theme = 'light' | 'dark'

type ThemeContextType = {
  theme: Theme
  setTheme: (theme: Theme) => void
}

const ThemeContext = createContext<ThemeContextType | null>(null)

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<Theme>('light')

  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  )
}

export function useTheme() {
  const context = useContext(ThemeContext)
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider')
  }
  return context
}

// 使用例
function ThemeToggle() {
  const { theme, setTheme } = useTheme()

  return (
    <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
      {theme === 'light' ? '🌙' : '☀️'}
    </button>
  )
}
```

### Zustand

シンプルで高速な状態管理ライブラリです。

```bash
pnpm add zustand
```

```typescript
// store/userStore.ts
import { create } from 'zustand'

interface User {
  id: string
  name: string
  email: string
}

interface UserStore {
  user: User | null
  setUser: (user: User) => void
  logout: () => void
}

export const useUserStore = create<UserStore>((set) => ({
  user: null,
  setUser: (user) => set({ user }),
  logout: () => set({ user: null }),
}))

// 使用例
'use client'

import { useUserStore } from '@/store/userStore'

export function UserProfile() {
  const { user, logout } = useUserStore()

  if (!user) return <div>ログインしてください</div>

  return (
    <div>
      <p>{user.name}</p>
      <p>{user.email}</p>
      <button onClick={logout}>ログアウト</button>
    </div>
  )
}
```

### Zustandの高度な使い方

#### セレクタ（再レンダリング最適化）

```tsx
// ✅ 良い例: セレクタで必要な部分だけ購読
const userName = useUserStore((state) => state.user?.name)

// ❌ 悪い例: ストア全体を購読
const { user } = useUserStore()
const userName = user?.name
```

#### 永続化

```typescript
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

export const useUserStore = create<UserStore>()(
  persist(
    (set) => ({
      user: null,
      setUser: (user) => set({ user }),
      logout: () => set({ user: null }),
    }),
    {
      name: 'user-storage', // localStorageのキー
    }
  )
)
```

## サーバー状態

サーバーから取得するデータは、TanStack Query（旧React Query）を使用することを推奨します。

```bash
pnpm add @tanstack/react-query
```

```tsx
// app/providers.tsx
'use client'

import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { useState } from 'react'

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient())

  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}

// app/layout.tsx
import { Providers } from './providers'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  )
}
```

```tsx
// hooks/useUser.ts
import { useQuery } from '@tanstack/react-query'

async function fetchUser(userId: string) {
  const res = await fetch(`/api/users/${userId}`)
  if (!res.ok) throw new Error('Failed to fetch user')
  return res.json()
}

export function useUser(userId: string) {
  return useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  })
}

// 使用例
'use client'

import { useUser } from '@/hooks/useUser'

export function UserProfile({ userId }: { userId: string }) {
  const { data: user, isLoading, error } = useUser(userId)

  if (isLoading) return <div>読み込み中...</div>
  if (error) return <div>エラー: {error.message}</div>
  if (!user) return null

  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  )
}
```

## 状態管理の選択

| 状態の種類 | 推奨ライブラリ |
|----------|--------------|
| ローカル状態 | useState、useReducer |
| グローバル状態（小規模）| Context API |
| グローバル状態（中〜大規模）| Zustand |
| サーバー状態 | TanStack Query |
| フォーム状態 | React Hook Form |

## 実践例: ショッピングカート

```typescript
// store/cartStore.ts
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface CartItem {
  id: string
  name: string
  price: number
  quantity: number
}

interface CartStore {
  items: CartItem[]
  addItem: (item: CartItem) => void
  removeItem: (id: string) => void
  updateQuantity: (id: string, quantity: number) => void
  clearCart: () => void
  totalPrice: () => number
}

export const useCartStore = create<CartStore>()(
  persist(
    (set, get) => ({
      items: [],
      addItem: (item) => {
        const { items } = get()
        const existingItem = items.find(i => i.id === item.id)

        if (existingItem) {
          set({
            items: items.map(i =>
              i.id === item.id
                ? { ...i, quantity: i.quantity + item.quantity }
                : i
            ),
          })
        } else {
          set({ items: [...items, item] })
        }
      },
      removeItem: (id) => {
        set({ items: get().items.filter(i => i.id !== id) })
      },
      updateQuantity: (id, quantity) => {
        if (quantity <= 0) {
          get().removeItem(id)
        } else {
          set({
            items: get().items.map(i =>
              i.id === id ? { ...i, quantity } : i
            ),
          })
        }
      },
      clearCart: () => set({ items: [] }),
      totalPrice: () => {
        return get().items.reduce((total, item) => total + item.price * item.quantity, 0)
      },
    }),
    {
      name: 'cart-storage',
    }
  )
)

// 使用例
'use client'

import { useCartStore } from '@/store/cartStore'

export function Cart() {
  const { items, removeItem, updateQuantity, totalPrice } = useCartStore()

  return (
    <div>
      <h2>カート</h2>
      {items.length === 0 ? (
        <p>カートは空です</p>
      ) : (
        <>
          <ul>
            {items.map(item => (
              <li key={item.id}>
                <span>{item.name}</span>
                <input
                  type="number"
                  value={item.quantity}
                  onChange={(e) => updateQuantity(item.id, Number(e.target.value))}
                  min="0"
                />
                <span>¥{item.price * item.quantity}</span>
                <button onClick={() => removeItem(item.id)}>削除</button>
              </li>
            ))}
          </ul>
          <p>合計: ¥{totalPrice()}</p>
        </>
      )}
    </div>
  )
}
```

## パフォーマンス最適化

### 状態の分割

```tsx
// ✅ 良い例: 状態を分割
const useFormStore = create((set) => ({
  name: '',
  setName: (name: string) => set({ name }),
}))

const useSubmitStore = create((set) => ({
  isSubmitting: false,
  setSubmitting: (isSubmitting: boolean) => set({ isSubmitting }),
}))

// ❌ 悪い例: 全てを1つのストアに
const useFormStore = create((set) => ({
  name: '',
  email: '',
  message: '',
  isSubmitting: false,
  errors: {},
  // ... 多すぎる
}))
```

## まとめ

状態管理は、アプリケーションの規模と要件に応じて適切な手法を選択します。

### チェックリスト

- ローカル状態はuseStateまたはuseReducer
- グローバル状態はZustand（小規模ならContext API）
- サーバー状態はTanStack Query
- 永続化が必要な場合はpersistミドルウェア
- セレクタで再レンダリングを最適化

## 次のステップ

Part 4では、技術ドキュメント作成について解説します。
