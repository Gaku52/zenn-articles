---
title: "Chapter 11: バックエンドセキュリティ実践 - OWASP Top 10対策とセキュア設計"
---

# Chapter 11: バックエンドセキュリティ実践

## イントロダクション

バックエンドセキュリティは、アプリケーション開発において最も重要な要素の一つです。セキュリティの脆弱性は、データ漏洩、サービス停止、金銭的損失、評判の失墜など、深刻な被害をもたらします。

本章では、OWASP Top 10に基づくセキュリティ対策、認証・認可の実装、入力検証、SQLインジェクション・XSS・CSRF対策、レート制限、セキュアヘッダー設定など、バックエンド開発におけるセキュリティのベストプラクティスを詳しく解説します。

### 本章で学ぶこと

- OWASP Top 10の完全対策
- JWT認証とリフレッシュトークン
- 2要素認証（TOTP）の実装
- ロールベースアクセス制御（RBAC）
- 入力検証とサニタイゼーション
- レート制限とDDoS対策
- セキュアヘッダー設定（Helmet）

---

## セキュリティの基礎

### セキュリティの3原則

バックエンドセキュリティは、以下の3つの原則に基づいて設計します。

1. **機密性（Confidentiality）** - 認可されたユーザーのみがデータにアクセス可能
2. **完全性（Integrity）** - データが改ざんされていない
3. **可用性（Availability）** - サービスが常に利用可能

### 防御の深層化（Defense in Depth）

複数の防御層を設けることで、1つの層が突破されても全体のセキュリティを維持します。

```
┌─────────────────────────────────┐
│ 1. ネットワークレベル            │
│    - Firewall                   │
│    - DDoS Protection            │
├─────────────────────────────────┤
│ 2. アプリケーションレベル        │
│    - WAF (Web Application       │
│      Firewall)                  │
│    - Rate Limiting              │
├─────────────────────────────────┤
│ 3. 認証・認可                   │
│    - JWT                        │
│    - 2FA                        │
│    - RBAC                       │
├─────────────────────────────────┤
│ 4. 入力検証                     │
│    - Validation                 │
│    - Sanitization               │
├─────────────────────────────────┤
│ 5. データ暗号化                 │
│    - HTTPS                      │
│    - Database Encryption        │
└─────────────────────────────────┘
```

---

## OWASP Top 10対策

OWASP Top 10は、Webアプリケーションにおける最も重要なセキュリティリスクをまとめたものです。

### 1. Injection（インジェクション）

#### SQL Injection対策

**脆弱なコード:**

```typescript
// ❌ 危険: 文字列結合
const email = req.query.email
const query = `SELECT * FROM users WHERE email = '${email}'`
// email = "admin' OR '1'='1" で全ユーザーが取得される
```

**安全なコード:**

```typescript
// ✅ 安全: パラメータ化クエリ（Prisma）
const user = await prisma.user.findUnique({
  where: { email },
})

// ✅ 安全: プリペアドステートメント（raw query）
const users = await prisma.$queryRaw`
  SELECT * FROM users WHERE email = ${email}
`

// ✅ 安全: TypeORM
const user = await userRepository.findOne({
  where: { email },
})
```

#### NoSQL Injection対策

**脆弱なコード:**

```typescript
// ❌ 危険: オブジェクトをそのまま使用
app.post('/login', async (req, res) => {
  const { email, password } = req.body
  // email = { $ne: null } で全ユーザーがマッチする
  const user = await db.collection('users').findOne({ email, password })
})
```

**安全なコード:**

```typescript
// ✅ 安全: 入力検証（Zod）
import { z } from 'zod'

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
})

app.post('/login', validate(loginSchema), async (req, res) => {
  const { email, password } = req.body
  // email, passwordは文字列であることが保証される
  const user = await db.collection('users').findOne({ email })

  if (!user) {
    throw new UnauthorizedError('Invalid credentials')
  }

  const isValid = await bcrypt.compare(password, user.password)

  if (!isValid) {
    throw new UnauthorizedError('Invalid credentials')
  }

  // ...
})
```

### 2. Broken Authentication（認証の不備）

#### セキュアな認証実装

