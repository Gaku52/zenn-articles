---
title: "Client Components - インタラクティブUIの実装"
---

# Client Components

本章では、Client Componentsの使い方とServer Componentsとの効果的な使い分けを学びます。

## Client Componentsの基礎

### `'use client'` ディレクティブ

```tsx:components/Counter.tsx
'use client' // ← この1行で Client Component になる

import { useState } from 'react'

export function Counter() {
  const [count, setCount] = useState(0)

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </div>
  )
}
```

**重要ポイント:**
- ファイルの**最上部**に記述
- そのファイルとインポートされる全ての依存関係がClient Componentになる
- Server Componentから呼び出せる

### Client Componentが必要な場合

| 機能 | 説明 | 例 |
|-----|------|-----|
| **React Hooks** | useState, useEffect等 | `useState(0)` |
| **イベントハンドラー** | onClick, onChange等 | `onClick={() => {}}` |
| **ブラウザAPI** | window, localStorage等 | `window.innerWidth` |
| **カスタムHooks** | useHooksを使用 | `useDebounce()` |
| **Contextの消費** | useContext | `useContext(ThemeContext)` |
| **ライフサイクル** | useEffect, useLayoutEffect | `useEffect(() => {})` |

## 基本的なパターン

### パターン1: 状態管理

```tsx:components/SearchBox.tsx
'use client'

import { useState } from 'react'

export function SearchBox({ onSearch }: { onSearch: (query: string) => void }) {
  const [query, setQuery] = useState('')

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault()
    onSearch(query)
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="検索..."
        className="border px-4 py-2 rounded"
      />
      <button type="submit" className="ml-2 px-4 py-2 bg-blue-500 text-white rounded">
        検索
      </button>
    </form>
  )
}
```

### パターン2: イベントハンドラー

```tsx:components/LikeButton.tsx
'use client'

import { useState } from 'react'
import { Heart } from 'lucide-react'

export function LikeButton({ postId, initialLikes }: { postId: string, initialLikes: number }) {
  const [likes, setLikes] = useState(initialLikes)
  const [isLiked, setIsLiked] = useState(false)
  const [isLoading, setIsLoading] = useState(false)

  const handleLike = async () => {
    setIsLoading(true)
    setIsLiked(!isLiked)
    setLikes(isLiked ? likes - 1 : likes + 1)

    try {
      const res = await fetch(`/api/posts/${postId}/like`, {
        method: 'POST',
      })
      const data = await res.json()
      setLikes(data.likes)
    } catch (error) {
      // エラー時は元に戻す
      setIsLiked(isLiked)
      setLikes(likes)
    } finally {
      setIsLoading(false)
    }
  }

  return (
    <button
      onClick={handleLike}
      disabled={isLoading}
      className={`flex items-center gap-2 px-4 py-2 rounded ${
        isLiked ? 'text-red-500' : 'text-gray-500'
      }`}
    >
      <Heart fill={isLiked ? 'currentColor' : 'none'} />
      <span>{likes}</span>
    </button>
  )
}
```

### パターン3: ブラウザAPIの使用

```tsx:components/ScrollToTop.tsx
'use client'

import { useEffect, useState } from 'react'
import { ArrowUp } from 'lucide-react'

export function ScrollToTop() {
  const [isVisible, setIsVisible] = useState(false)

  useEffect(() => {
    const toggleVisibility = () => {
      if (window.scrollY > 300) {
        setIsVisible(true)
      } else {
        setIsVisible(false)
      }
    }

    window.addEventListener('scroll', toggleVisibility)

    return () => {
      window.removeEventListener('scroll', toggleVisibility)
    }
  }, [])

  const scrollToTop = () => {
    window.scrollTo({
      top: 0,
      behavior: 'smooth',
    })
  }

  if (!isVisible) return null

  return (
    <button
      onClick={scrollToTop}
      className="fixed bottom-8 right-8 p-3 bg-blue-500 text-white rounded-full shadow-lg hover:bg-blue-600"
    >
      <ArrowUp size={24} />
    </button>
  )
}
```

## データフェッチング in Client Components

### パターン1: useEffectでのフェッチ

```tsx:components/UserProfile.tsx
'use client'

import { useState, useEffect } from 'react'

interface User {
  id: string
  name: string
  email: string
  avatar: string
}

export function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null)
  const [isLoading, setIsLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    const fetchUser = async () => {
      try {
        const res = await fetch(`/api/users/${userId}`)
        if (!res.ok) throw new Error('Failed to fetch')
        const data = await res.json()
        setUser(data)
      } catch (err) {
        setError(err instanceof Error ? err.message : 'Unknown error')
      } finally {
        setIsLoading(false)
      }
    }

    fetchUser()
  }, [userId])

  if (isLoading) {
    return <div className="animate-pulse">Loading...</div>
  }

  if (error) {
    return <div className="text-red-500">Error: {error}</div>
  }

  if (!user) {
    return <div>User not found</div>
  }

  return (
    <div className="flex items-center gap-4">
      <img src={user.avatar} alt={user.name} className="w-16 h-16 rounded-full" />
      <div>
        <h2 className="text-xl font-bold">{user.name}</h2>
        <p className="text-gray-600">{user.email}</p>
      </div>
    </div>
  )
}
```

