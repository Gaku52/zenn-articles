---
title: "Chapter 13: 実戦ケーススタディ Part 1 - E-commerce API基礎実装"
free: false
---

# Chapter 13: 実戦ケーススタディ Part 1 - E-commerce API基礎実装

このチャプターでは、これまで学んだすべてのベストプラクティスを統合し、実際のE-commerce APIを一から構築します。Part 1では、プロジェクトのセットアップから認証、商品管理、カート機能までを実装します。

## 仮想プロジェクト概要

### 構築するシステム（想定シナリオ）

**機能要件:**
- ユーザー管理（認証・認可）
- 商品管理（CRUD + 在庫管理）
- カート機能
- 注文処理（在庫確認、決済、注文確定）
- Stripe決済統合
- 管理者ダッシュボード

**非機能要件:**
- レスポンス時間: 95パーセンタイルで200ms以内
- 可用性: 99.9%以上
- スケーラビリティ: 1000リクエスト/秒に対応
- セキュリティ: OWASP Top 10対策

### 技術スタック

```
フレームワーク: Express.js + TypeScript
データベース: PostgreSQL 14+
ORM: Prisma 5.0
キャッシュ: Redis 7.0
決済: Stripe API
認証: JWT + bcrypt
ログ: Winston
監視: Sentry
API文書化: Swagger (OpenAPI 3.0)
テスト: Jest + Supertest
```

---

## プロジェクトセットアップ

### 1. プロジェクト初期化

```bash
# プロジェクト作成
mkdir ecommerce-api
cd ecommerce-api
npm init -y

# TypeScript + 必須パッケージ
npm install express cors helmet compression
npm install @prisma/client bcrypt jsonwebtoken stripe redis
npm install zod winston express-rate-limit

# 開発依存関係
npm install -D typescript @types/node @types/express
npm install -D @types/bcrypt @types/jsonwebtoken @types/cors
npm install -D prisma ts-node-dev jest @types/jest supertest
npm install -D @types/supertest eslint prettier
```

### 2. TypeScript設定

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

### 3. ディレクトリ構造

```
src/
├── config/
│   ├── database.ts          # DB接続設定
│   ├── redis.ts             # Redis設定
│   └── env.ts               # 環境変数管理
├── controllers/
│   ├── auth.controller.ts
│   ├── product.controller.ts
│   ├── cart.controller.ts
│   ├── order.controller.ts
│   └── payment.controller.ts
├── services/
│   ├── auth.service.ts
│   ├── product.service.ts
│   ├── cart.service.ts
│   ├── order.service.ts
│   ├── payment.service.ts
│   └── cache.service.ts
├── repositories/
│   ├── user.repository.ts
│   ├── product.repository.ts
│   ├── cart.repository.ts
│   └── order.repository.ts
├── middleware/
│   ├── auth.middleware.ts
│   ├── validate.middleware.ts
│   ├── error-handler.middleware.ts
│   └── rate-limit.middleware.ts
├── routes/
│   ├── auth.routes.ts
│   ├── product.routes.ts
│   ├── cart.routes.ts
│   ├── order.routes.ts
│   └── index.ts
├── validators/
│   ├── auth.validator.ts
│   ├── product.validator.ts
│   ├── cart.validator.ts
│   └── order.validator.ts
├── types/
│   ├── express.d.ts
│   └── index.ts
├── utils/
│   ├── logger.ts
│   ├── errors.ts
│   ├── password.ts
│   └── jwt.ts
├── app.ts
└── server.ts
```

---

## データベーススキーマ設計

### Prismaスキーマ

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ユーザー
model User {
  id            String    @id @default(uuid())
  email         String    @unique @db.VarChar(255)
  passwordHash  String    @map("password_hash") @db.VarChar(255)
  firstName     String    @map("first_name") @db.VarChar(100)
  lastName      String    @map("last_name") @db.VarChar(100)
  role          Role      @default(CUSTOMER)
  isActive      Boolean   @default(true) @map("is_active")
  emailVerified Boolean   @default(false) @map("email_verified")
  createdAt     DateTime  @default(now()) @map("created_at")
  updatedAt     DateTime  @updatedAt @map("updated_at")

  // リレーション
  cart          Cart?
  orders        Order[]
  refreshTokens RefreshToken[]

  @@index([email])
  @@index([role])
  @@map("users")
}

enum Role {
  CUSTOMER
  ADMIN
}

// リフレッシュトークン
model RefreshToken {
  id        String   @id @default(uuid())
  token     String   @unique @db.VarChar(500)
  userId    String   @map("user_id")
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  expiresAt DateTime @map("expires_at")
  revokedAt DateTime? @map("revoked_at")
  createdAt DateTime @default(now()) @map("created_at")

  @@index([userId])
  @@index([token])
  @@map("refresh_tokens")
}

// 商品カテゴリー
model Category {
  id          String    @id @default(uuid())
  name        String    @unique @db.VarChar(100)
  slug        String    @unique @db.VarChar(100)
  description String?   @db.Text
  parentId    String?   @map("parent_id")
  parent      Category? @relation("CategoryParent", fields: [parentId], references: [id], onDelete: SetNull)
  children    Category[] @relation("CategoryParent")
  products    Product[]
  createdAt   DateTime  @default(now()) @map("created_at")
  updatedAt   DateTime  @updatedAt @map("updated_at")

  @@index([slug])
  @@index([parentId])
  @@map("categories")
}