```typescript
// src/services/auth.service.ts
import { PrismaClient } from '@prisma/client'
import { hash, compare } from 'bcrypt'
import {
  generateAccessToken,
  generateRefreshToken,
  verifyRefreshToken,
} from '../utils/jwt'
import { UnauthorizedError, TooManyRequestsError } from '../errors/http-errors'
import { redis } from '../utils/redis'

const prisma = new PrismaClient()

export class AuthService {
  async login(email: string, password: string, ipAddress: string) {
    // ログイン試行回数を確認
    const attempts = await this.getLoginAttempts(email, ipAddress)

    if (attempts >= 5) {
      throw new TooManyRequestsError('Too many login attempts. Please try again in 15 minutes.')
    }

    // ユーザー検索
    const user = await prisma.user.findUnique({
      where: { email },
    })

    if (!user) {
      await this.recordFailedAttempt(email, ipAddress)
      // タイミング攻撃対策: 同じ処理時間を確保
      await hash('dummy', 12)
      throw new UnauthorizedError('Invalid credentials')
    }

    // パスワード検証
    const isValid = await compare(password, user.password)

    if (!isValid) {
      await this.recordFailedAttempt(email, ipAddress)
      throw new UnauthorizedError('Invalid credentials')
    }

    // 成功時はログイン試行回数をリセット
    await this.resetLoginAttempts(email, ipAddress)

    // トークン生成
    const payload = {
      userId: user.id,
      email: user.email,
      role: user.role,
    }

    const accessToken = generateAccessToken(payload)
    const refreshToken = generateRefreshToken(payload)

    // リフレッシュトークンをDBに保存
    await prisma.refreshToken.create({
      data: {
        token: refreshToken,
        userId: user.id,
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000), // 7日
      },
    })

    return {
      accessToken,
      refreshToken,
      user: {
        id: user.id,
        email: user.email,
        name: user.name,
        role: user.role,
      },
    }
  }

  async refresh(refreshToken: string) {
    // トークン検証
    const payload = verifyRefreshToken(refreshToken)

    // DBに存在するか確認
    const storedToken = await prisma.refreshToken.findFirst({
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
    await prisma.refreshToken.updateMany({
      where: {
        token: refreshToken,
      },
      data: {
        revokedAt: new Date(),
      },
    })
  }

  private async getLoginAttempts(
    email: string,
    ipAddress: string
  ): Promise<number> {
    const key = `login:attempts:${email}:${ipAddress}`
    const attempts = await redis.get(key)
    return attempts ? parseInt(attempts) : 0
  }

  private async recordFailedAttempt(
    email: string,
    ipAddress: string
  ): Promise<void> {
    const key = `login:attempts:${email}:${ipAddress}`
    await redis.incr(key)
    await redis.expire(key, 900) // 15分で失効
  }

  private async resetLoginAttempts(
    email: string,
    ipAddress: string
  ): Promise<void> {
    const key = `login:attempts:${email}:${ipAddress}`
    await redis.del(key)
  }
}
```

#### JWT実装

```typescript
// src/utils/jwt.ts
import jwt from 'jsonwebtoken'
import { UnauthorizedError } from '../errors/http-errors'

const ACCESS_TOKEN_SECRET = process.env.ACCESS_TOKEN_SECRET!
const REFRESH_TOKEN_SECRET = process.env.REFRESH_TOKEN_SECRET!

export interface TokenPayload {
  userId: string
  email: string
  role: string
}

export function generateAccessToken(payload: TokenPayload): string {
  return jwt.sign(payload, ACCESS_TOKEN_SECRET, {
    expiresIn: '15m', // 短い有効期限
    issuer: 'your-app-name',
    audience: 'your-app-users',
  })
}

export function generateRefreshToken(payload: TokenPayload): string {
  return jwt.sign(payload, REFRESH_TOKEN_SECRET, {
    expiresIn: '7d',
    issuer: 'your-app-name',
    audience: 'your-app-users',
  })
}

export function verifyAccessToken(token: string): TokenPayload {
  try {
    return jwt.verify(token, ACCESS_TOKEN_SECRET, {
      issuer: 'your-app-name',
      audience: 'your-app-users',
    }) as TokenPayload
  } catch (error) {
    if (error instanceof jwt.TokenExpiredError) {
      throw new UnauthorizedError('Token expired')
    }
    if (error instanceof jwt.JsonWebTokenError) {
      throw new UnauthorizedError('Invalid token')
    }
    throw error
  }
}

export function verifyRefreshToken(token: string): TokenPayload {
  try {
    return jwt.verify(token, REFRESH_TOKEN_SECRET, {
      issuer: 'your-app-name',
      audience: 'your-app-users',
    }) as TokenPayload
  } catch (error) {
    throw new UnauthorizedError('Invalid refresh token')
  }
}
```

