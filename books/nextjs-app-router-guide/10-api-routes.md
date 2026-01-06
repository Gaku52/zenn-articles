---
title: "API Routes - RESTful APIの構築"
---

# API Routes

本章では、Next.js App RouterのRoute Handlersを使ったRESTful API構築を学びます。

## Route Handlersの基礎

### 基本的なGETエンドポイント

```tsx:app/api/hello/route.ts
export async function GET() {
  return Response.json({ message: 'Hello, World!' })
}
```

**アクセス:** `GET /api/hello`

**レスポンス:**
```json
{
  "message": "Hello, World!"
}
```

### HTTPメソッド

```tsx:app/api/posts/route.ts
import { NextRequest } from 'next/server'

// GET /api/posts
export async function GET(request: NextRequest) {
  return Response.json({ posts: [] })
}

// POST /api/posts
export async function POST(request: NextRequest) {
  const body = await request.json()
  return Response.json({ success: true }, { status: 201 })
}

// PUT /api/posts
export async function PUT(request: NextRequest) {
  const body = await request.json()
  return Response.json({ success: true })
}

// DELETE /api/posts
export async function DELETE(request: NextRequest) {
  return new Response(null, { status: 204 })
}

// PATCH /api/posts
export async function PATCH(request: NextRequest) {
  const body = await request.json()
  return Response.json({ success: true })
}
```

## リクエストの処理

### クエリパラメータ

```tsx:app/api/search/route.ts
import { NextRequest } from 'next/server'

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams
  const query = searchParams.get('q')
  const page = searchParams.get('page') || '1'
  const limit = searchParams.get('limit') || '10'

  return Response.json({
    query,
    page: parseInt(page),
    limit: parseInt(limit),
  })
}
```

**使用例:**
```
GET /api/search?q=nextjs&page=2&limit=20
```

### リクエストボディ

```tsx:app/api/users/route.ts
import { NextRequest } from 'next/server'
import { z } from 'zod'

const UserSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  age: z.number().min(0).optional(),
})

export async function POST(request: NextRequest) {
  try {
    const body = await request.json()

    // バリデーション
    const validated = UserSchema.parse(body)

    // ユーザー作成処理
    const user = await createUser(validated)

    return Response.json(user, { status: 201 })
  } catch (error) {
    if (error instanceof z.ZodError) {
      return Response.json(
        { error: 'Validation failed', details: error.errors },
        { status: 400 }
      )
    }

    return Response.json(
      { error: 'Internal server error' },
      { status: 500 }
    )
  }
}
```

### ヘッダーの取得

```tsx:app/api/auth/route.ts
import { NextRequest } from 'next/server'

export async function GET(request: NextRequest) {
  const authorization = request.headers.get('authorization')
  const userAgent = request.headers.get('user-agent')

  if (!authorization) {
    return Response.json(
      { error: 'Unauthorized' },
      { status: 401 }
    )
  }

  return Response.json({
    authorized: true,
    userAgent,
  })
}
```

### Cookieの取得・設定

```tsx:app/api/session/route.ts
import { cookies } from 'next/headers'
import { NextRequest } from 'next/server'

export async function GET() {
  const cookieStore = cookies()
  const session = cookieStore.get('session')

  return Response.json({ session: session?.value })
}

export async function POST(request: NextRequest) {
  const body = await request.json()
  const sessionId = generateSessionId()

  cookies().set('session', sessionId, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 60 * 60 * 24 * 7, // 7日間
  })

  return Response.json({ success: true })
}
```

## 動的ルート

### パスパラメータ