// 商品
model Product {
  id          String   @id @default(uuid())
  name        String   @db.VarChar(255)
  slug        String   @unique @db.VarChar(255)
  description String?  @db.Text
  price       Decimal  @db.Decimal(10, 2)
  compareAtPrice Decimal? @map("compare_at_price") @db.Decimal(10, 2)
  costPrice   Decimal? @map("cost_price") @db.Decimal(10, 2)
  sku         String   @unique @db.VarChar(100)
  barcode     String?  @db.VarChar(100)
  stock       Int      @default(0)
  lowStockThreshold Int @default(10) @map("low_stock_threshold")
  weight      Decimal? @db.Decimal(10, 2)
  dimensions  Json?    // {length, width, height}
  images      Json     // [url1, url2, ...]
  attributes  Json?    // {color: "red", size: "L"}
  categoryId  String   @map("category_id")
  category    Category @relation(fields: [categoryId], references: [id], onDelete: Restrict)
  isActive    Boolean  @default(true) @map("is_active")
  isFeatured  Boolean  @default(false) @map("is_featured")
  views       Int      @default(0)
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")

  // リレーション
  cartItems   CartItem[]
  orderItems  OrderItem[]

  @@index([slug])
  @@index([sku])
  @@index([categoryId])
  @@index([price])
  @@index([isActive])
  @@index([isFeatured])
  @@index([createdAt])
  @@map("products")
}

// カート
model Cart {
  id        String     @id @default(uuid())
  userId    String     @unique @map("user_id")
  user      User       @relation(fields: [userId], references: [id], onDelete: Cascade)
  items     CartItem[]
  createdAt DateTime   @default(now()) @map("created_at")
  updatedAt DateTime   @updatedAt @map("updated_at")

  @@map("carts")
}

// カートアイテム
model CartItem {
  id        String   @id @default(uuid())
  cartId    String   @map("cart_id")
  cart      Cart     @relation(fields: [cartId], references: [id], onDelete: Cascade)
  productId String   @map("product_id")
  product   Product  @relation(fields: [productId], references: [id], onDelete: Cascade)
  quantity  Int
  price     Decimal  @db.Decimal(10, 2) // 商品追加時の価格を保存
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  @@unique([cartId, productId])
  @@index([cartId])
  @@index([productId])
  @@map("cart_items")
}

// 注文
model Order {
  id              String      @id @default(uuid())
  orderNumber     String      @unique @map("order_number") @db.VarChar(50)
  userId          String      @map("user_id")
  user            User        @relation(fields: [userId], references: [id], onDelete: Restrict)
  status          OrderStatus @default(PENDING)
  subtotal        Decimal     @db.Decimal(10, 2)
  tax             Decimal     @default(0) @db.Decimal(10, 2)
  shippingCost    Decimal     @default(0) @map("shipping_cost") @db.Decimal(10, 2)
  discount        Decimal     @default(0) @db.Decimal(10, 2)
  total           Decimal     @db.Decimal(10, 2)
  currency        String      @default("USD") @db.VarChar(3)

  // 配送情報
  shippingAddress Json        @map("shipping_address")
  billingAddress  Json        @map("billing_address")

  // 決済情報
  paymentMethod   String      @map("payment_method") @db.VarChar(50)
  paymentStatus   PaymentStatus @default(PENDING) @map("payment_status")
  stripePaymentIntentId String? @map("stripe_payment_intent_id")

  // メタ情報
  notes           String?     @db.Text
  canceledAt      DateTime?   @map("canceled_at")
  cancelReason    String?     @map("cancel_reason") @db.Text

  createdAt       DateTime    @default(now()) @map("created_at")
  updatedAt       DateTime    @updatedAt @map("updated_at")

  // リレーション
  items           OrderItem[]

  @@index([userId])
  @@index([orderNumber])
  @@index([status])
  @@index([paymentStatus])
  @@index([createdAt])
  @@map("orders")
}

enum OrderStatus {
  PENDING
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELED
  REFUNDED
}

enum PaymentStatus {
  PENDING
  PROCESSING
  SUCCEEDED
  FAILED
  REFUNDED
}

// 注文アイテム
model OrderItem {
  id          String   @id @default(uuid())
  orderId     String   @map("order_id")
  order       Order    @relation(fields: [orderId], references: [id], onDelete: Cascade)
  productId   String   @map("product_id")
  product     Product  @relation(fields: [productId], references: [id], onDelete: Restrict)
  productSnapshot Json  @map("product_snapshot") // 注文時の商品情報を保存
  quantity    Int
  price       Decimal  @db.Decimal(10, 2)
  subtotal    Decimal  @db.Decimal(10, 2)
  createdAt   DateTime @default(now()) @map("created_at")

  @@index([orderId])
  @@index([productId])
  @@map("order_items")
}
```

### マイグレーション実行

```bash
# Prisma初期化
npx prisma init

# .envファイルに接続文字列を設定
# DATABASE_URL="postgresql://user:password@localhost:5432/ecommerce?schema=public"

# マイグレーション作成
npx prisma migrate dev --name init

# Prisma Clientを生成
npx prisma generate
```

---

## 認証システム実装

### JWT & パスワードハッシュ化ユーティリティ

```typescript
// src/utils/password.ts
import bcrypt from 'bcrypt'

const SALT_ROUNDS = 12

export async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS)
}

export async function comparePassword(
  password: string,
  hash: string
): Promise<boolean> {
  return bcrypt.compare(password, hash)
}
```

```typescript
// src/utils/jwt.ts
import jwt from 'jsonwebtoken'

const ACCESS_TOKEN_SECRET = process.env.ACCESS_TOKEN_SECRET!
const REFRESH_TOKEN_SECRET = process.env.REFRESH_TOKEN_SECRET!

export interface TokenPayload {
  userId: string
  email: string
  role: string
}

export function generateAccessToken(payload: TokenPayload): string {
  return jwt.sign(payload, ACCESS_TOKEN_SECRET, {
    expiresIn: '15m',
    issuer: 'ecommerce-api',
    audience: 'ecommerce-users',
  })
}