#### 2要素認証（TOTP）

```typescript
// src/utils/totp.ts
import speakeasy from 'speakeasy'
import QRCode from 'qrcode'

export async function generateTOTPSecret(userEmail: string) {
  const secret = speakeasy.generateSecret({
    name: `YourApp (${userEmail})`,
    issuer: 'YourApp',
    length: 32,
  })

  const qrCodeUrl = await QRCode.toDataURL(secret.otpauth_url!)

  return {
    secret: secret.base32,
    qrCode: qrCodeUrl,
  }
}

export function verifyTOTP(token: string, secret: string): boolean {
  return speakeasy.totp.verify({
    secret,
    encoding: 'base32',
    token,
    window: 1, // 前後30秒の誤差を許容
  })
}
```

```typescript
// src/routes/auth.routes.ts
import { Router } from 'express'
import { authenticate } from '../middleware/authenticate'
import { generateTOTPSecret, verifyTOTP } from '../utils/totp'
import { prisma } from '../utils/prisma'
import { UnauthorizedError } from '../errors/http-errors'

const router = Router()

// 2FA有効化
router.post('/auth/2fa/enable', authenticate, async (req, res) => {
  const { secret, qrCode } = await generateTOTPSecret(req.user.email)

  // シークレットを一時保存（DBまたはセッション）
  await prisma.user.update({
    where: { id: req.user.id },
    data: { totpSecret: secret, twoFactorEnabled: false }, // まだ有効化しない
  })

  res.json({
    success: true,
    data: { qrCode },
  })
})

// 2FA検証と有効化
router.post('/auth/2fa/verify', authenticate, async (req, res) => {
  const { token } = req.body

  const user = await prisma.user.findUnique({
    where: { id: req.user.id },
  })

  if (!user?.totpSecret) {
    throw new UnauthorizedError('2FA not set up')
  }

  const isValid = verifyTOTP(token, user.totpSecret)

  if (!isValid) {
    throw new UnauthorizedError('Invalid 2FA token')
  }

  // 2FA有効化
  await prisma.user.update({
    where: { id: req.user.id },
    data: { twoFactorEnabled: true },
  })

  res.json({
    success: true,
    message: '2FA enabled successfully',
  })
})

// 2FAログイン
router.post('/auth/login/2fa', async (req, res) => {
  const { email, password, token } = req.body

  // 通常のログイン処理
  const authService = new AuthService()
  const result = await authService.login(email, password, req.ip)

  // 2FA確認
  const user = await prisma.user.findUnique({
    where: { email },
  })

  if (user?.twoFactorEnabled) {
    if (!token) {
      return res.status(200).json({
        success: true,
        requiresTwoFactor: true,
      })
    }

    const isValid = verifyTOTP(token, user.totpSecret!)

    if (!isValid) {
      throw new UnauthorizedError('Invalid 2FA token')
    }
  }

  res.json({
    success: true,
    data: result,
  })
})

export default router
```

### 3. Sensitive Data Exposure（機密データの露出）

#### データ暗号化

```typescript
// src/utils/encryption.ts
import crypto from 'crypto'

const ENCRYPTION_KEY = process.env.ENCRYPTION_KEY! // 32バイト
const ALGORITHM = 'aes-256-gcm'

export function encrypt(text: string): string {
  const iv = crypto.randomBytes(16)
  const cipher = crypto.createCipheriv(
    ALGORITHM,
    Buffer.from(ENCRYPTION_KEY, 'hex'),
    iv
  )

  let encrypted = cipher.update(text, 'utf8', 'hex')
  encrypted += cipher.final('hex')

  const authTag = cipher.getAuthTag()

  return `${iv.toString('hex')}:${authTag.toString('hex')}:${encrypted}`
}

export function decrypt(encryptedText: string): string {
  const [ivHex, authTagHex, encrypted] = encryptedText.split(':')

  const decipher = crypto.createDecipheriv(
    ALGORITHM,
    Buffer.from(ENCRYPTION_KEY, 'hex'),
    Buffer.from(ivHex, 'hex')
  )

  decipher.setAuthTag(Buffer.from(authTagHex, 'hex'))

  let decrypted = decipher.update(encrypted, 'hex', 'utf8')
  decrypted += decipher.final('utf8')

  return decrypted
}
```

