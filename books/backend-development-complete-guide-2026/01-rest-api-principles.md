---
title: "REST API設計原則"
---

# REST API設計原則

## この章で学ぶこと

この章では、REST APIの基礎から実践的な設計原則まで、包括的に学習します。単なる理論だけでなく、実際のプロダクション環境で使用されている設計パターン、パフォーマンス最適化、トラブルシューティングまでカバーします。

- RESTの6つの制約とその実装方法
- リソース設計の原則とベストプラクティス
- HTTPメソッドとステータスコードの正しい使用法
- 統一されたレスポンス形式の設計
- ページネーション、フィルタリング、ソートの実装
- バージョニング戦略
- HATEOASの実装
- 実測データとパフォーマンス改善事例

## RESTとは

REST (Representational State Transfer) は、Roy Fieldingによって2000年に提唱されたアーキテクチャスタイルです。Web APIの設計において最も広く採用されている標準的なアプローチです。

### RESTの6つの制約

RESTful APIを設計する際、以下の6つの制約に従う必要があります。これらの制約を理解し、適切に実装することで、スケーラブルで保守性の高いAPIを構築できます。

#### 1. クライアント・サーバー (Client-Server)

クライアントとサーバーの責務を明確に分離します。

**原則:**
- クライアント: ユーザーインターフェースとユーザー体験を担当
- サーバー: データストレージとビジネスロジックを担当

**メリット:**
- 独立した進化: クライアントとサーバーを個別に開発・デプロイ可能
- スケーラビリティ: サーバー側を独立してスケール可能
- 再利用性: 同じAPIを複数のクライアント（Web、モバイル、IoT）で使用可能

**実装例:**

```typescript
// クライアント側 (React)
// UIとデータ取得ロジックを分離
import { useEffect, useState } from 'react'

function UserList() {
  const [users, setUsers] = useState([])

  useEffect(() => {
    fetch('https://api.example.com/users')
      .then(res => res.json())
      .then(data => setUsers(data))
  }, [])

  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  )
}
```

```typescript
// サーバー側 (Express)
// ビジネスロジックとデータ管理を担当
import express from 'express'
import { PrismaClient } from '@prisma/client'

const app = express()
const prisma = new PrismaClient()

app.get('/users', async (req, res) => {
  const users = await prisma.user.findMany({
    select: {
      id: true,
      name: true,
      email: true,
    },
  })

  res.json(users)
})
```

#### 2. ステートレス (Stateless)

各リクエストは完全に独立しており、サーバーはクライアントの状態を保持しません。

**原則:**
- リクエストに必要な情報をすべて含める
- サーバーはセッション情報を保存しない
- 認証情報は各リクエストに含める（JWT等）

**メリット:**
- スケーラビリティ: サーバーインスタンスを自由に追加・削除可能
- 信頼性: サーバー障害時の影響を最小化
- シンプル性: セッション管理の複雑さを排除

**実装例:**

```typescript
// ✅ ステートレス - 正しい実装
import { Request, Response, NextFunction } from 'express'
import jwt from 'jsonwebtoken'

interface JwtPayload {
  userId: string
  email: string
  role: string
}

// すべてのリクエストに認証情報を含める
app.get('/users/me', authenticate, async (req, res) => {
  const user = await prisma.user.findUnique({
    where: { id: req.user.userId },
  })

  res.json(user)
})

// 認証ミドルウェア - トークンから状態を復元
function authenticate(req: Request, res: Response, next: NextFunction) {
  const token = req.headers.authorization?.split(' ')[1]

  if (!token) {
    return res.status(401).json({ error: 'No token provided' })
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!) as JwtPayload
    req.user = decoded
    next()
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' })
  }
}
```

```typescript
// ❌ ステートフル - 避けるべき実装
import session from 'express-session'

app.use(session({
  secret: 'secret',
  resave: false,
  saveUninitialized: true,
}))

// セッションに状態を保存（スケールしにくい）
app.post('/login', async (req, res) => {
  const user = await authenticateUser(req.body)
  req.session.userId = user.id // サーバーに状態を保存
  res.json({ success: true })
})

app.get('/users/me', (req, res) => {
  const userId = req.session.userId // セッションから取得
  // ...
})
```

#### 3. キャッシュ可能 (Cacheable)

レスポンスにキャッシュ可能かどうかの情報を含め、クライアント側でキャッシュできるようにします。

**原則:**
- レスポンスに適切なキャッシュヘッダーを設定
- 変更頻度に応じたキャッシュ戦略を選択
- ETags や Last-Modified ヘッダーを活用

**メリット:**
- パフォーマンス: ネットワーク往復を削減
- スケーラビリティ: サーバー負荷を軽減
- オフライン対応: キャッシュデータで動作可能

**実装例:**

```typescript
// src/middleware/cache.ts
import { Request, Response, NextFunction } from 'express'
import crypto from 'crypto'

// ETag生成ミドルウェア
export function etag() {
  return (req: Request, res: Response, next: NextFunction) => {
    const originalSend = res.json.bind(res)

    res.json = function(data: any) {
      const body = JSON.stringify(data)
      const hash = crypto.createHash('md5').update(body).digest('hex')

      res.setHeader('ETag', `"${hash}"`)

      // クライアントのETagと比較
      const clientEtag = req.headers['if-none-match']
      if (clientEtag === `"${hash}"`) {
        return res.status(304).end()
      }

      return originalSend(data)
    }

    next()
  }
}

// Cache-Controlヘッダーの設定
export function cacheControl(maxAge: number) {
  return (req: Request, res: Response, next: NextFunction) => {
    res.setHeader('Cache-Control', `public, max-age=${maxAge}`)
    next()
  }
}

// 使用例
app.get('/users/:id',
  etag(),
  cacheControl(3600), // 1時間キャッシュ
  async (req, res) => {
    const user = await prisma.user.findUnique({
      where: { id: req.params.id },
    })

    res.json(user)
  }
)

// 動的コンテンツのキャッシュ制御
app.get('/users',
  etag(),
  async (req, res) => {
    const users = await prisma.user.findMany()

    // 短時間のキャッシュ（5分）
    res.setHeader('Cache-Control', 'public, max-age=300')
    res.json(users)
  }
)

// キャッシュしないエンドポイント
app.get('/users/me',
  authenticate,
  async (req, res) => {
    const user = await prisma.user.findUnique({
      where: { id: req.user.userId },
    })

    // プライベートデータはキャッシュしない
    res.setHeader('Cache-Control', 'private, no-cache, no-store, must-revalidate')
    res.json(user)
  }
)
```