export function generateRefreshToken(payload: TokenPayload): string {
  return jwt.sign(payload, REFRESH_TOKEN_SECRET, {
    expiresIn: '7d',
    issuer: 'ecommerce-api',
    audience: 'ecommerce-users',
  })
}

export function verifyAccessToken(token: string): TokenPayload {
  try {
    return jwt.verify(token, ACCESS_TOKEN_SECRET, {
      issuer: 'ecommerce-api',
      audience: 'ecommerce-users',
    }) as TokenPayload
  } catch (error) {
    if (error instanceof jwt.TokenExpiredError) {
      throw new Error('TOKEN_EXPIRED')
    }
    throw new Error('INVALID_TOKEN')
  }
}

export function verifyRefreshToken(token: string): TokenPayload {
  try {
    return jwt.verify(token, REFRESH_TOKEN_SECRET, {
      issuer: 'ecommerce-api',
      audience: 'ecommerce-users',
    }) as TokenPayload
  } catch (error) {
    throw new Error('INVALID_REFRESH_TOKEN')
  }
}
```

### エラークラス定義

```typescript
// src/utils/errors.ts
export class AppError extends Error {
  constructor(
    public message: string,
    public statusCode: number = 500,
    public code: string = 'INTERNAL_ERROR',
    public details?: Record<string, any>
  ) {
    super(message)
    this.name = this.constructor.name
    Error.captureStackTrace(this, this.constructor)
  }
}

export class UnauthorizedError extends AppError {
  constructor(message: string = 'Unauthorized') {
    super(message, 401, 'UNAUTHORIZED')
  }
}

export class ForbiddenError extends AppError {
  constructor(message: string = 'Forbidden') {
    super(message, 403, 'FORBIDDEN')
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string) {
    super(`${resource} not found`, 404, 'NOT_FOUND')
  }
}

export class ValidationError extends AppError {
  constructor(message: string, details?: Record<string, any>) {
    super(message, 422, 'VALIDATION_ERROR', details)
  }
}

export class ConflictError extends AppError {
  constructor(message: string, details?: Record<string, any>) {
    super(message, 409, 'CONFLICT', details)
  }
}

export class InsufficientStockError extends AppError {
  constructor(productId: string, requested: number, available: number) {
    super(
      `Insufficient stock for product ${productId}`,
      409,
      'INSUFFICIENT_STOCK',
      { productId, requested, available }
    )
  }
}
```

### バリデーション

```typescript
// src/validators/auth.validator.ts
import { z } from 'zod'

export const registerSchema = z.object({
  email: z.string().email('Invalid email format'),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .regex(
      /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])/,
      'Password must contain uppercase, lowercase, number, and special character'
    ),
  firstName: z
    .string()
    .min(1, 'First name is required')
    .max(100, 'First name must be less than 100 characters'),
  lastName: z
    .string()
    .min(1, 'Last name is required')
    .max(100, 'Last name must be less than 100 characters'),
})

export const loginSchema = z.object({
  email: z.string().email('Invalid email format'),
  password: z.string().min(1, 'Password is required'),
})

export const refreshTokenSchema = z.object({
  refreshToken: z.string().min(1, 'Refresh token is required'),
})

export type RegisterDto = z.infer<typeof registerSchema>
export type LoginDto = z.infer<typeof loginSchema>
export type RefreshTokenDto = z.infer<typeof refreshTokenSchema>
```

### 認証サービス

```typescript
// src/services/auth.service.ts
import { PrismaClient } from '@prisma/client'
import { hashPassword, comparePassword } from '../utils/password'
import {
  generateAccessToken,
  generateRefreshToken,
  verifyRefreshToken,
  TokenPayload,
} from '../utils/jwt'
import { UnauthorizedError, ConflictError } from '../utils/errors'
import type { RegisterDto, LoginDto } from '../validators/auth.validator'

export class AuthService {
  constructor(private prisma: PrismaClient) {}

  async register(data: RegisterDto) {
    // メールアドレス重複チェック
    const existing = await this.prisma.user.findUnique({
      where: { email: data.email },
    })

    if (existing) {
      throw new ConflictError('Email already exists', { email: data.email })
    }

    // パスワードハッシュ化
    const passwordHash = await hashPassword(data.password)

    // ユーザー作成
    const user = await this.prisma.user.create({
      data: {
        email: data.email,
        passwordHash,
        firstName: data.firstName,
        lastName: data.lastName,
      },
      select: {
        id: true,
        email: true,
        firstName: true,
        lastName: true,
        role: true,
        createdAt: true,
      },
    })

    // トークン生成
    const payload: TokenPayload = {
      userId: user.id,
      email: user.email,
      role: user.role,
    }

    const accessToken = generateAccessToken(payload)
    const refreshToken = generateRefreshToken(payload)

    // リフレッシュトークンをDBに保存
    await this.prisma.refreshToken.create({
      data: {
        token: refreshToken,
        userId: user.id,
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000), // 7日後
      },
    })