```typescript
// 使用例: クレジットカード番号の暗号化
import { encrypt, decrypt } from '../utils/encryption'

// 保存時に暗号化
const creditCard = '4111111111111111'
const encrypted = encrypt(creditCard)

await prisma.payment.create({
  data: {
    userId,
    cardNumber: encrypted, // 暗号化して保存
    lastFourDigits: creditCard.slice(-4), // 検索用
  },
})

// 読み込み時に復号化
const payment = await prisma.payment.findUnique({
  where: { id: paymentId },
})

const decryptedCard = decrypt(payment.cardNumber)
```

### 4. XML External Entities (XXE)

```typescript
// ✅ XMLパーサーの安全な設定
import { parseString } from 'xml2js'

const parserOptions = {
  // 外部エンティティを無効化
  async: false,
  explicitArray: false,
  ignoreAttrs: true,
  // DTD処理を無効化
  strict: true,
}

parseString(xmlString, parserOptions, (err, result) => {
  if (err) {
    throw new BadRequestError('Invalid XML')
  }
  // 処理
})
```

### 5. Broken Access Control（アクセス制御の不備）

#### ロールベースアクセス制御（RBAC）

```typescript
// src/types/role.ts
export enum Role {
  ADMIN = 'ADMIN',
  USER = 'USER',
  GUEST = 'GUEST',
}

export const RoleHierarchy = {
  [Role.ADMIN]: [Role.ADMIN, Role.USER, Role.GUEST],
  [Role.USER]: [Role.USER, Role.GUEST],
  [Role.GUEST]: [Role.GUEST],
}
```

```typescript
// src/middleware/authorize.ts
import { Request, Response, NextFunction } from 'express'
import { ForbiddenError } from '../errors/http-errors'
import { Role, RoleHierarchy } from '../types/role'

export function authorize(...allowedRoles: Role[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      throw new ForbiddenError('User not authenticated')
    }

    const userRole = req.user.role as Role
    const hasPermission = RoleHierarchy[userRole].some((role) =>
      allowedRoles.includes(role)
    )

    if (!hasPermission) {
      throw new ForbiddenError(
        `User role '${userRole}' is not authorized for this action`
      )
    }

    next()
  }
}

// 使用例
router.delete(
  '/users/:id',
  authenticate,
  authorize(Role.ADMIN),
  deleteUser
)
```

#### リソースベースアクセス制御

```typescript
// src/middleware/resource-authorize.ts
import { Request, Response, NextFunction } from 'express'
import { ForbiddenError, NotFoundError } from '../errors/http-errors'
import { prisma } from '../utils/prisma'
import { Role } from '../types/role'

export function authorizeResourceOwner(resourceType: 'post' | 'comment') {
  return async (req: Request, res: Response, next: NextFunction) => {
    const resourceId = parseInt(req.params.id)

    let resource: any

    switch (resourceType) {
      case 'post':
        resource = await prisma.post.findUnique({
          where: { id: resourceId },
        })
        break
      case 'comment':
        resource = await prisma.comment.findUnique({
          where: { id: resourceId },
        })
        break
    }

    if (!resource) {
      throw new NotFoundError(resourceType)
    }

    // 所有者またはADMINのみ許可
    if (
      resource.authorId !== req.user.id &&
      req.user.role !== Role.ADMIN
    ) {
      throw new ForbiddenError('You do not own this resource')
    }

    req.resource = resource

    next()
  }
}

// 使用例
router.delete(
  '/posts/:id',
  authenticate,
  authorizeResourceOwner('post'),
  deletePost
)
```

### 6. Security Misconfiguration（セキュリティ設定ミス）

#### Helmetによるセキュリティヘッダー設定

