---
title: "プロジェクトアーキテクチャ設計"
---

# Chapter 8: プロジェクトアーキテクチャ設計

> 保守しやすく、スケーラブルなプロジェクト構成

## この章で学べること

この章では、実践的なReactプロジェクトのアーキテクチャ設計を学びます。

- ✅ チーム規模別のディレクトリ構成
- ✅ 状態管理の設計パターン
- ✅ コンポーネント設計の原則
- ✅ ファイル命名規則とモジュール分割

**前提知識**: Chapter 01-07の内容

**所要時間**: 60-80分

**難易度**: ★★★★☆（中上級）

---

## 目次

1. [ディレクトリ構成](#1-ディレクトリ構成)
   - 小規模プロジェクト（1-3人）
   - 中規模プロジェクト（5-15人）
   - 大規模プロジェクト（20人以上）
2. [状態管理アーキテクチャ](#2-状態管理アーキテクチャ)
   - Server State vs Client State
   - 状態管理ライブラリの選択
3. [コンポーネント設計原則](#3-コンポーネント設計原則)
   - コンポーネント分割の基準
   - Props Drillingの回避
4. [ファイル命名規則](#4-ファイル命名規則)
   - コンポーネント
   - hooks / utils / types
5. [環境変数管理](#5-環境変数管理)
6. [実践例](#6-実践例)

---

## 1. ディレクトリ構成

### 小規模プロジェクト（1-3人）

```
project/
├── app/                      # Next.js App Router
│   ├── page.tsx              # / (トップページ)
│   ├── about/page.tsx        # /about
│   ├── blog/
│   │   ├── page.tsx          # /blog
│   │   └── [slug]/page.tsx   # /blog/[slug]
│   ├── api/
│   │   └── posts/route.ts    # /api/posts
│   ├── layout.tsx
│   └── globals.css
│
├── components/               # 共有コンポーネント
│   ├── ui/                   # 汎用UI
│   │   ├── button.tsx
│   │   ├── card.tsx
│   │   └── input.tsx
│   └── header.tsx
│
├── lib/                      # ユーティリティ
│   ├── utils.ts
│   └── api.ts
│
├── public/                   # 静的ファイル
│   └── images/
│
├── .env.local
├── next.config.js
├── tailwind.config.ts
└── package.json
```

**特徴**:
- シンプル・フラット
- ファイル数少ない（〜50ファイル）
- 全員が全体を把握可能

### 中規模プロジェクト（5-15人）

```
project/
├── app/
│   ├── (marketing)/          # ルートグループ
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   ├── about/page.tsx
│   │   └── pricing/page.tsx
│   ├── (dashboard)/          # ログイン後
│   │   ├── layout.tsx
│   │   ├── dashboard/page.tsx
│   │   ├── settings/page.tsx
│   │   └── profile/page.tsx
│   ├── api/
│   │   ├── auth/[...nextauth]/route.ts
│   │   ├── posts/route.ts
│   │   └── users/route.ts
│   ├── layout.tsx
│   └── globals.css
│
├── components/
│   ├── ui/                   # 汎用UI
│   │   ├── button.tsx
│   │   ├── dialog.tsx
│   │   └── table.tsx
│   ├── marketing/            # マーケティング専用
│   │   ├── hero.tsx
│   │   └── pricing-table.tsx
│   ├── dashboard/            # ダッシュボード専用
│   │   ├── sidebar.tsx
│   │   └── stats-card.tsx
│   └── layout/               # レイアウト
│       ├── header.tsx
│       └── footer.tsx
│
├── lib/
│   ├── api/                  # API関数
│   │   ├── posts.ts
│   │   └── users.ts
│   ├── utils/                # ユーティリティ
│   │   ├── format.ts
│   │   └── validation.ts
│   └── db.ts
│
├── hooks/                    # カスタムフック
│   ├── use-user.ts
│   ├── use-posts.ts
│   └── use-media-query.ts
│
├── types/                    # 型定義
│   ├── user.ts
│   ├── post.ts
│   └── api.ts
│
└── store/                    # 状態管理
    ├── user-store.ts
    └── ui-store.ts
```

**特徴**:
- 機能ごとに整理
- チーム分担しやすい
- 200-500ファイル程度

### 大規模プロジェクト（20人以上）

```
project/
├── apps/                     # モノレポ構成
│   ├── web/                  # メインアプリ
│   │   ├── app/
│   │   ├── components/
│   │   └── package.json
│   ├── admin/                # 管理画面
│   │   └── ...
│   └── mobile/               # モバイルアプリ
│       └── ...
│
├── packages/                 # 共有パッケージ
│   ├── ui/                   # UIライブラリ
│   │   ├── src/
│   │   │   ├── button/
│   │   │   ├── card/
│   │   │   └── index.ts
│   │   └── package.json
│   ├── utils/                # 共通ユーティリティ
│   │   └── ...
│   └── types/                # 共通型定義
│       └── ...
│
├── .github/                  # CI/CD設定
│   └── workflows/
│       ├── test.yml
│       └── deploy.yml
│
├── pnpm-workspace.yaml
├── turbo.json
└── package.json
```

**特徴**:
- モノレポ構成（Turborepo, pnpm workspace）
- パッケージの共有
- 1000+ファイル

---

## 2. 状態管理アーキテクチャ

### Server State vs Client State

```typescript
// Server State: サーバーから取得するデータ
// → React Query / SWR / TanStack Query
const serverState = {
  users: 'サーバーから取得',
  posts: 'サーバーから取得',
  comments: 'サーバーから取得'
}

// Client State: クライアント固有の状態
// → useState / Zustand / Jotai
const clientState = {
  isDarkMode: 'テーマ設定',
  sidebarOpen: 'UI状態',
  selectedTab: 'UI状態'
}
```

### パターン1: React Query + Zustand

```typescript
// lib/api/posts.ts (Server State)
import { useQuery, useMutation } from '@tanstack/react-query'

export function usePosts() {
  return useQuery({
    queryKey: ['posts'],
    queryFn: async () => {
      const res = await fetch('/api/posts')
      return res.json()
    }
  })
}

export function useCreatePost() {
  return useMutation({
    mutationFn: async (data: CreatePostInput) => {
      const res = await fetch('/api/posts', {
        method: 'POST',
        body: JSON.stringify(data)
      })
      return res.json()
    }
  })
}
```

```typescript
// store/ui-store.ts (Client State)
import { create } from 'zustand'

interface UIStore {
  sidebarOpen: boolean
  toggleSidebar: () => void
  isDarkMode: boolean
  setDarkMode: (value: boolean) => void
}

export const useUIStore = create<UIStore>((set) => ({
  sidebarOpen: true,
  toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
  isDarkMode: false,
  setDarkMode: (value) => set({ isDarkMode: value })
}))
```

### パターン2: Context API（小規模）

```typescript
// contexts/auth-context.tsx
'use client'

import { createContext, useContext, useState, ReactNode } from 'react'

interface AuthContextType {
  user: User | null
  login: (email: string, password: string) => Promise<void>
  logout: () => void
}

const AuthContext = createContext<AuthContextType | undefined>(undefined)

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null)

  const login = async (email: string, password: string) => {
    const res = await fetch('/api/auth/login', {
      method: 'POST',
      body: JSON.stringify({ email, password })
    })
    const data = await res.json()
    setUser(data.user)
  }

  const logout = () => {
    setUser(null)
  }

  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      {children}
    </AuthContext.Provider>
  )
}

export function useAuth() {
  const context = useContext(AuthContext)
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider')
  }
  return context
}
```

---

## 3. コンポーネント設計原則

### コンポーネント分割の基準

#### 1. 単一責任の原則

```tsx
// ❌ 悪い例: 1つのコンポーネントが多すぎる責任
function UserDashboard() {
  const [user, setUser] = useState(null)
  const [posts, setPosts] = useState([])
  const [notifications, setNotifications] = useState([])

  // ユーザー取得、投稿取得、通知取得...
  // レンダリングも全部1つのコンポーネントで

  return (
    <div>
      {/* 500行のJSX */}
    </div>
  )
}

// ✅ 良い例: 責任を分割
function UserDashboard() {
  return (
    <div>
      <UserProfile />
      <PostsList />
      <NotificationPanel />
    </div>
  )
}

function UserProfile() {
  const { data: user } = useUser()
  return <div>{user?.name}</div>
}

function PostsList() {
  const { data: posts } = usePosts()
  return <ul>{posts?.map(post => <PostItem key={post.id} post={post} />)}</ul>
}
```

#### 2. Composition Pattern

```tsx
// ✅ 良い例: コンポーネント合成
interface CardProps {
  children: React.ReactNode
}

function Card({ children }: CardProps) {
  return <div className="card">{children}</div>
}

function CardHeader({ children }: CardProps) {
  return <div className="card-header">{children}</div>
}

function CardBody({ children }: CardProps) {
  return <div className="card-body">{children}</div>
}

function CardFooter({ children }: CardProps) {
  return <div className="card-footer">{children}</div>
}

// Namespace pattern
Card.Header = CardHeader
Card.Body = CardBody
Card.Footer = CardFooter

// 使用例
function ProductCard({ product }: { product: Product }) {
  return (
    <Card>
      <Card.Header>
        <h3>{product.name}</h3>
      </Card.Header>
      <Card.Body>
        <p>{product.description}</p>
      </Card.Body>
      <Card.Footer>
        <button>購入</button>
      </Card.Footer>
    </Card>
  )
}
```

### Props Drillingの回避

```tsx
// ❌ 悪い例: Props Drilling
function App() {
  const [user, setUser] = useState(null)
  return <Dashboard user={user} setUser={setUser} />
}

function Dashboard({ user, setUser }) {
  return <Sidebar user={user} setUser={setUser} />
}

function Sidebar({ user, setUser }) {
  return <UserMenu user={user} setUser={setUser} />
}

function UserMenu({ user, setUser }) {
  return <div>{user?.name}</div>
}

// ✅ 良い例: Context または状態管理
function App() {
  return (
    <AuthProvider>
      <Dashboard />
    </AuthProvider>
  )
}

function UserMenu() {
  const { user } = useAuth()  // どこからでもアクセス
  return <div>{user?.name}</div>
}
```

---

## 4. ファイル命名規則

### コンポーネント

```bash
# ✅ 推奨: PascalCase
components/
├── Button.tsx
├── UserProfile.tsx
└── PostCard.tsx

# コンポーネントごとにフォルダ（複雑な場合）
components/
├── Button/
│   ├── Button.tsx
│   ├── Button.test.tsx
│   ├── Button.stories.tsx
│   └── index.ts
```

### Hooks

```bash
# ✅ 推奨: use-プレフィックス + kebab-case
hooks/
├── use-user.ts
├── use-posts.ts
├── use-media-query.ts
└── use-debounce.ts
```

### Utils / Lib

```bash
# ✅ 推奨: kebab-case
lib/
├── utils/
│   ├── format-date.ts
│   ├── format-currency.ts
│   └── validation.ts
└── api/
    ├── posts.ts
    └── users.ts
```

### Types

```bash
# ✅ 推奨: kebab-case または PascalCase
types/
├── user.ts
├── post.ts
├── api.ts
└── index.ts
```

---

## 5. 環境変数管理

### .env ファイルの分離

```bash
# 開発環境
.env.local

# 本番環境
.env.production

# 全環境共通
.env
```

### Next.js環境変数

```bash
# .env.local

# サーバー側のみ（秘密鍵）
DATABASE_URL="postgresql://..."
API_SECRET_KEY="secret"

# クライアント側にも公開（NEXT_PUBLIC_プレフィックス必須）
NEXT_PUBLIC_API_URL="https://api.example.com"
NEXT_PUBLIC_GA_ID="G-XXXXXXXXXX"
```

### 型安全な環境変数

```typescript
// lib/env.ts
import { z } from 'zod'

const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  API_SECRET_KEY: z.string().min(32),
  NEXT_PUBLIC_API_URL: z.string().url(),
  NEXT_PUBLIC_GA_ID: z.string().optional()
})

export const env = envSchema.parse({
  DATABASE_URL: process.env.DATABASE_URL,
  API_SECRET_KEY: process.env.API_SECRET_KEY,
  NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL,
  NEXT_PUBLIC_GA_ID: process.env.NEXT_PUBLIC_GA_ID
})

// 使用例
import { env } from '@/lib/env'

const apiUrl = env.NEXT_PUBLIC_API_URL  // 型安全
```

---

## 6. 実践例

### 実際のプロジェクト構成（E-Commerceサイト）

```
ecommerce-app/
├── app/
│   ├── (shop)/                # 公開ページ
│   │   ├── layout.tsx
│   │   ├── page.tsx           # トップページ
│   │   ├── products/
│   │   │   ├── page.tsx       # 商品一覧
│   │   │   └── [id]/page.tsx  # 商品詳細
│   │   ├── cart/page.tsx
│   │   └── checkout/page.tsx
│   ├── (auth)/                # 認証ページ
│   │   ├── login/page.tsx
│   │   └── register/page.tsx
│   ├── (account)/             # マイページ
│   │   ├── layout.tsx
│   │   ├── orders/page.tsx
│   │   ├── profile/page.tsx
│   │   └── settings/page.tsx
│   ├── api/
│   │   ├── products/route.ts
│   │   ├── orders/route.ts
│   │   └── checkout/route.ts
│   ├── layout.tsx
│   └── globals.css
│
├── components/
│   ├── ui/                    # 汎用UI
│   │   ├── button.tsx
│   │   ├── card.tsx
│   │   ├── input.tsx
│   │   └── badge.tsx
│   ├── products/              # 商品関連
│   │   ├── product-card.tsx
│   │   ├── product-grid.tsx
│   │   └── product-filters.tsx
│   ├── cart/                  # カート関連
│   │   ├── cart-item.tsx
│   │   └── cart-summary.tsx
│   └── layout/
│       ├── header.tsx
│       ├── footer.tsx
│       └── navbar.tsx
│
├── lib/
│   ├── api/
│   │   ├── products.ts
│   │   ├── orders.ts
│   │   └── users.ts
│   ├── utils/
│   │   ├── format-price.ts
│   │   └── calculate-tax.ts
│   └── db.ts
│
├── hooks/
│   ├── use-cart.ts
│   ├── use-products.ts
│   └── use-checkout.ts
│
├── types/
│   ├── product.ts
│   ├── order.ts
│   ├── cart.ts
│   └── user.ts
│
├── store/
│   ├── cart-store.ts         # カート状態管理
│   └── ui-store.ts
│
└── prisma/
    ├── schema.prisma
    └── seed.ts
```

### コード例

```typescript
// types/product.ts
export interface Product {
  id: string
  name: string
  price: number
  description: string
  imageUrl: string
  category: string
}

// lib/api/products.ts
import { useQuery } from '@tanstack/react-query'
import type { Product } from '@/types/product'

export function useProducts() {
  return useQuery({
    queryKey: ['products'],
    queryFn: async (): Promise<Product[]> => {
      const res = await fetch('/api/products')
      return res.json()
    }
  })
}

// components/products/product-card.tsx
import Image from 'next/image'
import { Card } from '@/components/ui/card'
import { Button } from '@/components/ui/button'
import type { Product } from '@/types/product'

interface ProductCardProps {
  product: Product
  onAddToCart: (product: Product) => void
}

export function ProductCard({ product, onAddToCart }: ProductCardProps) {
  return (
    <Card>
      <Image
        src={product.imageUrl}
        alt={product.name}
        width={300}
        height={300}
      />
      <h3>{product.name}</h3>
      <p>${product.price}</p>
      <Button onClick={() => onAddToCart(product)}>
        カートに追加
      </Button>
    </Card>
  )
}

// app/(shop)/products/page.tsx
import { ProductGrid } from '@/components/products/product-grid'

export default async function ProductsPage() {
  const products = await fetch('https://api.example.com/products').then(r => r.json())

  return (
    <div>
      <h1>商品一覧</h1>
      <ProductGrid products={products} />
    </div>
  )
}
```

---

## まとめ

この章では、プロジェクトアーキテクチャ設計を学びました。

**重要ポイント**:
- ✅ チーム規模に応じたディレクトリ構成を選択
- ✅ Server StateとClient Stateを分けて管理
- ✅ 単一責任の原則でコンポーネントを分割
- ✅ Props Drillingは状態管理で回避
- ✅ ファイル命名規則を統一
- ✅ 環境変数は型安全に管理

**次章予告**: Chapter 9では、本番運用とデプロイメントを学びます。

---

**参考リンク**:
- [Next.js Project Structure](https://nextjs.org/docs/getting-started/project-structure)
- [Bulletproof React](https://github.com/alan2207/bulletproof-react)
- [React TypeScript Cheatsheet](https://react-typescript-cheatsheet.netlify.app/)