**Redisを使用したサーバー側キャッシュ:**

```typescript
// src/middleware/redis-cache.ts
import { Request, Response, NextFunction } from 'express'
import { createClient } from 'redis'

const redisClient = createClient({
  url: process.env.REDIS_URL,
})

redisClient.connect()

export function redisCache(ttl: number = 3600) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const key = `cache:${req.originalUrl}`

    try {
      const cached = await redisClient.get(key)

      if (cached) {
        console.log('Cache hit:', key)
        return res.json(JSON.parse(cached))
      }

      // レスポンスをキャッシュ
      const originalJson = res.json.bind(res)
      res.json = function(data: any) {
        redisClient.setEx(key, ttl, JSON.stringify(data))
        return originalJson(data)
      }

      next()
    } catch (error) {
      console.error('Redis error:', error)
      next()
    }
  }
}

// 使用例
app.get('/users',
  redisCache(300), // 5分間キャッシュ
  async (req, res) => {
    const users = await prisma.user.findMany()
    res.json(users)
  }
)
```

#### 4. 統一インターフェース (Uniform Interface)

APIのインターフェースを統一し、予測可能にします。

**4つの制約:**

1. **リソース識別**: URIでリソースを一意に識別
2. **表現によるリソース操作**: JSON等の表現形式でリソースを操作
3. **自己記述的メッセージ**: メッセージだけで処理方法が分かる
4. **ハイパーメディア**: レスポンスに関連リソースへのリンクを含める (HATEOAS)

**実装例:**

```typescript
// 1. リソース識別 - URIの設計
const routes = {
  // ✅ 良い設計
  users: '/users',
  user: '/users/:id',
  userPosts: '/users/:id/posts',
  post: '/posts/:id',

  // ❌ 悪い設計
  // getUser: '/getUser?id=123',
  // createNewUser: '/createNewUser',
}

// 2. 表現によるリソース操作
interface User {
  id: string
  email: string
  name: string
  createdAt: Date
  updatedAt: Date
}

interface CreateUserDto {
  email: string
  name: string
  password: string
}

// 3. 自己記述的メッセージ
app.post('/users', async (req, res) => {
  const { email, name, password }: CreateUserDto = req.body

  const user = await prisma.user.create({
    data: { email, name, password },
  })

  res.status(201)
    .setHeader('Content-Type', 'application/json')
    .setHeader('Location', `/users/${user.id}`)
    .json({
      success: true,
      data: user,
      meta: {
        timestamp: new Date().toISOString(),
      },
    })
})

// 4. HATEOAS - ハイパーメディアリンク
app.get('/users/:id', async (req, res) => {
  const user = await prisma.user.findUnique({
    where: { id: req.params.id },
  })

  res.json({
    success: true,
    data: {
      ...user,
      _links: {
        self: { href: `/users/${user.id}` },
        posts: { href: `/users/${user.id}/posts` },
        comments: { href: `/users/${user.id}/comments` },
      },
    },
  })
})
```

#### 5. 階層化システム (Layered System)

クライアントはサーバーの内部構造を知る必要がありません。

**原則:**
- ロードバランサー、キャッシュ、プロキシを透過的に配置可能
- クライアントは直接通信している相手がエンドサーバーか中間サーバーか区別不要

**実装例:**

```typescript
// リバースプロキシ (nginx) の設定
/*
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
}

server {
    listen 80;
    server_name api.example.com;

    location /api {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
*/

// Express側 - プロキシを意識した実装
import express from 'express'

const app = express()

// プロキシ背後での動作を有効化
app.set('trust proxy', true)

app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    server: process.env.SERVER_ID,
  })
})

// 実際のIPアドレスを取得
app.get('/my-ip', (req, res) => {
  res.json({
    ip: req.ip, // プロキシ経由でも正しいIPを取得
  })
})
```

#### 6. コードオンデマンド (Code-On-Demand) - オプション

クライアントの機能を動的に拡張できます（JavaScriptのダウンロード等）。

この制約は**オプション**であり、必須ではありません。

## リソース設計の原則

RESTにおいて最も重要なのは**リソース**の設計です。適切なリソース設計により、直感的で保守性の高いAPIを構築できます。

### リソース命名規則

#### 基本原則

1. **名詞を使用** - 動詞ではなく名詞でリソースを表現
2. **複数形を使用** - コレクションは複数形
3. **小文字とハイフン** - kebab-case を使用
4. **階層構造** - 親子関係を明確に

**良い設計例:**

```
GET    /users                    # ユーザー一覧
GET    /users/123                # 特定ユーザー
GET    /users/123/posts          # ユーザーの投稿一覧
GET    /users/123/posts/456      # ユーザーの特定投稿
GET    /posts/456/comments       # 投稿のコメント一覧
GET    /blog-posts               # ブログ投稿一覧
```

**悪い設計例:**

```
GET    /getUsers                 # 動詞を使用
GET    /user/123                 # 単数形
GET    /users/getPosts/123       # 動詞が混在
GET    /blogPosts                # camelCase
GET    /users_posts              # snake_case
```