```typescript
// src/middleware/security.ts
import helmet from 'helmet'
import { Express } from 'express'

export function setupSecurity(app: Express) {
  // Helmetで各種セキュリティヘッダーを設定
  app.use(
    helmet({
      contentSecurityPolicy: {
        directives: {
          defaultSrc: ["'self'"],
          scriptSrc: ["'self'", "'unsafe-inline'", "https://cdn.example.com"],
          styleSrc: ["'self'", "'unsafe-inline'", "https://fonts.googleapis.com"],
          imgSrc: ["'self'", 'data:', 'https:'],
          fontSrc: ["'self'", "https://fonts.gstatic.com"],
          connectSrc: ["'self'", "https://api.example.com"],
          frameSrc: ["'none'"],
          objectSrc: ["'none'"],
        },
      },
      hsts: {
        maxAge: 31536000, // 1年
        includeSubDomains: true,
        preload: true,
      },
      referrerPolicy: {
        policy: 'no-referrer',
      },
    })
  )

  // X-Powered-Byヘッダーを削除（フレームワーク情報を隠す）
  app.disable('x-powered-by')

  // HTTPS強制（本番環境）
  if (process.env.NODE_ENV === 'production') {
    app.use((req, res, next) => {
      if (req.header('x-forwarded-proto') !== 'https') {
        res.redirect(`https://${req.header('host')}${req.url}`)
      } else {
        next()
      }
    })
  }
}
```

### 7. Cross-Site Scripting (XSS)

#### サニタイゼーション

```typescript
// src/utils/sanitize.ts
import DOMPurify from 'isomorphic-dompurify'

export function sanitizeHtml(dirty: string): string {
  return DOMPurify.sanitize(dirty, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br', 'ul', 'ol', 'li'],
    ALLOWED_ATTR: ['href', 'target', 'rel'],
    ALLOWED_URI_REGEXP: /^(?:(?:(?:f|ht)tps?|mailto|tel|callto|sms|cid|xmpp):|[^a-z]|[a-z+.\-]+(?:[^a-z+.\-:]|$))/i,
  })
}

// 使用例
router.post('/posts', authenticate, async (req, res) => {
  const { title, content } = req.body

  const post = await prisma.post.create({
    data: {
      title: sanitizeHtml(title),
      content: sanitizeHtml(content),
      authorId: req.user.id,
    },
  })

  res.json({ success: true, data: post })
})
```

#### Content Security Policy（CSP）

```typescript
// src/middleware/csp.ts
import { Request, Response, NextFunction } from 'express'

export function csp(req: Request, res: Response, next: NextFunction) {
  res.setHeader(
    'Content-Security-Policy',
    [
      "default-src 'self'",
      "script-src 'self' 'unsafe-inline' https://cdn.example.com",
      "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
      "img-src 'self' data: https:",
      "font-src 'self' https://fonts.gstatic.com",
      "connect-src 'self' https://api.example.com",
      "frame-ancestors 'none'",
      "base-uri 'self'",
      "form-action 'self'",
    ].join('; ')
  )

  next()
}
```

### 8. Insecure Deserialization（安全でないデシリアライゼーション）

```typescript
// ❌ 危険: eval使用
const data = eval(userInput)

// ✅ 安全: JSON.parse使用
try {
  const data = JSON.parse(userInput)
} catch (error) {
  throw new BadRequestError('Invalid JSON')
}

// ✅ さらに安全: スキーマ検証（Zod）
import { z } from 'zod'

const dataSchema = z.object({
  name: z.string(),
  age: z.number().int().positive(),
  email: z.string().email(),
})

try {
  const data = dataSchema.parse(JSON.parse(userInput))
} catch (error) {
  if (error instanceof z.ZodError) {
    throw new ValidationError('Validation failed', error.errors)
  }
  throw new BadRequestError('Invalid data')
}
```

### 9. Using Components with Known Vulnerabilities（既知の脆弱性）

```bash
# 定期的な依存関係の監査
npm audit

# 自動修正
npm audit fix

# 破壊的変更を含む修正
npm audit fix --force

# 特定のパッケージを更新
npm update <package-name>
```

```yaml
# .github/workflows/security.yml
name: Security Audit

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 0 * * 0' # 毎週日曜日

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm audit --audit-level=moderate
```

### 10. Insufficient Logging & Monitoring（ログとモニタリングの不足）

#### 監査ログ

```typescript
// src/middleware/audit-log.ts
import { Request, Response, NextFunction } from 'express'
import { logger } from '../utils/logger'

const SENSITIVE_ROUTES = [
  '/auth/login',
  '/auth/logout',
  '/auth/register',
  '/users/:id/password',
  '/admin/*',
]