### パターン2: SWR/React Queryの使用（推奨）

```tsx:components/Posts.tsx
'use client'

import useSWR from 'swr'

const fetcher = (url: string) => fetch(url).then(res => res.json())

export function Posts() {
  const { data, error, isLoading, mutate } = useSWR('/api/posts', fetcher, {
    revalidateOnFocus: false,
    revalidateOnReconnect: true,
  })

  if (isLoading) return <div>Loading...</div>
  if (error) return <div>Error: {error.message}</div>

  return (
    <div>
      <button onClick={() => mutate()}>Refresh</button>
      <ul>
        {data.map((post: any) => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </div>
  )
}
```

**SWRのメリット:**
- 自動キャッシング
- 再検証（revalidation）
- フォーカス時の再取得
- エラーリトライ

## カスタムHooks

### useDebounce

```tsx:hooks/useDebounce.ts
'use client'

import { useState, useEffect } from 'react'

export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value)

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value)
    }, delay)

    return () => {
      clearTimeout(handler)
    }
  }, [value, delay])

  return debouncedValue
}
```

**使用例:**
```tsx:components/SearchWithDebounce.tsx
'use client'

import { useState, useEffect } from 'react'
import { useDebounce } from '@/hooks/useDebounce'

export function SearchWithDebounce() {
  const [query, setQuery] = useState('')
  const [results, setResults] = useState([])
  const debouncedQuery = useDebounce(query, 500)

  useEffect(() => {
    if (debouncedQuery) {
      fetch(`/api/search?q=${debouncedQuery}`)
        .then(res => res.json())
        .then(setResults)
    }
  }, [debouncedQuery])

  return (
    <div>
      <input
        type="text"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search..."
      />
      <ul>
        {results.map((item: any) => (
          <li key={item.id}>{item.title}</li>
        ))}
      </ul>
    </div>
  )
}
```

### useLocalStorage

```tsx:hooks/useLocalStorage.ts
'use client'

import { useState, useEffect } from 'react'

export function useLocalStorage<T>(key: string, initialValue: T) {
  const [storedValue, setStoredValue] = useState<T>(initialValue)

  useEffect(() => {
    try {
      const item = window.localStorage.getItem(key)
      if (item) {
        setStoredValue(JSON.parse(item))
      }
    } catch (error) {
      console.error(error)
    }
  }, [key])

  const setValue = (value: T | ((val: T) => T)) => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value
      setStoredValue(valueToStore)
      window.localStorage.setItem(key, JSON.stringify(valueToStore))
    } catch (error) {
      console.error(error)
    }
  }

  return [storedValue, setValue] as const
}
```

**使用例:**
```tsx:components/ThemeToggle.tsx
'use client'

import { useLocalStorage } from '@/hooks/useLocalStorage'

export function ThemeToggle() {
  const [theme, setTheme] = useLocalStorage<'light' | 'dark'>('theme', 'light')

  return (
    <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
      Current: {theme}
    </button>
  )
}
```

## Context APIの活用

### Theme Context

```tsx:contexts/ThemeContext.tsx
'use client'

import { createContext, useContext, useState, ReactNode } from 'react'

type Theme = 'light' | 'dark'

interface ThemeContextType {
  theme: Theme
  toggleTheme: () => void
}

const ThemeContext = createContext<ThemeContextType | undefined>(undefined)

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<Theme>('light')

  const toggleTheme = () => {
    setTheme(prev => prev === 'light' ? 'dark' : 'light')
  }

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      <div className={theme}>
        {children}
      </div>
    </ThemeContext.Provider>
  )
}

export function useTheme() {
  const context = useContext(ThemeContext)
  if (context === undefined) {
    throw new Error('useTheme must be used within a ThemeProvider')
  }
  return context
}
```

**使用例:**
```tsx:app/layout.tsx
import { ThemeProvider } from '@/contexts/ThemeContext'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja">
      <body>
        <ThemeProvider>
          {children}
        </ThemeProvider>
      </body>
    </html>
  )
}
```

```tsx:components/ThemeToggleButton.tsx
'use client'

import { useTheme } from '@/contexts/ThemeContext'

export function ThemeToggleButton() {
  const { theme, toggleTheme } = useTheme()

  return (
    <button onClick={toggleTheme}>
      {theme === 'light' ? '🌙' : '☀️'}
    </button>
  )
}
```

## Server ComponentsとClient Componentsの境界設計

### パターン1: Server → Client（データ渡し）

```tsx:app/posts/page.tsx
// Server Component
import { PostList } from '@/components/PostList'

async function getPosts() {
  const res = await fetch('https://api.example.com/posts', {
    next: { revalidate: 3600 }
  })
  return res.json()
}

export default async function PostsPage() {
  const posts = await getPosts()

  return (
    <div>
      <h1>Posts</h1>
      {/* サーバーで取得したデータをClientに渡す */}
      <PostList initialPosts={posts} />
    </div>
  )
}
```