### HTTPメソッドの使用

各HTTPメソッドには明確な意味があります。

| メソッド | 用途 | 冪等性 | 安全性 |
|---------|------|--------|--------|
| **GET** | リソース取得 | ✅ | ✅ |
| **POST** | リソース作成 | ❌ | ❌ |
| **PUT** | リソース置換（全体更新） | ✅ | ❌ |
| **PATCH** | リソース部分更新 | ❌ | ❌ |
| **DELETE** | リソース削除 | ✅ | ❌ |

**冪等性**: 同じリクエストを複数回実行しても結果が同じ
**安全性**: リソースの状態を変更しない

### 完全なCRUD実装例

```typescript
// src/types/user.types.ts
export interface User {
  id: string
  email: string
  name: string
  role: 'user' | 'admin'
  createdAt: Date
  updatedAt: Date
}

export interface CreateUserDto {
  email: string
  name: string
  password: string
  role?: 'user' | 'admin'
}

export interface UpdateUserDto {
  email?: string
  name?: string
  role?: 'user' | 'admin'
}

export interface PaginationQuery {
  page?: number
  limit?: number
  sortBy?: string
  order?: 'asc' | 'desc'
  search?: string
}
```

```typescript
// src/controllers/user.controller.ts
import { Request, Response, NextFunction } from 'express'
import { PrismaClient } from '@prisma/client'
import { CreateUserDto, UpdateUserDto, PaginationQuery } from '../types/user.types'
import { AppError } from '../utils/errors'
import bcrypt from 'bcrypt'

const prisma = new PrismaClient()

export class UserController {
  // GET /users - ユーザー一覧取得
  async getUsers(
    req: Request<{}, {}, {}, PaginationQuery>,
    res: Response,
    next: NextFunction
  ) {
    try {
      const {
        page = 1,
        limit = 10,
        sortBy = 'createdAt',
        order = 'desc',
        search = '',
      } = req.query

      const skip = (Number(page) - 1) * Number(limit)

      // WHERE条件の構築
      const where = search ? {
        OR: [
          { name: { contains: search, mode: 'insensitive' as const } },
          { email: { contains: search, mode: 'insensitive' as const } },
        ],
      } : {}

      // 並列実行でパフォーマンス向上
      const [users, total] = await Promise.all([
        prisma.user.findMany({
          where,
          skip,
          take: Number(limit),
          orderBy: { [sortBy]: order },
          select: {
            id: true,
            email: true,
            name: true,
            role: true,
            createdAt: true,
            updatedAt: true,
            // パスワードは除外
          },
        }),
        prisma.user.count({ where }),
      ])

      const totalPages = Math.ceil(total / Number(limit))

      res.json({
        success: true,
        data: users,
        meta: {
          page: Number(page),
          limit: Number(limit),
          total,
          totalPages,
          hasNextPage: Number(page) < totalPages,
          hasPreviousPage: Number(page) > 1,
        },
      })
    } catch (error) {
      next(error)
    }
  }

  // GET /users/:id - 特定ユーザー取得
  async getUserById(
    req: Request<{ id: string }>,
    res: Response,
    next: NextFunction
  ) {
    try {
      const user = await prisma.user.findUnique({
        where: { id: req.params.id },
        select: {
          id: true,
          email: true,
          name: true,
          role: true,
          createdAt: true,
          updatedAt: true,
        },
      })

      if (!user) {
        throw new AppError('User not found', 404, 'USER_NOT_FOUND')
      }

      res.json({
        success: true,
        data: user,
      })
    } catch (error) {
      next(error)
    }
  }

  // POST /users - ユーザー作成
  async createUser(
    req: Request<{}, {}, CreateUserDto>,
    res: Response,
    next: NextFunction
  ) {
    try {
      const { email, name, password, role = 'user' } = req.body

      // メールアドレスの重複チェック
      const existing = await prisma.user.findUnique({
        where: { email },
      })

      if (existing) {
        throw new AppError(
          'Email already exists',
          409,
          'EMAIL_ALREADY_EXISTS'
        )
      }

      // パスワードのハッシュ化
      const hashedPassword = await bcrypt.hash(password, 10)

      const user = await prisma.user.create({
        data: {
          email,
          name,
          password: hashedPassword,
          role,
        },
        select: {
          id: true,
          email: true,
          name: true,
          role: true,
          createdAt: true,
          updatedAt: true,
        },
      })

      res.status(201)
        .setHeader('Location', `/users/${user.id}`)
        .json({
          success: true,
          data: user,
        })
    } catch (error) {
      next(error)
    }
  }

  // PUT /users/:id - ユーザー全体更新
  async replaceUser(
    req: Request<{ id: string }, {}, CreateUserDto>,
    res: Response,
    next: NextFunction
  ) {
    try {
      const { email, name, password, role = 'user' } = req.body

      // ユーザーの存在確認
      const existing = await prisma.user.findUnique({
        where: { id: req.params.id },
      })

      if (!existing) {
        throw new AppError('User not found', 404, 'USER_NOT_FOUND')
      }

      const hashedPassword = await bcrypt.hash(password, 10)

      const user = await prisma.user.update({
        where: { id: req.params.id },
        data: {
          email,
          name,
          password: hashedPassword,
          role,
        },
        select: {
          id: true,
          email: true,
          name: true,
          role: true,
          createdAt: true,
          updatedAt: true,
        },
      })

      res.json({
        success: true,
        data: user,
      })
    } catch (error) {
      next(error)
    }
  }

  // PATCH /users/:id - ユーザー部分更新
  async updateUser(
    req: Request<{ id: string }, {}, UpdateUserDto>,
    res: Response,
    next: NextFunction
  ) {
    try {
      const updates = req.body

      // 空のリクエストをチェック
      if (Object.keys(updates).length === 0) {
        throw new AppError(
          'No update fields provided',
          400,
          'NO_UPDATE_FIELDS'
        )
      }

      const user = await prisma.user.update({
        where: { id: req.params.id },
        data: updates,
        select: {
          id: true,
          email: true,
          name: true,
          role: true,
          createdAt: true,
          updatedAt: true,
        },
      })

      res.json({
        success: true,
        data: user,
      })
    } catch (error) {
      next(error)
    }
  }

  // DELETE /users/:id - ユーザー削除
  async deleteUser(
    req: Request<{ id: string }>,
    res: Response,
    next: NextFunction
  ) {
    try {
      await prisma.user.delete({
        where: { id: req.params.id },
      })

      // 204 No Content - 成功したが返すデータなし
      res.status(204).send()
    } catch (error) {
      next(error)
    }
  }
}
```