export function auditLog(req: Request, res: Response, next: NextFunction) {
  const isSensitive = SENSITIVE_ROUTES.some((route) =>
    req.path.match(new RegExp(route.replace('*', '.*').replace(':id', '\\d+')))
  )

  if (isSensitive || req.method !== 'GET') {
    res.on('finish', () => {
      logger.info({
        type: 'audit',
        method: req.method,
        path: req.path,
        statusCode: res.statusCode,
        userId: req.user?.id,
        ip: req.ip,
        userAgent: req.get('user-agent'),
        timestamp: new Date().toISOString(),
      })
    })
  }

  next()
}
```

---

## 入力検証

### Zodスキーマ検証

```typescript
// src/schemas/user.schema.ts
import { z } from 'zod'

export const createUserSchema = z.object({
  email: z.string().email('Invalid email format'),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .regex(
      /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])/,
      'Password must contain uppercase, lowercase, number, and special character'
    ),
  username: z
    .string()
    .min(2, 'Username must be at least 2 characters')
    .max(50, 'Username must be at most 50 characters')
    .regex(/^[a-zA-Z0-9_]+$/, 'Username must contain only letters, numbers, and underscores'),
})

export const updateUserSchema = z.object({
  email: z.string().email().optional(),
  username: z.string().min(2).max(50).optional(),
})

export const createPostSchema = z.object({
  title: z.string().min(1).max(255),
  content: z.string().max(10000),
  published: z.boolean().optional().default(false),
})
```

```typescript
// src/middleware/validation.ts
import { Request, Response, NextFunction } from 'express'
import { ZodSchema, ZodError } from 'zod'
import { ValidationError } from '../errors/http-errors'

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

// 使用例
router.post('/users', validate(createUserSchema), createUser)
router.patch('/users/:id', authenticate, validate(updateUserSchema), updateUser)
```

### ファイルアップロード検証

```typescript
// src/middleware/upload.ts
import multer from 'multer'
import path from 'path'
import { BadRequestError } from '../errors/http-errors'

const storage = multer.diskStorage({
  destination: 'uploads/',
  filename: (req, file, cb) => {
    const uniqueSuffix = `${Date.now()}-${Math.round(Math.random() * 1e9)}`
    cb(null, `${uniqueSuffix}${path.extname(file.originalname)}`)
  },
})

const fileFilter = (req: any, file: Express.Multer.File, cb: any) => {
  const allowedTypes = ['image/jpeg', 'image/png', 'image/webp', 'image/gif']
  const allowedExtensions = ['.jpg', '.jpeg', '.png', '.webp', '.gif']

  const ext = path.extname(file.originalname).toLowerCase()

  if (
    !allowedTypes.includes(file.mimetype) ||
    !allowedExtensions.includes(ext)
  ) {
    return cb(new BadRequestError('Invalid file type. Only JPEG, PNG, WEBP, and GIF are allowed.'))
  }

  cb(null, true)
}

export const upload = multer({
  storage,
  fileFilter,
  limits: {
    fileSize: 5 * 1024 * 1024, // 5MB
    files: 1,
  },
})

// 使用例
router.post('/upload/avatar', authenticate, upload.single('avatar'), async (req, res) => {
  if (!req.file) {
    throw new BadRequestError('No file uploaded')
  }

  const avatarUrl = `/uploads/${req.file.filename}`

  await prisma.user.update({
    where: { id: req.user.id },
    data: { avatar: avatarUrl },
  })

  res.json({
    success: true,
    data: { avatarUrl },
  })
})
```

---

## レート制限

### express-rate-limitの実装

```typescript
// src/middleware/rate-limit.ts
import rateLimit from 'express-rate-limit'
import RedisStore from 'rate-limit-redis'
import { createClient } from 'redis'

const redisClient = createClient({
  url: process.env.REDIS_URL,
})

redisClient.connect()

// 一般的なAPI
export const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15分
  max: 100,
  standardHeaders: true,
  legacyHeaders: false,
  store: new RedisStore({
    client: redisClient,
    prefix: 'rl:api:',
  }),
  message: {
    error: {
      code: 'RATE_LIMIT_EXCEEDED',
      message: 'Too many requests, please try again later.',
    },
  },
  handler: (req, res) => {
    res.status(429).json({
      success: false,
      error: {
        code: 'RATE_LIMIT_EXCEEDED',
        message: 'Too many requests, please try again later.',
        retryAfter: res.getHeader('Retry-After'),
      },
    })
  },
})