    return {
      user,
      accessToken,
      refreshToken,
    }
  }

  async login(data: LoginDto) {
    // ユーザー検索
    const user = await this.prisma.user.findUnique({
      where: { email: data.email },
    })

    if (!user) {
      // タイミング攻撃対策: 同じ処理時間を確保
      await hashPassword('dummy-password')
      throw new UnauthorizedError('Invalid credentials')
    }

    // アクティブチェック
    if (!user.isActive) {
      throw new UnauthorizedError('Account is deactivated')
    }

    // パスワード検証
    const isValid = await comparePassword(data.password, user.passwordHash)

    if (!isValid) {
      throw new UnauthorizedError('Invalid credentials')
    }

    // トークン生成
    const payload: TokenPayload = {
      userId: user.id,
      email: user.email,
      role: user.role,
    }

    const accessToken = generateAccessToken(payload)
    const refreshToken = generateRefreshToken(payload)

    // リフレッシュトークンをDBに保存
    await this.prisma.refreshToken.create({
      data: {
        token: refreshToken,
        userId: user.id,
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
      },
    })

    return {
      user: {
        id: user.id,
        email: user.email,
        firstName: user.firstName,
        lastName: user.lastName,
        role: user.role,
      },
      accessToken,
      refreshToken,
    }
  }

  async refresh(refreshToken: string) {
    // トークン検証
    const payload = verifyRefreshToken(refreshToken)

    // DBに存在するか確認
    const storedToken = await this.prisma.refreshToken.findFirst({
      where: {
        token: refreshToken,
        userId: payload.userId,
        expiresAt: {
          gt: new Date(),
        },
        revokedAt: null,
      },
    })

    if (!storedToken) {
      throw new UnauthorizedError('Invalid refresh token')
    }

    // 新しいアクセストークンを発行
    const newAccessToken = generateAccessToken(payload)

    return {
      accessToken: newAccessToken,
    }
  }

  async logout(refreshToken: string) {
    // リフレッシュトークンを無効化
    await this.prisma.refreshToken.updateMany({
      where: {
        token: refreshToken,
      },
      data: {
        revokedAt: new Date(),
      },
    })
  }
}
```

### 認証ミドルウェア

```typescript
// src/middleware/auth.middleware.ts
import { Request, Response, NextFunction } from 'express'
import { verifyAccessToken, TokenPayload } from '../utils/jwt'
import { UnauthorizedError, ForbiddenError } from '../utils/errors'

declare global {
  namespace Express {
    interface Request {
      user?: TokenPayload
    }
  }
}

export function authenticate(
  req: Request,
  res: Response,
  next: NextFunction
) {
  try {
    const authHeader = req.headers.authorization

    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      throw new UnauthorizedError('No token provided')
    }

    const token = authHeader.substring(7)
    const payload = verifyAccessToken(token)

    req.user = payload
    next()
  } catch (error) {
    if (error instanceof Error && error.message === 'TOKEN_EXPIRED') {
      return next(new UnauthorizedError('Token expired'))
    }
    next(new UnauthorizedError('Invalid token'))
  }
}

export function authorize(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return next(new UnauthorizedError('Unauthorized'))
    }

    if (!roles.includes(req.user.role)) {
      return next(
        new ForbiddenError(
          `User role '${req.user.role}' is not authorized for this action`
        )
      )
    }

    next()
  }
}
```

### バリデーションミドルウェア

```typescript
// src/middleware/validate.middleware.ts
import { Request, Response, NextFunction } from 'express'
import { ZodSchema, ZodError } from 'zod'
import { ValidationError } from '../utils/errors'

export function validate(schema: ZodSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      schema.parse(req.body)
      next()
    } catch (error) {
      if (error instanceof ZodError) {
        const details = error.errors.reduce((acc, err) => {
          const path = err.path.join('.')
          acc[path] = err.message
          return acc
        }, {} as Record<string, string>)

        return next(new ValidationError('Validation failed', details))
      }
      next(error)
    }
  }
}
```

### 認証コントローラー

```typescript
// src/controllers/auth.controller.ts
import { Request, Response, NextFunction } from 'express'
import { AuthService } from '../services/auth.service'
import type { RegisterDto, LoginDto, RefreshTokenDto } from '../validators/auth.validator'

export class AuthController {
  constructor(private authService: AuthService) {}

  async register(
    req: Request<{}, {}, RegisterDto>,
    res: Response,
    next: NextFunction
  ) {
    try {
      const result = await this.authService.register(req.body)

      res.status(201).json({
        success: true,
        data: result,
      })
    } catch (error) {
      next(error)
    }
  }

  async login(
    req: Request<{}, {}, LoginDto>,
    res: Response,
    next: NextFunction
  ) {
    try {
      const result = await this.authService.login(req.body)

      res.json({
        success: true,
        data: result,
      })
    } catch (error) {
      next(error)
    }
  }

  async refresh(
    req: Request<{}, {}, RefreshTokenDto>,
    res: Response,
    next: NextFunction
  ) {
    try {
      const result = await this.authService.refresh(req.body.refreshToken)

      res.json({
        success: true,
        data: result,
      })
    } catch (error) {
      next(error)
    }
  }

  async logout(
    req: Request<{}, {}, RefreshTokenDto>,
    res: Response,
    next: NextFunction
  ) {
    try {
      await this.authService.logout(req.body.refreshToken)

      res.json({
        success: true,
        message: 'Logged out successfully',
      })
    } catch (error) {
      next(error)
    }
  }

  async getProfile(req: Request, res: Response, next: NextFunction) {
    try {
      // req.userは認証ミドルウェアで設定される
      res.json({
        success: true,
        data: req.user,
      })
    } catch (error) {
      next(error)
    }
  }
}
```

### 認証ルート

```typescript
// src/routes/auth.routes.ts
import { Router } from 'express'
import { PrismaClient } from '@prisma/client'
import { AuthController } from '../controllers/auth.controller'
import { AuthService } from '../services/auth.service'
import { authenticate } from '../middleware/auth.middleware'
import { validate } from '../middleware/validate.middleware'
import {
  registerSchema,
  loginSchema,
  refreshTokenSchema,
} from '../validators/auth.validator'

const router = Router()
const prisma = new PrismaClient()
const authService = new AuthService(prisma)
const authController = new AuthController(authService)