```typescript
// src/routes/user.routes.ts
import { Router } from 'express'
import { UserController } from '../controllers/user.controller'
import { authenticate, authorize } from '../middleware/auth'
import { validate } from '../middleware/validation'
import {
  createUserSchema,
  updateUserSchema,
  paginationSchema,
} from '../schemas/user.schema'

const router = Router()
const userController = new UserController()

// GET /users - 一覧取得（認証必須）
router.get(
  '/',
  authenticate,
  validate(paginationSchema, 'query'),
  userController.getUsers.bind(userController)
)

// GET /users/:id - 詳細取得（認証必須）
router.get(
  '/:id',
  authenticate,
  userController.getUserById.bind(userController)
)

// POST /users - 作成（管理者のみ）
router.post(
  '/',
  authenticate,
  authorize('admin'),
  validate(createUserSchema),
  userController.createUser.bind(userController)
)

// PUT /users/:id - 全体更新（管理者のみ）
router.put(
  '/:id',
  authenticate,
  authorize('admin'),
  validate(createUserSchema),
  userController.replaceUser.bind(userController)
)

// PATCH /users/:id - 部分更新（本人または管理者）
router.patch(
  '/:id',
  authenticate,
  validate(updateUserSchema),
  userController.updateUser.bind(userController)
)

// DELETE /users/:id - 削除（管理者のみ）
router.delete(
  '/:id',
  authenticate,
  authorize('admin'),
  userController.deleteUser.bind(userController)
)

export default router
```

## HTTPステータスコードの正しい使用

適切なステータスコードを返すことで、クライアント側でのエラーハンドリングが容易になります。

### 成功レスポンス (2xx)

| コード | 名前 | 使用例 |
|-------|------|--------|
| **200** | OK | GET/PUT/PATCHの成功 |
| **201** | Created | POSTでリソース作成成功 |
| **202** | Accepted | 非同期処理を受け付けた |
| **204** | No Content | DELETE成功（レスポンスボディなし） |

```typescript
// 200 OK - データを返す
app.get('/users/:id', async (req, res) => {
  const user = await findUser(req.params.id)
  res.status(200).json({ data: user })
})

// 201 Created - 新規リソースを作成
app.post('/users', async (req, res) => {
  const user = await createUser(req.body)
  res.status(201)
    .setHeader('Location', `/users/${user.id}`)
    .json({ data: user })
})

// 202 Accepted - 非同期処理を受け付け
app.post('/users/:id/send-email', async (req, res) => {
  const jobId = await queueEmailJob(req.params.id, req.body)
  res.status(202).json({
    message: 'Email job queued',
    jobId,
    statusUrl: `/jobs/${jobId}`,
  })
})

// 204 No Content - 成功したがレスポンスなし
app.delete('/users/:id', async (req, res) => {
  await deleteUser(req.params.id)
  res.status(204).send()
})
```

### クライアントエラー (4xx)

| コード | 名前 | 使用例 |
|-------|------|--------|
| **400** | Bad Request | バリデーションエラー |
| **401** | Unauthorized | 認証エラー |
| **403** | Forbidden | 権限不足 |
| **404** | Not Found | リソースが存在しない |
| **405** | Method Not Allowed | HTTPメソッドが許可されていない |
| **409** | Conflict | リソースの競合（重複等） |
| **422** | Unprocessable Entity | セマンティックエラー |
| **429** | Too Many Requests | レート制限超過 |

```typescript
// 400 Bad Request - バリデーションエラー
app.post('/users', validate(createUserSchema), async (req, res) => {
  // バリデーション失敗時は自動的に400を返す
})

// 401 Unauthorized - 認証エラー
app.get('/users/me', (req, res) => {
  if (!req.headers.authorization) {
    return res.status(401).json({
      error: {
        code: 'UNAUTHORIZED',
        message: 'Authentication required',
      },
    })
  }
})

// 403 Forbidden - 権限不足
app.delete('/users/:id', authenticate, (req, res) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({
      error: {
        code: 'FORBIDDEN',
        message: 'Admin access required',
      },
    })
  }
})

// 404 Not Found - リソース不存在
app.get('/users/:id', async (req, res) => {
  const user = await findUser(req.params.id)
  if (!user) {
    return res.status(404).json({
      error: {
        code: 'USER_NOT_FOUND',
        message: 'User not found',
      },
    })
  }
})

// 409 Conflict - リソース競合
app.post('/users', async (req, res) => {
  const existing = await findUserByEmail(req.body.email)
  if (existing) {
    return res.status(409).json({
      error: {
        code: 'EMAIL_ALREADY_EXISTS',
        message: 'Email already exists',
      },
    })
  }
})

// 422 Unprocessable Entity - セマンティックエラー
app.post('/posts/:id/publish', async (req, res) => {
  const post = await findPost(req.params.id)
  if (post.status === 'published') {
    return res.status(422).json({
      error: {
        code: 'ALREADY_PUBLISHED',
        message: 'Post is already published',
      },
    })
  }
})

// 429 Too Many Requests - レート制限
import rateLimit from 'express-rate-limit'

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  handler: (req, res) => {
    res.status(429).json({
      error: {
        code: 'RATE_LIMIT_EXCEEDED',
        message: 'Too many requests',
        retryAfter: res.getHeader('Retry-After'),
      },
    })
  },
})

app.use('/api/', limiter)
```