```tsx:app/api/posts/[id]/route.ts
import { NextRequest } from 'next/server'
import { prisma } from '@/lib/prisma'

interface RouteContext {
  params: { id: string }
}

// GET /api/posts/:id
export async function GET(
  request: NextRequest,
  { params }: RouteContext
) {
  const post = await prisma.post.findUnique({
    where: { id: params.id },
  })

  if (!post) {
    return Response.json(
      { error: 'Post not found' },
      { status: 404 }
    )
  }

  return Response.json(post)
}

// PUT /api/posts/:id
export async function PUT(
  request: NextRequest,
  { params }: RouteContext
) {
  const body = await request.json()

  const post = await prisma.post.update({
    where: { id: params.id },
    data: body,
  })

  return Response.json(post)
}

// DELETE /api/posts/:id
export async function DELETE(
  request: NextRequest,
  { params }: RouteContext
) {
  await prisma.post.delete({
    where: { id: params.id },
  })

  return new Response(null, { status: 204 })
}
```

### 複数のパラメータ

```tsx:app/api/users/[userId]/posts/[postId]/route.ts
import { NextRequest } from 'next/server'

interface RouteContext {
  params: {
    userId: string
    postId: string
  }
}

export async function GET(
  request: NextRequest,
  { params }: RouteContext
) {
  const post = await prisma.post.findFirst({
    where: {
      id: params.postId,
      authorId: params.userId,
    },
  })

  return Response.json(post)
}
```

**アクセス:** `GET /api/users/123/posts/456`

## データベース統合

### CRUD操作の完全実装

```tsx:app/api/posts/route.ts
import { NextRequest } from 'next/server'
import { prisma } from '@/lib/prisma'
import { z } from 'zod'

const PostSchema = z.object({
  title: z.string().min(1).max(100),
  content: z.string().min(1),
  published: z.boolean().default(false),
})

// GET /api/posts - 一覧取得
export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams
  const page = parseInt(searchParams.get('page') || '1')
  const limit = parseInt(searchParams.get('limit') || '10')
  const skip = (page - 1) * limit

  const [posts, total] = await Promise.all([
    prisma.post.findMany({
      take: limit,
      skip,
      include: {
        author: {
          select: {
            id: true,
            name: true,
            email: true,
          },
        },
      },
      orderBy: { createdAt: 'desc' },
    }),
    prisma.post.count(),
  ])

  return Response.json({
    posts,
    pagination: {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit),
    },
  })
}

// POST /api/posts - 作成
export async function POST(request: NextRequest) {
  try {
    const body = await request.json()
    const validated = PostSchema.parse(body)

    const post = await prisma.post.create({
      data: {
        ...validated,
        authorId: 'user-id', // 実際は認証から取得
      },
      include: {
        author: true,
      },
    })

    return Response.json(post, { status: 201 })
  } catch (error) {
    if (error instanceof z.ZodError) {
      return Response.json(
        { error: 'Validation failed', details: error.errors },
        { status: 400 }
      )
    }

    return Response.json(
      { error: 'Internal server error' },
      { status: 500 }
    )
  }
}
```

```tsx:app/api/posts/[id]/route.ts
import { NextRequest } from 'next/server'
import { prisma } from '@/lib/prisma'
import { z } from 'zod'

interface RouteContext {
  params: { id: string }
}

const UpdatePostSchema = z.object({
  title: z.string().min(1).max(100).optional(),
  content: z.string().min(1).optional(),
  published: z.boolean().optional(),
})

// GET /api/posts/:id - 詳細取得
export async function GET(
  request: NextRequest,
  { params }: RouteContext
) {
  const post = await prisma.post.findUnique({
    where: { id: params.id },
    include: {
      author: true,
      comments: {
        include: { author: true },
        orderBy: { createdAt: 'desc' },
      },
    },
  })

  if (!post) {
    return Response.json(
      { error: 'Post not found' },
      { status: 404 }
    )
  }

  return Response.json(post)
}

// PATCH /api/posts/:id - 更新
export async function PATCH(
  request: NextRequest,
  { params }: RouteContext
) {
  try {
    const body = await request.json()
    const validated = UpdatePostSchema.parse(body)

    const post = await prisma.post.update({
      where: { id: params.id },
      data: validated,
    })

    return Response.json(post)
  } catch (error) {
    if (error instanceof z.ZodError) {
      return Response.json(
        { error: 'Validation failed', details: error.errors },
        { status: 400 }
      )
    }

    return Response.json(
      { error: 'Internal server error' },
      { status: 500 }
    )
  }
}

// DELETE /api/posts/:id - 削除
export async function DELETE(
  request: NextRequest,
  { params }: RouteContext
) {
  try {
    await prisma.post.delete({
      where: { id: params.id },
    })

    return new Response(null, { status: 204 })
  } catch (error) {
    return Response.json(
      { error: 'Post not found' },
      { status: 404 }
    )
  }
}
```