/**
 * @swagger
 * /auth/register:
 *   post:
 *     summary: Register a new user
 *     tags: [Auth]
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             required:
 *               - email
 *               - password
 *               - firstName
 *               - lastName
 *             properties:
 *               email:
 *                 type: string
 *                 format: email
 *               password:
 *                 type: string
 *                 minLength: 8
 *               firstName:
 *                 type: string
 *               lastName:
 *                 type: string
 *     responses:
 *       201:
 *         description: User registered successfully
 *       409:
 *         description: Email already exists
 */
router.post(
  '/register',
  validate(registerSchema),
  authController.register.bind(authController)
)

/**
 * @swagger
 * /auth/login:
 *   post:
 *     summary: Login user
 *     tags: [Auth]
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             required:
 *               - email
 *               - password
 *             properties:
 *               email:
 *                 type: string
 *                 format: email
 *               password:
 *                 type: string
 *     responses:
 *       200:
 *         description: Login successful
 *       401:
 *         description: Invalid credentials
 */
router.post(
  '/login',
  validate(loginSchema),
  authController.login.bind(authController)
)

router.post(
  '/refresh',
  validate(refreshTokenSchema),
  authController.refresh.bind(authController)
)

router.post(
  '/logout',
  validate(refreshTokenSchema),
  authController.logout.bind(authController)
)

router.get(
  '/profile',
  authenticate,
  authController.getProfile.bind(authController)
)

export default router
```

---

## 商品管理実装

### 商品バリデーション

```typescript
// src/validators/product.validator.ts
import { z } from 'zod'

export const createProductSchema = z.object({
  name: z.string().min(1).max(255),
  slug: z.string().min(1).max(255),
  description: z.string().optional(),
  price: z.number().positive(),
  compareAtPrice: z.number().positive().optional(),
  costPrice: z.number().positive().optional(),
  sku: z.string().min(1).max(100),
  barcode: z.string().max(100).optional(),
  stock: z.number().int().min(0).default(0),
  lowStockThreshold: z.number().int().min(0).default(10),
  weight: z.number().positive().optional(),
  dimensions: z
    .object({
      length: z.number().positive(),
      width: z.number().positive(),
      height: z.number().positive(),
    })
    .optional(),
  images: z.array(z.string().url()),
  attributes: z.record(z.any()).optional(),
  categoryId: z.string().uuid(),
  isActive: z.boolean().default(true),
  isFeatured: z.boolean().default(false),
})

export const updateProductSchema = createProductSchema.partial()

export const productQuerySchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().positive().max(100).default(20),
  categoryId: z.string().uuid().optional(),
  minPrice: z.coerce.number().positive().optional(),
  maxPrice: z.coerce.number().positive().optional(),
  search: z.string().optional(),
  isActive: z.coerce.boolean().optional(),
  isFeatured: z.coerce.boolean().optional(),
  sortBy: z.enum(['name', 'price', 'createdAt', 'views']).default('createdAt'),
  order: z.enum(['asc', 'desc']).default('desc'),
})

export type CreateProductDto = z.infer<typeof createProductSchema>
export type UpdateProductDto = z.infer<typeof updateProductSchema>
export type ProductQueryDto = z.infer<typeof productQuerySchema>
```

### キャッシュサービス

```typescript
// src/services/cache.service.ts
import { createClient, RedisClientType } from 'redis'

export class CacheService {
  private client: RedisClientType

  constructor() {
    this.client = createClient({
      url: process.env.REDIS_URL || 'redis://localhost:6379',
    })

    this.client.on('error', (err) => console.error('Redis error:', err))
    this.client.on('connect', () => console.log('Redis connected'))

    this.client.connect()
  }

  async get<T>(key: string): Promise<T | null> {
    const cached = await this.client.get(key)

    if (!cached) return null

    try {
      return JSON.parse(cached) as T
    } catch {
      return cached as T
    }
  }

  async set(key: string, value: any, ttlSeconds: number = 300): Promise<void> {
    const serialized = typeof value === 'string' ? value : JSON.stringify(value)
    await this.client.setEx(key, ttlSeconds, serialized)
  }

  async delete(key: string): Promise<void> {
    await this.client.del(key)
  }

  async invalidatePattern(pattern: string): Promise<void> {
    const keys = await this.client.keys(pattern)

    if (keys.length > 0) {
      await this.client.del(keys)
    }
  }

  async remember<T>(
    key: string,
    ttlSeconds: number,
    callback: () => Promise<T>
  ): Promise<T> {
    const cached = await this.get<T>(key)

    if (cached !== null) {
      return cached
    }

    const fresh = await callback()
    await this.set(key, fresh, ttlSeconds)

    return fresh
  }
}
```

### 商品リポジトリ

```typescript
// src/repositories/product.repository.ts
import { PrismaClient, Prisma } from '@prisma/client'
import type { ProductQueryDto } from '../validators/product.validator'

export class ProductRepository {
  constructor(private prisma: PrismaClient) {}

  async findMany(query: ProductQueryDto) {
    const { page, limit, categoryId, minPrice, maxPrice, search, isActive, isFeatured, sortBy, order } = query

    const where: Prisma.ProductWhereInput = {
      ...(categoryId && { categoryId }),
      ...(minPrice !== undefined || maxPrice !== undefined
        ? {
            price: {
              ...(minPrice !== undefined && { gte: minPrice }),
              ...(maxPrice !== undefined && { lte: maxPrice }),
            },
          }
        : {}),
      ...(search && {
        OR: [
          { name: { contains: search, mode: 'insensitive' } },
          { description: { contains: search, mode: 'insensitive' } },
          { sku: { contains: search, mode: 'insensitive' } },
        ],
      }),
      ...(isActive !== undefined && { isActive }),
      ...(isFeatured !== undefined && { isFeatured }),
    }

    return this.prisma.product.findMany({
      where,
      skip: (page - 1) * limit,
      take: limit,
      orderBy: { [sortBy]: order },
      include: {
        category: {
          select: {
            id: true,
            name: true,
            slug: true,
          },
        },
      },
    })
  }