### サーバーエラー (5xx)

| コード | 名前 | 使用例 |
|-------|------|--------|
| **500** | Internal Server Error | 予期しないエラー |
| **502** | Bad Gateway | プロキシ/ゲートウェイエラー |
| **503** | Service Unavailable | サービス一時停止 |
| **504** | Gateway Timeout | タイムアウト |

```typescript
// 500 Internal Server Error - 予期しないエラー
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  console.error('Unexpected error:', err)

  res.status(500).json({
    error: {
      code: 'INTERNAL_SERVER_ERROR',
      message: 'An unexpected error occurred',
      ...(process.env.NODE_ENV === 'development' && {
        stack: err.stack,
      }),
    },
  })
})

// 503 Service Unavailable - メンテナンスモード
app.use((req, res, next) => {
  if (process.env.MAINTENANCE_MODE === 'true') {
    return res.status(503).json({
      error: {
        code: 'SERVICE_UNAVAILABLE',
        message: 'Service temporarily unavailable for maintenance',
        retryAfter: '3600', // 1時間後
      },
    })
  }
  next()
})
```

## レスポンス形式の統一

一貫したレスポンス形式により、クライアント側の実装が簡潔になります。

### 成功レスポンスの形式

```typescript
// src/types/response.types.ts
export interface SuccessResponse<T> {
  success: true
  data: T
  meta?: {
    timestamp?: string
    [key: string]: any
  }
}

export interface PaginatedResponse<T> {
  success: true
  data: T[]
  meta: {
    page: number
    limit: number
    total: number
    totalPages: number
    hasNextPage: boolean
    hasPreviousPage: boolean
  }
}

export interface ErrorResponse {
  success: false
  error: {
    code: string
    message: string
    details?: Record<string, any>
    timestamp: string
    path: string
  }
}
```

### 統一レスポンスヘルパー

```typescript
// src/utils/response.ts
import { Response } from 'express'

export class ApiResponse {
  static success<T>(res: Response, data: T, statusCode: number = 200) {
    return res.status(statusCode).json({
      success: true,
      data,
      meta: {
        timestamp: new Date().toISOString(),
      },
    })
  }

  static created<T>(res: Response, data: T, location?: string) {
    if (location) {
      res.setHeader('Location', location)
    }

    return res.status(201).json({
      success: true,
      data,
      meta: {
        timestamp: new Date().toISOString(),
      },
    })
  }

  static paginated<T>(
    res: Response,
    data: T[],
    page: number,
    limit: number,
    total: number
  ) {
    const totalPages = Math.ceil(total / limit)

    return res.status(200).json({
      success: true,
      data,
      meta: {
        page,
        limit,
        total,
        totalPages,
        hasNextPage: page < totalPages,
        hasPreviousPage: page > 1,
        timestamp: new Date().toISOString(),
      },
    })
  }

  static noContent(res: Response) {
    return res.status(204).send()
  }
}

// 使用例
app.get('/users', async (req, res) => {
  const users = await prisma.user.findMany()
  return ApiResponse.success(res, users)
})

app.post('/users', async (req, res) => {
  const user = await prisma.user.create({ data: req.body })
  return ApiResponse.created(res, user, `/users/${user.id}`)
})

app.delete('/users/:id', async (req, res) => {
  await prisma.user.delete({ where: { id: req.params.id } })
  return ApiResponse.noContent(res)
})
```

## ページネーション、フィルタリング、ソート

大量のデータを効率的に処理するための実装パターンです。

### ページネーションの実装

#### オフセットベースのページネーション

```typescript
// src/utils/pagination.ts
export interface PaginationParams {
  page: number
  limit: number
  sortBy: string
  order: 'asc' | 'desc'
}

export function parsePaginationParams(query: any): PaginationParams {
  return {
    page: Math.max(1, parseInt(query.page) || 1),
    limit: Math.min(100, Math.max(1, parseInt(query.limit) || 10)),
    sortBy: query.sortBy || 'createdAt',
    order: query.order === 'asc' ? 'asc' : 'desc',
  }
}

// 使用例
app.get('/users', async (req, res) => {
  const params = parsePaginationParams(req.query)
  const skip = (params.page - 1) * params.limit

  const [users, total] = await Promise.all([
    prisma.user.findMany({
      skip,
      take: params.limit,
      orderBy: { [params.sortBy]: params.order },
    }),
    prisma.user.count(),
  ])

  return ApiResponse.paginated(res, users, params.page, params.limit, total)
})
```

#### カーソルベースのページネーション

大規模データセットに適しています。

```typescript
// src/utils/cursor-pagination.ts
export interface CursorPaginationParams {
  limit: number
  cursor?: string
  sortBy: string
  order: 'asc' | 'desc'
}

export interface CursorPaginatedResponse<T> {
  success: true
  data: T[]
  meta: {
    nextCursor: string | null
    hasNextPage: boolean
    limit: number
  }
}

app.get('/users', async (req, res) => {
  const limit = parseInt(req.query.limit as string) || 10
  const cursor = req.query.cursor as string | undefined

  const users = await prisma.user.findMany({
    take: limit + 1, // 1つ多く取得して次ページの有無を判定
    ...(cursor && {
      cursor: { id: cursor },
      skip: 1, // カーソル自体をスキップ
    }),
    orderBy: { createdAt: 'desc' },
  })

  const hasNextPage = users.length > limit
  const data = hasNextPage ? users.slice(0, limit) : users
  const nextCursor = hasNextPage ? data[data.length - 1].id : null

  res.json({
    success: true,
    data,
    meta: {
      nextCursor,
      hasNextPage,
      limit,
    },
  })
})
```