## 認証とミドルウェア

### JWT認証

```bash
npm install jsonwebtoken
npm install --save-dev @types/jsonwebtoken
```

```tsx:lib/auth.ts
import jwt from 'jsonwebtoken'

const JWT_SECRET = process.env.JWT_SECRET || 'your-secret-key'

export function generateToken(userId: string) {
  return jwt.sign({ userId }, JWT_SECRET, { expiresIn: '7d' })
}

export function verifyToken(token: string) {
  try {
    return jwt.verify(token, JWT_SECRET) as { userId: string }
  } catch (error) {
    return null
  }
}
```

```tsx:app/api/auth/login/route.ts
import { NextRequest } from 'next/server'
import { prisma } from '@/lib/prisma'
import { generateToken } from '@/lib/auth'
import bcrypt from 'bcrypt'

export async function POST(request: NextRequest) {
  const { email, password } = await request.json()

  const user = await prisma.user.findUnique({
    where: { email },
  })

  if (!user) {
    return Response.json(
      { error: 'Invalid credentials' },
      { status: 401 }
    )
  }

  const isValid = await bcrypt.compare(password, user.password)

  if (!isValid) {
    return Response.json(
      { error: 'Invalid credentials' },
      { status: 401 }
    )
  }

  const token = generateToken(user.id)

  return Response.json({
    user: {
      id: user.id,
      name: user.name,
      email: user.email,
    },
    token,
  })
}
```

### 認証ミドルウェア

```tsx:lib/middleware.ts
import { NextRequest } from 'next/server'
import { verifyToken } from './auth'

export function withAuth(handler: Function) {
  return async (request: NextRequest, context: any) => {
    const authorization = request.headers.get('authorization')

    if (!authorization || !authorization.startsWith('Bearer ')) {
      return Response.json(
        { error: 'Unauthorized' },
        { status: 401 }
      )
    }

    const token = authorization.substring(7)
    const payload = verifyToken(token)

    if (!payload) {
      return Response.json(
        { error: 'Invalid token' },
        { status: 401 }
      )
    }

    // リクエストにユーザー情報を追加
    request.headers.set('x-user-id', payload.userId)

    return handler(request, context)
  }
}
```

**使用例:**
```tsx:app/api/profile/route.ts
import { NextRequest } from 'next/server'
import { withAuth } from '@/lib/middleware'
import { prisma } from '@/lib/prisma'

async function handler(request: NextRequest) {
  const userId = request.headers.get('x-user-id')!

  const user = await prisma.user.findUnique({
    where: { id: userId },
    select: {
      id: true,
      name: true,
      email: true,
    },
  })

  return Response.json(user)
}

export const GET = withAuth(handler)
```

## エラーハンドリング

### 統一されたエラーレスポンス

```tsx:lib/errors.ts
export class APIError extends Error {
  constructor(
    public statusCode: number,
    message: string,
    public code?: string
  ) {
    super(message)
    this.name = 'APIError'
  }
}

export function errorResponse(error: unknown) {
  if (error instanceof APIError) {
    return Response.json(
      {
        error: error.message,
        code: error.code,
      },
      { status: error.statusCode }
    )
  }

  console.error('Unexpected error:', error)

  return Response.json(
    { error: 'Internal server error' },
    { status: 500 }
  )
}
```