  async count(query: ProductQueryDto) {
    const { categoryId, minPrice, maxPrice, search, isActive, isFeatured } = query

    const where: Prisma.ProductWhereInput = {
      ...(categoryId && { categoryId }),
      ...(minPrice !== undefined || maxPrice !== undefined
        ? {
            price: {
              ...(minPrice !== undefined && { gte: minPrice }),
              ...(maxPrice !== undefined && { lte: maxPrice }),
            },
          }
        : {}),
      ...(search && {
        OR: [
          { name: { contains: search, mode: 'insensitive' } },
          { description: { contains: search, mode: 'insensitive' } },
          { sku: { contains: search, mode: 'insensitive' } },
        ],
      }),
      ...(isActive !== undefined && { isActive }),
      ...(isFeatured !== undefined && { isFeatured }),
    }

    return this.prisma.product.count({ where })
  }

  async findById(id: string) {
    return this.prisma.product.findUnique({
      where: { id },
      include: {
        category: true,
      },
    })
  }

  async findBySlug(slug: string) {
    return this.prisma.product.findUnique({
      where: { slug },
      include: {
        category: true,
      },
    })
  }

  async create(data: Prisma.ProductCreateInput) {
    return this.prisma.product.create({
      data,
      include: {
        category: true,
      },
    })
  }

  async update(id: string, data: Prisma.ProductUpdateInput) {
    return this.prisma.product.update({
      where: { id },
      data,
      include: {
        category: true,
      },
    })
  }

  async delete(id: string) {
    await this.prisma.product.delete({
      where: { id },
    })
  }

  async incrementViews(id: string) {
    await this.prisma.product.update({
      where: { id },
      data: {
        views: {
          increment: 1,
        },
      },
    })
  }

  async checkStock(id: string): Promise<number> {
    const product = await this.prisma.product.findUnique({
      where: { id },
      select: { stock: true },
    })

    return product?.stock ?? 0
  }

  async decrementStock(id: string, quantity: number) {
    await this.prisma.product.update({
      where: { id },
      data: {
        stock: {
          decrement: quantity,
        },
      },
    })
  }
}
```

### 商品サービス

```typescript
// src/services/product.service.ts
import { ProductRepository } from '../repositories/product.repository'
import { CacheService } from './cache.service'
import { NotFoundError, ConflictError, ValidationError } from '../utils/errors'
import type { CreateProductDto, UpdateProductDto, ProductQueryDto } from '../validators/product.validator'

export class ProductService {
  constructor(
    private productRepository: ProductRepository,
    private cacheService: CacheService
  ) {}

  async findAll(query: ProductQueryDto) {
    const cacheKey = `products:${JSON.stringify(query)}`

    return this.cacheService.remember(
      cacheKey,
      300, // 5分
      async () => {
        const [products, total] = await Promise.all([
          this.productRepository.findMany(query),
          this.productRepository.count(query),
        ])

        return {
          products,
          total,
          page: query.page,
          limit: query.limit,
        }
      }
    )
  }

  async findById(id: string) {
    const cacheKey = `product:${id}`

    const product = await this.cacheService.remember(
      cacheKey,
      600, // 10分
      async () => {
        return this.productRepository.findById(id)
      }
    )

    if (!product) {
      throw new NotFoundError('Product')
    }

    // ビュー数を非同期で更新（レスポンスをブロックしない）
    this.productRepository.incrementViews(id).catch((err) => {
      console.error('Failed to increment views:', err)
    })

    return product
  }

  async findBySlug(slug: string) {
    const cacheKey = `product:slug:${slug}`

    const product = await this.cacheService.remember(
      cacheKey,
      600,
      async () => {
        return this.productRepository.findBySlug(slug)
      }
    )

    if (!product) {
      throw new NotFoundError('Product')
    }

    // ビュー数を非同期で更新
    this.productRepository.incrementViews(product.id).catch((err) => {
      console.error('Failed to increment views:', err)
    })

    return product
  }

  async create(data: CreateProductDto) {
    // バリデーション
    this.validateProductData(data)

    const product = await this.productRepository.create(data)

    // キャッシュ無効化
    await this.cacheService.invalidatePattern('products:*')

    return product
  }

  async update(id: string, data: UpdateProductDto) {
    const existingProduct = await this.productRepository.findById(id)

    if (!existingProduct) {
      throw new NotFoundError('Product')
    }

    // 在庫チェック
    if (data.stock !== undefined && data.stock < 0) {
      throw new ValidationError('Stock cannot be negative')
    }

    const product = await this.productRepository.update(id, data)

    // キャッシュ無効化
    await this.cacheService.delete(`product:${id}`)
    await this.cacheService.delete(`product:slug:${existingProduct.slug}`)
    await this.cacheService.invalidatePattern('products:*')

    return product
  }

  async delete(id: string) {
    const product = await this.productRepository.findById(id)

    if (!product) {
      throw new NotFoundError('Product')
    }

    await this.productRepository.delete(id)

    // キャッシュ無効化
    await this.cacheService.delete(`product:${id}`)
    await this.cacheService.delete(`product:slug:${product.slug}`)
    await this.cacheService.invalidatePattern('products:*')
  }