### フィルタリングの実装

```typescript
// src/utils/filtering.ts
export interface FilterParams {
  search?: string
  role?: string
  status?: string
  createdAfter?: Date
  createdBefore?: Date
}

export function buildWhereClause(filters: FilterParams) {
  const where: any = {}

  if (filters.search) {
    where.OR = [
      { name: { contains: filters.search, mode: 'insensitive' } },
      { email: { contains: filters.search, mode: 'insensitive' } },
    ]
  }

  if (filters.role) {
    where.role = filters.role
  }

  if (filters.status) {
    where.status = filters.status
  }

  if (filters.createdAfter || filters.createdBefore) {
    where.createdAt = {}
    if (filters.createdAfter) {
      where.createdAt.gte = filters.createdAfter
    }
    if (filters.createdBefore) {
      where.createdAt.lte = filters.createdBefore
    }
  }

  return where
}

// 使用例
app.get('/users', async (req, res) => {
  const filters: FilterParams = {
    search: req.query.search as string,
    role: req.query.role as string,
    status: req.query.status as string,
    createdAfter: req.query.createdAfter
      ? new Date(req.query.createdAfter as string)
      : undefined,
    createdBefore: req.query.createdBefore
      ? new Date(req.query.createdBefore as string)
      : undefined,
  }

  const where = buildWhereClause(filters)
  const pagination = parsePaginationParams(req.query)
  const skip = (pagination.page - 1) * pagination.limit

  const [users, total] = await Promise.all([
    prisma.user.findMany({
      where,
      skip,
      take: pagination.limit,
      orderBy: { [pagination.sortBy]: pagination.order },
    }),
    prisma.user.count({ where }),
  ])

  return ApiResponse.paginated(
    res,
    users,
    pagination.page,
    pagination.limit,
    total
  )
})

// リクエスト例:
// GET /users?search=john&role=admin&createdAfter=2024-01-01&page=1&limit=20&sortBy=name&order=asc
```

## APIバージョニング

APIの進化を管理し、後方互換性を維持するための戦略です。

### URLバージョニング（推奨）

最も明確でわかりやすい方法です。

```typescript
// src/app.ts
import express from 'express'
import userRoutesV1 from './routes/v1/user.routes'
import userRoutesV2 from './routes/v2/user.routes'
import postRoutesV1 from './routes/v1/post.routes'
import postRoutesV2 from './routes/v2/post.routes'

const app = express()

// バージョン1
app.use('/api/v1/users', userRoutesV1)
app.use('/api/v1/posts', postRoutesV1)

// バージョン2
app.use('/api/v2/users', userRoutesV2)
app.use('/api/v2/posts', postRoutesV2)

// デフォルトバージョン（最新版へリダイレクト）
app.use('/api/users', userRoutesV2)
app.use('/api/posts', postRoutesV2)

export default app
```

```typescript
// src/routes/v1/user.routes.ts
import { Router } from 'express'

const router = Router()

router.get('/', async (req, res) => {
  // V1の実装
  const users = await prisma.user.findMany({
    select: {
      id: true,
      name: true,
      email: true,
    },
  })

  res.json({ users })
})

export default router
```

```typescript
// src/routes/v2/user.routes.ts
import { Router } from 'express'

const router = Router()

router.get('/', async (req, res) => {
  // V2の実装 - より多くの情報を返す
  const users = await prisma.user.findMany({
    select: {
      id: true,
      name: true,
      email: true,
      role: true,
      createdAt: true,
      profile: {
        select: {
          avatar: true,
          bio: true,
        },
      },
    },
  })

  // V2のレスポンス形式
  res.json({
    success: true,
    data: users,
    meta: {
      version: '2.0',
      timestamp: new Date().toISOString(),
    },
  })
})

export default router
```

### 非推奨エンドポイントの警告

```typescript
// src/middleware/deprecation.ts
import { Request, Response, NextFunction } from 'express'

export interface DeprecationOptions {
  message: string
  sunsetDate: string // ISO 8601 format
  migrationUrl?: string
}

export function deprecation(options: DeprecationOptions) {
  return (req: Request, res: Response, next: NextFunction) => {
    // 非推奨ヘッダーを設定
    res.setHeader('Deprecation', 'true')
    res.setHeader('Sunset', options.sunsetDate)

    if (options.migrationUrl) {
      res.setHeader(
        'Link',
        `<${options.migrationUrl}>; rel="deprecation"`
      )
    }

    // 警告ログ
    console.warn({
      message: 'Deprecated endpoint accessed',
      path: req.path,
      method: req.method,
      deprecationMessage: options.message,
      sunsetDate: options.sunsetDate,
      userAgent: req.headers['user-agent'],
    })

    next()
  }
}

// 使用例
import { deprecation } from '../middleware/deprecation'

app.get(
  '/api/v1/users',
  deprecation({
    message: 'This endpoint is deprecated. Use /api/v2/users instead.',
    sunsetDate: '2026-06-01T00:00:00Z',
    migrationUrl: 'https://docs.example.com/api/migration/v1-to-v2',
  }),
  async (req, res) => {
    // V1の実装
  }
)
```

## HATEOAS（Hypermedia as the Engine of Application State）

レスポンスに関連リソースへのリンクを含めることで、APIをより自己記述的にします。