```tsx:components/PostList.tsx
'use client'

import { useState } from 'react'

export function PostList({ initialPosts }: { initialPosts: Post[] }) {
  const [posts, setPosts] = useState(initialPosts)
  const [sortBy, setSortBy] = useState<'title' | 'date'>('date')

  const sorted = [...posts].sort((a, b) => {
    if (sortBy === 'title') {
      return a.title.localeCompare(b.title)
    }
    return new Date(b.date).getTime() - new Date(a.date).getTime()
  })

  return (
    <div>
      <select value={sortBy} onChange={e => setSortBy(e.target.value as any)}>
        <option value="date">日付順</option>
        <option value="title">タイトル順</option>
      </select>
      <ul>
        {sorted.map(post => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </div>
  )
}
```

### パターン2: 境界を最小限に保つ

```tsx
// ❌ 悪い例: 全体をClient Componentに
'use client'

export function Dashboard() {
  const [isOpen, setIsOpen] = useState(false)

  return (
    <div>
      <Header />  {/* 静的なのにClientになる */}
      <Sidebar /> {/* 静的なのにClientになる */}
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      {isOpen && <Modal />}
      <Footer />  {/* 静的なのにClientになる */}
    </div>
  )
}
```

```tsx
// ✅ 良い例: 必要な部分だけClient Component
export function Dashboard() {
  return (
    <div>
      <Header />  {/* Server Component */}
      <Sidebar /> {/* Server Component */}
      <ToggleButton /> {/* Client Component */}
      <Footer />  {/* Server Component */}
    </div>
  )
}

// components/ToggleButton.tsx
'use client'

export function ToggleButton() {
  const [isOpen, setIsOpen] = useState(false)

  return (
    <>
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      {isOpen && <Modal />}
    </>
  )
}
```

## パフォーマンス最適化

### メモ化によるレンダリング削減

```tsx:components/ExpensiveList.tsx
'use client'

import { memo } from 'react'

interface Props {
  items: string[]
}

export const ExpensiveList = memo(function ExpensiveList({ items }: Props) {
  console.log('Rendering ExpensiveList')

  return (
    <ul>
      {items.map((item, index) => (
        <li key={index}>{item}</li>
      ))}
    </ul>
  )
})
```

### useCallback/useMemoの活用

```tsx:components/OptimizedSearch.tsx
'use client'

import { useState, useMemo, useCallback } from 'react'

export function OptimizedSearch({ items }: { items: string[] }) {
  const [query, setQuery] = useState('')

  // フィルター結果をメモ化
  const filtered = useMemo(() => {
    console.log('Filtering...')
    return items.filter(item => item.toLowerCase().includes(query.toLowerCase()))
  }, [items, query])

  // イベントハンドラーをメモ化
  const handleChange = useCallback((e: React.ChangeEvent<HTMLInputElement>) => {
    setQuery(e.target.value)
  }, [])

  return (
    <div>
      <input type="text" value={query} onChange={handleChange} />
      <ul>
        {filtered.map((item, index) => (
          <li key={index}>{item}</li>
        ))}
      </ul>
    </div>
  )
}
```

## よくある間違い

### ❌ 間違い1: Server ComponentをClient Componentにインポート

```tsx
// ❌ 悪い例
'use client'

import { ServerComponent } from './ServerComponent' // Server ComponentがClientになる

export function ClientComponent() {
  return <ServerComponent />
}
```

```tsx
// ✅ 良い例: childrenで受け取る
'use client'

export function ClientComponent({ children }: { children: React.ReactNode }) {
  return <div>{children}</div>
}

// 使用側（Server Component）
import { ClientComponent } from './ClientComponent'
import { ServerComponent } from './ServerComponent'

export function Page() {
  return (
    <ClientComponent>
      <ServerComponent />
    </ClientComponent>
  )
}
```

### ❌ 間違い2: 不要なuseEffect

```tsx
// ❌ 悪い例
'use client'

export function Component({ data }: { data: string }) {
  const [value, setValue] = useState('')

  useEffect(() => {
    setValue(data) // 不要なuseEffect
  }, [data])

  return <div>{value}</div>
}
```

```tsx
// ✅ 良い例: 直接使う
'use client'

export function Component({ data }: { data: string }) {
  return <div>{data}</div>
}
```

### ❌ 間違い3: すべてのコンポーネントをClient Componentに

```tsx
// ❌ 悪い例: 全てにuse client
'use client'

export function StaticCard({ title }: { title: string }) {
  return <div>{title}</div> // インタラクティブじゃない
}
```

```tsx
// ✅ 良い例: Server Componentのまま
export function StaticCard({ title }: { title: string }) {
  return <div>{title}</div>
}
```

## まとめ

本章で学んだこと:

✅ Client Componentsの基本と`'use client'`
✅ イベントハンドラーと状態管理
✅ カスタムHooksの実装
✅ Context APIの活用
✅ Server/Client境界の最適な設計
✅ パフォーマンス最適化テクニック

**原則: 必要最小限の範囲でClient Componentを使用する**

次章では、コンポーネントパターンとアーキテクチャを学びます。