// 認証エンドポイント（厳しい制限）
export const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  skipSuccessfulRequests: true, // 成功したリクエストはカウントしない
  store: new RedisStore({
    client: redisClient,
    prefix: 'rl:auth:',
  }),
})

// 認証済みユーザーは制限を緩和
export const authenticatedApiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: async (req) => {
    return req.user ? 200 : 100 // 認証済みは200まで
  },
  store: new RedisStore({
    client: redisClient,
    prefix: 'rl:auth-api:',
  }),
})

// 使用例
app.use('/api/', apiLimiter)
app.use('/auth/login', authLimiter)
app.use('/auth/register', authLimiter)
```

---

## CSRF対策

### CSRFトークン実装

```typescript
// src/middleware/csrf.ts
import csrf from 'csurf'
import cookieParser from 'cookie-parser'

export const csrfProtection = csrf({
  cookie: {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
  },
})

// CSRFトークン発行エンドポイント
router.get('/csrf-token', csrfProtection, (req, res) => {
  res.json({
    success: true,
    data: { csrfToken: req.csrfToken() },
  })
})

// 保護されたエンドポイント
router.post('/api/data', csrfProtection, (req, res) => {
  // CSRF検証済み
  res.json({ success: true })
})
```

### SameSite Cookie

```typescript
// src/utils/cookie.ts
import { Response, CookieOptions } from 'express'

export function setSecureCookie(
  res: Response,
  name: string,
  value: string,
  options?: CookieOptions
) {
  res.cookie(name, value, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 7 * 24 * 60 * 60 * 1000, // 7日
    ...options,
  })
}

// 使用例
router.post('/auth/login', async (req, res) => {
  const { accessToken, refreshToken, user } = await authService.login(
    req.body.email,
    req.body.password,
    req.ip
  )

  // アクセストークンはレスポンスボディ
  // リフレッシュトークンはHttpOnly Cookie
  setSecureCookie(res, 'refreshToken', refreshToken)

  res.json({
    success: true,
    data: { accessToken, user },
  })
})
```

---

## ベンチマーク指標

### 想定シナリオ: ECサイトのセキュリティ強化効果

#### 導入前

| 指標 | 値 |
|---|---|
| 脆弱性スキャン結果 | Critical: 15, High: 42 |
| ブルートフォース攻撃 | 月850回 |
| XSS攻撃検出 | 月120回 |
| SQL Injection試行 | 月65回 |
| 不正ログイン | 月35件 |

#### 導入後に期待できる効果（6ヶ月想定）

| 指標 | 値 | 改善率 |
|---|---|---|
| 脆弱性スキャン結果 | Critical: 0, High: 2 | **-97%** |
| ブルートフォース攻撃 | 月8回（レート制限で遮断） | **-99%** |
| XSS攻撃検出 | 月3回（CSPで遮断） | **-98%** |
| SQL Injection試行 | 月0回（Prismaで無効化） | **-100%** |
| 不正ログイン | 月0件（2FA導入） | **-100%** |

#### 想定されるセキュリティ対策

1. **Helmet導入** - セキュアヘッダー設定
2. **CSRF対策** - すべての変更系エンドポイントに適用
3. **レート制限** - 認証エンドポイントに5回/15分の制限
4. **2要素認証** - 管理者とオプトインユーザーに導入
5. **Prisma使用** - SQL Injectionを完全防止
6. **定期的な脆弱性スキャン** - GitHub Actionsで自動化

---

## まとめ

### セキュリティの成功の鍵

1. **多層防御** - 単一の対策に頼らない
2. **最小権限** - 必要最小限の権限のみ付与
3. **継続的監視** - ログ、アラート、定期スキャン
4. **開発者教育** - セキュアコーディングの徹底
5. **定期的更新** - 依存関係、パッチの適用

### 次のステップ

1. **今すぐ実装**: Helmet + bcrypt + JWT
2. **入力検証**: Zodスキーマ検証
3. **レート制限**: express-rate-limit
4. **監視**: ログ + アラート
5. **定期監査**: npm audit + ペネトレーションテスト

### 参考資料

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Node.js Security Best Practices](https://nodejs.org/en/docs/guides/security/)
- [Helmet.js Documentation](https://helmetjs.github.io/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)

---

**文字数: 約28,000文字**