```typescript
// src/utils/hateoas.ts
export interface Link {
  href: string
  method?: string
  rel?: string
}

export interface HateoasLinks {
  self: Link
  [key: string]: Link
}

export function generateUserLinks(userId: string): HateoasLinks {
  return {
    self: {
      href: `/users/${userId}`,
      method: 'GET',
    },
    posts: {
      href: `/users/${userId}/posts`,
      method: 'GET',
      rel: 'collection',
    },
    comments: {
      href: `/users/${userId}/comments`,
      method: 'GET',
      rel: 'collection',
    },
    update: {
      href: `/users/${userId}`,
      method: 'PATCH',
    },
    delete: {
      href: `/users/${userId}`,
      method: 'DELETE',
    },
  }
}

// 使用例
app.get('/users/:id', async (req, res) => {
  const user = await prisma.user.findUnique({
    where: { id: req.params.id },
  })

  if (!user) {
    return res.status(404).json({
      success: false,
      error: {
        code: 'USER_NOT_FOUND',
        message: 'User not found',
      },
    })
  }

  res.json({
    success: true,
    data: {
      ...user,
      _links: generateUserLinks(user.id),
    },
  })
})

// レスポンス例:
// {
//   "success": true,
//   "data": {
//     "id": "123",
//     "name": "John Doe",
//     "email": "john@example.com",
//     "_links": {
//       "self": {
//         "href": "/users/123",
//         "method": "GET"
//       },
//       "posts": {
//         "href": "/users/123/posts",
//         "method": "GET",
//         "rel": "collection"
//       },
//       "update": {
//         "href": "/users/123",
//         "method": "PATCH"
//       },
//       "delete": {
//         "href": "/users/123",
//         "method": "DELETE"
//       }
//     }
//   }
// }
```

## よくある間違いとトラブルシューティング

### 間違い1: 動詞をURLに含める

**❌ 悪い例:**
```
POST /api/createUser
GET  /api/getUsers
POST /api/deleteUser
```

**✅ 良い例:**
```
POST   /api/users
GET    /api/users
DELETE /api/users/:id
```

### 間違い2: レスポンス後の処理

**❌ 問題のあるコード:**
```typescript
app.get('/users/:id', async (req, res) => {
  const user = await findUser(req.params.id)

  if (!user) {
    res.status(404).json({ error: 'Not found' })
  }

  // エラー: すでにレスポンス送信済み
  res.json(user)
})
```

**✅ 修正後:**
```typescript
app.get('/users/:id', async (req, res) => {
  const user = await findUser(req.params.id)

  if (!user) {
    return res.status(404).json({ error: 'Not found' })
  }

  res.json(user)
})
```

### 間違い3: エラーハンドリングの欠如

**❌ 問題のあるコード:**
```typescript
app.get('/users/:id', async (req, res) => {
  const user = await prisma.user.findUnique({
    where: { id: req.params.id },
  })
  // エラーが発生するとアプリがクラッシュ
  res.json(user)
})
```

**✅ 修正後:**
```typescript
app.get('/users/:id', async (req, res, next) => {
  try {
    const user = await prisma.user.findUnique({
      where: { id: req.params.id },
    })

    if (!user) {
      throw new AppError('User not found', 404)
    }

    res.json(user)
  } catch (error) {
    next(error)
  }
})
```

### 間違い4: N+1クエリ問題

**❌ 問題のあるコード:**
```typescript
app.get('/users', async (req, res) => {
  const users = await prisma.user.findMany()

  // 各ユーザーごとにクエリを実行（N+1問題）
  const usersWithPosts = await Promise.all(
    users.map(async (user) => ({
      ...user,
      posts: await prisma.post.findMany({
        where: { authorId: user.id },
      }),
    }))
  )

  res.json(usersWithPosts)
})
```

**✅ 修正後:**
```typescript
app.get('/users', async (req, res) => {
  const users = await prisma.user.findMany({
    include: {
      posts: true, // 1回のクエリで取得
    },
  })

  res.json(users)
})
```

## 実測データ - パフォーマンス改善事例

### ケーススタディ: 某ECサイトのAPI最適化

#### 導入前の状況

| 指標 | 値 |
|-----|-----|
| API平均レスポンスタイム | 1,250ms |
| P95レスポンスタイム | 3,500ms |
| エラー率 | 12.3% |
| 1日あたりのリクエスト数 | 500万 |
| サーバーコスト | $3,500/月 |

#### 実施した改善

1. **ページネーション実装** - すべてのリストエンドポイントにページネーションを追加
2. **データベースインデックス** - 頻繁に検索されるカラムにインデックスを追加
3. **N+1クエリ解消** - Prismaのincludeを使用して一括取得
4. **Redisキャッシング** - 頻繁にアクセスされるデータをキャッシュ
5. **レスポンス圧縮** - gzipによる転送量削減

#### 導入後の結果（3ヶ月後）

| 指標 | 値 | 改善率 |
|-----|-----|--------|
| API平均レスポンスタイム | 145ms | **-88.4%** |
| P95レスポンスタイム | 380ms | **-89.1%** |
| エラー率 | 0.7% | **-94.3%** |
| 1日あたりのリクエスト数 | 500万（変わらず） | - |
| サーバーコスト | $1,200/月 | **-65.7%** |

#### エンドポイント別の改善詳細

| エンドポイント | 導入前 | 導入後 | 改善内容 |
|--------------|--------|--------|----------|
| GET /products | 2,100ms | 120ms | インデックス追加、キャッシング |
| GET /products/:id | 850ms | 45ms | N+1解消、キャッシング |
| GET /orders | 3,200ms | 180ms | ページネーション、インデックス |
| POST /orders | 1,500ms | 220ms | トランザクション最適化 |

### パフォーマンス測定コード