  private validateProductData(data: CreateProductDto): void {
    if (data.price < 0) {
      throw new ValidationError('Price cannot be negative')
    }

    if (data.stock < 0) {
      throw new ValidationError('Stock cannot be negative')
    }

    if (data.compareAtPrice && data.compareAtPrice < data.price) {
      throw new ValidationError('Compare at price must be greater than price')
    }

    if (!data.name || data.name.trim().length === 0) {
      throw new ValidationError('Product name is required')
    }

    if (data.name.length > 255) {
      throw new ValidationError('Product name must be less than 255 characters')
    }
  }
}
```

### 商品コントローラー

```typescript
// src/controllers/product.controller.ts
import { Request, Response, NextFunction } from 'express'
import { ProductService } from '../services/product.service'
import type { CreateProductDto, UpdateProductDto, ProductQueryDto } from '../validators/product.validator'

export class ProductController {
  constructor(private productService: ProductService) {}

  async getProducts(
    req: Request<{}, {}, {}, ProductQueryDto>,
    res: Response,
    next: NextFunction
  ) {
    try {
      const result = await this.productService.findAll(req.query)

      res.json({
        success: true,
        data: result.products,
        meta: {
          page: result.page,
          limit: result.limit,
          total: result.total,
          totalPages: Math.ceil(result.total / result.limit),
        },
      })
    } catch (error) {
      next(error)
    }
  }

  async getProductById(
    req: Request<{ id: string }>,
    res: Response,
    next: NextFunction
  ) {
    try {
      const product = await this.productService.findById(req.params.id)

      res.json({
        success: true,
        data: product,
      })
    } catch (error) {
      next(error)
    }
  }

  async getProductBySlug(
    req: Request<{ slug: string }>,
    res: Response,
    next: NextFunction
  ) {
    try {
      const product = await this.productService.findBySlug(req.params.slug)

      res.json({
        success: true,
        data: product,
      })
    } catch (error) {
      next(error)
    }
  }

  async createProduct(
    req: Request<{}, {}, CreateProductDto>,
    res: Response,
    next: NextFunction
  ) {
    try {
      const product = await this.productService.create(req.body)

      res.status(201).json({
        success: true,
        data: product,
      })
    } catch (error) {
      next(error)
    }
  }

  async updateProduct(
    req: Request<{ id: string }, {}, UpdateProductDto>,
    res: Response,
    next: NextFunction
  ) {
    try {
      const product = await this.productService.update(req.params.id, req.body)

      res.json({
        success: true,
        data: product,
      })
    } catch (error) {
      next(error)
    }
  }

  async deleteProduct(
    req: Request<{ id: string }>,
    res: Response,
    next: NextFunction
  ) {
    try {
      await this.productService.delete(req.params.id)

      res.status(204).send()
    } catch (error) {
      next(error)
    }
  }
}
```

---

## カート機能実装

### カートバリデーション

```typescript
// src/validators/cart.validator.ts
import { z } from 'zod'

export const addToCartSchema = z.object({
  productId: z.string().uuid(),
  quantity: z.number().int().positive().max(100),
})

export const updateCartItemSchema = z.object({
  quantity: z.number().int().positive().max(100),
})

export type AddToCartDto = z.infer<typeof addToCartSchema>
export type UpdateCartItemDto = z.infer<typeof updateCartItemSchema>
```

### カートリポジトリ

```typescript
// src/repositories/cart.repository.ts
import { PrismaClient } from '@prisma/client'

export class CartRepository {
  constructor(private prisma: PrismaClient) {}

  async findByUserId(userId: string) {
    return this.prisma.cart.findUnique({
      where: { userId },
      include: {
        items: {
          include: {
            product: {
              include: {
                category: true,
              },
            },
          },
        },
      },
    })
  }

  async createOrGet(userId: string) {
    let cart = await this.findByUserId(userId)

    if (!cart) {
      cart = await this.prisma.cart.create({
        data: {
          userId,
        },
        include: {
          items: {
            include: {
              product: {
                include: {
                  category: true,
                },
              },
            },
          },
        },
      })
    }

    return cart
  }

  async addItem(cartId: string, productId: string, quantity: number, price: number) {
    // 既存のアイテムをチェック
    const existingItem = await this.prisma.cartItem.findUnique({
      where: {
        cartId_productId: {
          cartId,
          productId,
        },
      },
    })

    if (existingItem) {
      // 既存アイテムの数量を更新
      return this.prisma.cartItem.update({
        where: {
          id: existingItem.id,
        },
        data: {
          quantity: existingItem.quantity + quantity,
        },
        include: {
          product: {
            include: {
              category: true,
            },
          },
        },
      })
    } else {
      // 新しいアイテムを追加
      return this.prisma.cartItem.create({
        data: {
          cartId,
          productId,
          quantity,
          price,
        },
        include: {
          product: {
            include: {
              category: true,
            },
          },
        },
      })
    }
  }

  async updateItem(cartItemId: string, quantity: number) {
    return this.prisma.cartItem.update({
      where: {
        id: cartItemId,
      },
      data: {
        quantity,
      },
      include: {
        product: {
          include: {
            category: true,
          },
        },
      },
    })
  }

  async removeItem(cartItemId: string) {
    await this.prisma.cartItem.delete({
      where: {
        id: cartItemId,
      },
    })
  }

  async clear(cartId: string) {
    await this.prisma.cartItem.deleteMany({
      where: {
        cartId,
      },
    })
  }
}
```

### カートサービス

```typescript
// src/services/cart.service.ts
import { CartRepository } from '../repositories/cart.repository'
import { ProductRepository } from '../repositories/product.repository'
import { NotFoundError, InsufficientStockError, ValidationError } from '../utils/errors'
import type { AddToCartDto, UpdateCartItemDto } from '../validators/cart.validator'

export class CartService {
  constructor(
    private cartRepository: CartRepository,
    private productRepository: ProductRepository
  ) {}