**使用例:**
```tsx:app/api/posts/[id]/route.ts
import { NextRequest } from 'next/server'
import { prisma } from '@/lib/prisma'
import { APIError, errorResponse } from '@/lib/errors'

export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  try {
    const post = await prisma.post.findUnique({
      where: { id: params.id },
    })

    if (!post) {
      throw new APIError(404, 'Post not found', 'POST_NOT_FOUND')
    }

    return Response.json(post)
  } catch (error) {
    return errorResponse(error)
  }
}
```

## レート制限

```tsx:lib/rate-limit.ts
import { NextRequest } from 'next/server'

const rateLimit = new Map<string, { count: number; resetAt: number }>()

export function checkRateLimit(
  request: NextRequest,
  limit: number = 100,
  window: number = 60 * 1000 // 1分
) {
  const ip = request.ip || 'unknown'
  const now = Date.now()

  const record = rateLimit.get(ip)

  if (!record || now > record.resetAt) {
    rateLimit.set(ip, { count: 1, resetAt: now + window })
    return { allowed: true, remaining: limit - 1 }
  }

  if (record.count >= limit) {
    return { allowed: false, remaining: 0 }
  }

  record.count++
  return { allowed: true, remaining: limit - record.count }
}
```

**使用例:**
```tsx:app/api/search/route.ts
import { NextRequest } from 'next/server'
import { checkRateLimit } from '@/lib/rate-limit'

export async function GET(request: NextRequest) {
  const { allowed, remaining } = checkRateLimit(request, 30, 60 * 1000)

  if (!allowed) {
    return Response.json(
      { error: 'Too many requests' },
      {
        status: 429,
        headers: {
          'X-RateLimit-Remaining': '0',
          'X-RateLimit-Reset': String(Date.now() + 60 * 1000),
        },
      }
    )
  }

  // 検索処理...

  return Response.json(
    { results: [] },
    {
      headers: {
        'X-RateLimit-Remaining': String(remaining),
      },
    }
  )
}
```

## CORS設定

```tsx:app/api/public/route.ts
import { NextRequest } from 'next/server'

export async function GET(request: NextRequest) {
  return Response.json(
    { message: 'Public API' },
    {
      headers: {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
        'Access-Control-Allow-Headers': 'Content-Type, Authorization',
      },
    }
  )
}

export async function OPTIONS(request: NextRequest) {
  return new Response(null, {
    status: 204,
    headers: {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type, Authorization',
    },
  })
}
```

## ストリーミングレスポンス

```tsx:app/api/stream/route.ts
export async function GET() {
  const encoder = new TextEncoder()

  const stream = new ReadableStream({
    async start(controller) {
      for (let i = 0; i < 10; i++) {
        const message = `データ ${i + 1}\n`
        controller.enqueue(encoder.encode(message))
        await new Promise(resolve => setTimeout(resolve, 1000))
      }
      controller.close()
    },
  })

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/plain',
      'Transfer-Encoding': 'chunked',
    },
  })
}
```

## パフォーマンス最適化

### キャッシング

```tsx:app/api/stats/route.ts
export async function GET() {
  const stats = await getStats()

  return Response.json(stats, {
    headers: {
      'Cache-Control': 'public, s-maxage=60, stale-while-revalidate=30',
    },
  })
}
```

### データベースクエリ最適化

```tsx
// ❌ 悪い例: N+1問題
const posts = await prisma.post.findMany()
const postsWithAuthors = await Promise.all(
  posts.map(async post => ({
    ...post,
    author: await prisma.user.findUnique({ where: { id: post.authorId } })
  }))
)

// ✅ 良い例: include
const posts = await prisma.post.findMany({
  include: { author: true }
})
```

## まとめ

本章で学んだこと:

✅ Route Handlersの基本とHTTPメソッド
✅ リクエスト/レスポンスの処理
✅ 動的ルートとパラメータ
✅ データベース統合とCRUD操作
✅ JWT認証とミドルウェア
✅ エラーハンドリングとレート制限
✅ CORS設定とストリーミング

次章では、データベース統合の詳細を学びます。