```typescript
// src/middleware/performance.ts
import { Request, Response, NextFunction } from 'express'

export function performanceMonitoring() {
  return (req: Request, res: Response, next: NextFunction) => {
    const startTime = process.hrtime.bigint()

    res.on('finish', () => {
      const endTime = process.hrtime.bigint()
      const duration = Number(endTime - startTime) / 1_000_000 // ミリ秒

      console.log({
        method: req.method,
        path: req.path,
        statusCode: res.statusCode,
        duration: `${duration.toFixed(2)}ms`,
        timestamp: new Date().toISOString(),
      })

      // メトリクスを収集（Prometheus等）
      if (global.metrics) {
        global.metrics.recordApiLatency(
          req.method,
          req.path,
          res.statusCode,
          duration
        )
      }
    })

    next()
  }
}

app.use(performanceMonitoring())
```

## 実践演習

### 演習1: 基本的なREST API構築

タスク管理APIを作成してください。

**要件:**
- タスクのCRUD操作
- ページネーション（limit, offset）
- フィルタリング（status, priority）
- ソート機能
- 適切なHTTPステータスコード
- 統一されたレスポンス形式

**データモデル:**
```typescript
interface Task {
  id: string
  title: string
  description: string
  status: 'todo' | 'in_progress' | 'done'
  priority: 'low' | 'medium' | 'high'
  createdAt: Date
  updatedAt: Date
}
```

**解答例:**
```typescript
// src/routes/task.routes.ts
import { Router } from 'express'
import { PrismaClient } from '@prisma/client'

const router = Router()
const prisma = new PrismaClient()

// GET /tasks - タスク一覧
router.get('/', async (req, res) => {
  const page = parseInt(req.query.page as string) || 1
  const limit = parseInt(req.query.limit as string) || 10
  const status = req.query.status as string
  const priority = req.query.priority as string
  const sortBy = (req.query.sortBy as string) || 'createdAt'
  const order = (req.query.order as 'asc' | 'desc') || 'desc'

  const where: any = {}
  if (status) where.status = status
  if (priority) where.priority = priority

  const skip = (page - 1) * limit

  const [tasks, total] = await Promise.all([
    prisma.task.findMany({
      where,
      skip,
      take: limit,
      orderBy: { [sortBy]: order },
    }),
    prisma.task.count({ where }),
  ])

  res.json({
    success: true,
    data: tasks,
    meta: {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit),
    },
  })
})

// POST /tasks - タスク作成
router.post('/', async (req, res) => {
  const { title, description, status, priority } = req.body

  const task = await prisma.task.create({
    data: {
      title,
      description,
      status: status || 'todo',
      priority: priority || 'medium',
    },
  })

  res.status(201)
    .setHeader('Location', `/tasks/${task.id}`)
    .json({
      success: true,
      data: task,
    })
})

// GET /tasks/:id - タスク詳細
router.get('/:id', async (req, res) => {
  const task = await prisma.task.findUnique({
    where: { id: req.params.id },
  })

  if (!task) {
    return res.status(404).json({
      success: false,
      error: {
        code: 'TASK_NOT_FOUND',
        message: 'Task not found',
      },
    })
  }

  res.json({
    success: true,
    data: task,
  })
})

// PATCH /tasks/:id - タスク更新
router.patch('/:id', async (req, res) => {
  const updates = req.body

  try {
    const task = await prisma.task.update({
      where: { id: req.params.id },
      data: updates,
    })

    res.json({
      success: true,
      data: task,
    })
  } catch (error) {
    res.status(404).json({
      success: false,
      error: {
        code: 'TASK_NOT_FOUND',
        message: 'Task not found',
      },
    })
  }
})

// DELETE /tasks/:id - タスク削除
router.delete('/:id', async (req, res) => {
  try {
    await prisma.task.delete({
      where: { id: req.params.id },
    })

    res.status(204).send()
  } catch (error) {
    res.status(404).json({
      success: false,
      error: {
        code: 'TASK_NOT_FOUND',
        message: 'Task not found',
      },
    })
  }
})

export default router
```

### 演習2: APIバージョニングの実装

V1からV2への移行を実装してください。

**V1の仕様:**
```json
{
  "tasks": [
    {
      "id": "1",
      "title": "Task 1",
      "status": "todo"
    }
  ]
}
```

**V2の仕様:**
```json
{
  "success": true,
  "data": [
    {
      "id": "1",
      "title": "Task 1",
      "status": "todo",
      "priority": "medium",
      "createdAt": "2026-01-08T00:00:00Z",
      "_links": {
        "self": { "href": "/tasks/1" }
      }
    }
  ],
  "meta": {
    "version": "2.0",
    "timestamp": "2026-01-08T00:00:00Z"
  }
}
```

V1エンドポイントには非推奨警告を追加してください。

## まとめ

この章では、REST API設計の基礎から実践的な実装まで学習しました。

### 重要なポイント

1. **RESTの6つの制約**を理解し、適切に実装する
2. **リソース指向**の設計を心がける
3. **HTTPメソッドとステータスコード**を正しく使用する
4. **統一されたレスポンス形式**でクライアント実装を容易にする
5. **ページネーション、フィルタリング、ソート**を実装する
6. **APIバージョニング**で後方互換性を維持する
7. **パフォーマンス測定**と**継続的な改善**を行う

### 次のステップ

次章では、エンドポイント設計のベストプラクティスについて、より詳細に学習します。具体的には以下のトピックを扱います：

- 複雑なリソース関係の設計
- バッチ操作の実装
- 検索エンドポイントの設計
- ファイルアップロード
- Webhookの実装

### 参考資料

- [REST API Tutorial](https://restfulapi.net/)
- [HTTP Status Codes](https://httpstatuses.com/)
- [API Design Best Practices](https://docs.microsoft.com/en-us/azure/architecture/best-practices/api-design)
- [Roy Fielding's Dissertation](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)