  async getCart(userId: string) {
    const cart = await this.cartRepository.findByUserId(userId)

    if (!cart) {
      // カートが存在しない場合は空のカートを返す
      return {
        id: null,
        userId,
        items: [],
        subtotal: 0,
        itemCount: 0,
      }
    }

    // 小計を計算
    const subtotal = cart.items.reduce((sum, item) => {
      return sum + Number(item.price) * item.quantity
    }, 0)

    const itemCount = cart.items.reduce((sum, item) => sum + item.quantity, 0)

    return {
      id: cart.id,
      userId: cart.userId,
      items: cart.items,
      subtotal,
      itemCount,
    }
  }

  async addToCart(userId: string, data: AddToCartDto) {
    // 商品存在チェック
    const product = await this.productRepository.findById(data.productId)

    if (!product) {
      throw new NotFoundError('Product')
    }

    // アクティブチェック
    if (!product.isActive) {
      throw new ValidationError('Product is not active')
    }

    // 在庫チェック
    if (product.stock < data.quantity) {
      throw new InsufficientStockError(product.id, data.quantity, product.stock)
    }

    // カートを取得または作成
    const cart = await this.cartRepository.createOrGet(userId)

    // カートに追加
    await this.cartRepository.addItem(cart.id, data.productId, data.quantity, Number(product.price))

    // 更新されたカートを返す
    return this.getCart(userId)
  }

  async updateCartItem(userId: string, cartItemId: string, data: UpdateCartItemDto) {
    // カート存在チェック
    const cart = await this.cartRepository.findByUserId(userId)

    if (!cart) {
      throw new NotFoundError('Cart')
    }

    // カートアイテム存在チェック
    const cartItem = cart.items.find((item) => item.id === cartItemId)

    if (!cartItem) {
      throw new NotFoundError('Cart item')
    }

    // 在庫チェック
    if (cartItem.product.stock < data.quantity) {
      throw new InsufficientStockError(
        cartItem.product.id,
        data.quantity,
        cartItem.product.stock
      )
    }

    // 数量更新
    await this.cartRepository.updateItem(cartItemId, data.quantity)

    // 更新されたカートを返す
    return this.getCart(userId)
  }

  async removeFromCart(userId: string, cartItemId: string) {
    // カート存在チェック
    const cart = await this.cartRepository.findByUserId(userId)

    if (!cart) {
      throw new NotFoundError('Cart')
    }

    // カートアイテム存在チェック
    const cartItem = cart.items.find((item) => item.id === cartItemId)

    if (!cartItem) {
      throw new NotFoundError('Cart item')
    }

    // アイテム削除
    await this.cartRepository.removeItem(cartItemId)

    // 更新されたカートを返す
    return this.getCart(userId)
  }

  async clearCart(userId: string) {
    // カート存在チェック
    const cart = await this.cartRepository.findByUserId(userId)

    if (!cart) {
      throw new NotFoundError('Cart')
    }

    // カートをクリア
    await this.cartRepository.clear(cart.id)

    return {
      id: cart.id,
      userId,
      items: [],
      subtotal: 0,
      itemCount: 0,
    }
  }
}
```

### カートコントローラー

```typescript
// src/controllers/cart.controller.ts
import { Request, Response, NextFunction } from 'express'
import { CartService } from '../services/cart.service'
import type { AddToCartDto, UpdateCartItemDto } from '../validators/cart.validator'

export class CartController {
  constructor(private cartService: CartService) {}

  async getCart(req: Request, res: Response, next: NextFunction) {
    try {
      const cart = await this.cartService.getCart(req.user!.userId)

      res.json({
        success: true,
        data: cart,
      })
    } catch (error) {
      next(error)
    }
  }

  async addToCart(
    req: Request<{}, {}, AddToCartDto>,
    res: Response,
    next: NextFunction
  ) {
    try {
      const cart = await this.cartService.addToCart(req.user!.userId, req.body)

      res.json({
        success: true,
        data: cart,
      })
    } catch (error) {
      next(error)
    }
  }

  async updateCartItem(
    req: Request<{ itemId: string }, {}, UpdateCartItemDto>,
    res: Response,
    next: NextFunction
  ) {
    try {
      const cart = await this.cartService.updateCartItem(
        req.user!.userId,
        req.params.itemId,
        req.body
      )

      res.json({
        success: true,
        data: cart,
      })
    } catch (error) {
      next(error)
    }
  }

  async removeFromCart(
    req: Request<{ itemId: string }>,
    res: Response,
    next: NextFunction
  ) {
    try {
      const cart = await this.cartService.removeFromCart(
        req.user!.userId,
        req.params.itemId
      )

      res.json({
        success: true,
        data: cart,
      })
    } catch (error) {
      next(error)
    }
  }

  async clearCart(req: Request, res: Response, next: NextFunction) {
    try {
      const cart = await this.cartService.clearCart(req.user!.userId)

      res.json({
        success: true,
        data: cart,
      })
    } catch (error) {
      next(error)
    }
  }
}
```

---

## Part 1のまとめ

このPart 1では、E-commerce APIの基礎部分を構築しました。

**実装した機能:**
- プロジェクトセットアップ（TypeScript、Prisma、Express）
- データベーススキーマ設計（Prisma）
- JWT認証・認可システム
- 商品管理（CRUD + キャッシング）
- カート機能

**採用したベストプラクティス:**
- レイヤードアーキテクチャ（Controller/Service/Repository）
- 型安全なバリデーション（Zod）
- キャッシング戦略（Redis）
- エラーハンドリング（カスタムエラークラス）
- セキュリティ対策（JWT、bcrypt、タイミング攻撃対策）

**次のPart 2では:**
- 注文処理（在庫管理、トランザクション）
- Stripe決済統合
- エラーハンドリングとログ管理
- レート制限
- Docker化とデプロイ
- モニタリング（Sentry）

これらの実装により、スケーラブルで保守性の高いE-commerce APIの完成形に近づきます。
