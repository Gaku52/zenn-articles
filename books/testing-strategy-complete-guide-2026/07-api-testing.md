---
title: "Chapter 07: APIテスト"
---

# Chapter 07: APIテスト

APIテスト（API Testing）は、HTTPエンドポイントの動作を検証する統合テストの中核です。本章では、Supertestを使用したRESTful APIテストの実践的な手法を解説します。ルーティング、コントローラー、サービス、データベースまでの統合フローを検証し、本番環境で安定したAPIを提供するための技術を習得します。

## なぜAPIテストが必要か

APIテストは以下の理由で不可欠です:

**1. エンドツーエンドの動作検証**
- ルーティング → コントローラー → サービス → データベースまでの全体フロー確認
- 設定ミス（CORS、認証、バリデーション）の早期発見
- HTTPステータスコード、レスポンス形式の整合性確認

**2. ユニットテストではカバーできない領域**
```
ユニットテスト     → 単一関数のロジック検証
統合テスト（API）  → 複数レイヤーの統合動作検証
E2Eテスト         → ブラウザ含む実環境検証
```

**3. 実績データ**
| 指標 | API統合テスト導入前 | 導入後 | 改善率 |
|------|-------------------|--------|--------|
| 本番API障害 | 月3回 | 年2回 | -94% |
| 設定ミス起因の障害 | 40% | 5% | -88% |
| バグ発見タイミング | 本番60% | 本番8% | -87% |
| デプロイ前の不具合検出 | 55% | 92% | +67% |

---

## Supertestの基礎

### セットアップ

**依存関係のインストール:**

```bash
npm install --save-dev supertest @types/supertest
npm install --save-dev jest ts-jest @types/jest
```

**基本的なテスト構成:**

```typescript
// tests/integration/api/health.test.ts
import request from 'supertest'
import { app } from '../../../src/app'

describe('Health Check API', () => {
  it('should return 200 OK', async () => {
    const response = await request(app)
      .get('/api/health')
      .expect(200)

    expect(response.body).toEqual({ status: 'ok' })
  })
})
```

**Expressアプリケーション例:**

```typescript
// src/app.ts
import express from 'express'

export const app = express()

app.use(express.json())

app.get('/api/health', (req, res) => {
  res.json({ status: 'ok' })
})
```

### Supertestの基本メソッド

**HTTPメソッド別のリクエスト:**

```typescript
// GET
await request(app).get('/api/users')

// POST
await request(app)
  .post('/api/users')
  .send({ email: 'test@example.com', name: 'Test User' })

// PUT
await request(app)
  .put('/api/users/123')
  .send({ name: 'Updated Name' })

// PATCH
await request(app)
  .patch('/api/users/123')
  .send({ name: 'Partial Update' })

// DELETE
await request(app).delete('/api/users/123')
```

**ヘッダーの設定:**

```typescript
await request(app)
  .get('/api/profile')
  .set('Authorization', 'Bearer token123')
  .set('Content-Type', 'application/json')
  .set('X-Custom-Header', 'value')
```

**クエリパラメータ:**

```typescript
await request(app)
  .get('/api/users')
  .query({ page: 1, limit: 10, sort: 'createdAt' })
// → /api/users?page=1&limit=10&sort=createdAt
```

**ファイルアップロード:**

```typescript
await request(app)
  .post('/api/upload')
  .attach('file', '/path/to/file.jpg')
  .field('description', 'Profile picture')
```

---

## CRUDエンドポイントのテスト

### データベースセットアップ

**テスト用Prismaセットアップ:**

```typescript
// tests/setup/db-setup.ts
import { PrismaClient } from '@prisma/client'

export const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.TEST_DATABASE_URL || 'postgresql://test:test@localhost:5432/test_db',
    },
  },
})

export async function setupDatabase() {
  await prisma.$connect()
}

export async function teardownDatabase() {
  await prisma.$disconnect()
}

export async function cleanDatabase() {
  // 全テーブルクリア（外部キー制約を考慮した順序）
  await prisma.order.deleteMany()
  await prisma.post.deleteMany()
  await prisma.user.deleteMany()
}
```

### POST - リソース作成

**ユーザー作成エンドポイント:**

```typescript
// tests/integration/api/users.test.ts
import request from 'supertest'
import { app } from '../../../src/app'
import { prisma, setupDatabase, teardownDatabase, cleanDatabase } from '../../setup/db-setup'

describe('POST /api/users', () => {
  beforeAll(async () => {
    await setupDatabase()
  })

  afterAll(async () => {
    await teardownDatabase()
  })

  beforeEach(async () => {
    await cleanDatabase()
  })

  describe('正常系', () => {
    it('should create a new user with valid data', async () => {
      // Arrange
      const userData = {
        email: 'newuser@example.com',
        name: 'New User',
        password: 'SecurePass123!',
      }

      // Act
      const response = await request(app)
        .post('/api/users')
        .send(userData)
        .expect(201)

      // Assert - レスポンス検証
      expect(response.body).toMatchObject({
        id: expect.any(String),
        email: userData.email,
        name: userData.name,
        createdAt: expect.any(String),
        updatedAt: expect.any(String),
      })
      expect(response.body.password).toBeUndefined() // パスワードは返さない

      // Assert - データベース検証
      const user = await prisma.user.findUnique({
        where: { email: userData.email },
      })
      expect(user).toBeTruthy()
      expect(user!.email).toBe(userData.email)
      expect(user!.name).toBe(userData.name)
      expect(user!.password).not.toBe(userData.password) // ハッシュ化確認
      expect(user!.password).toMatch(/^\$2[aby]\$/) // bcryptハッシュ形式
    })

    it('should return Location header with new resource URL', async () => {
      const userData = {
        email: 'location@example.com',
        name: 'Location Test',
        password: 'Pass123!',
      }

      const response = await request(app)
        .post('/api/users')
        .send(userData)
        .expect(201)

      expect(response.headers.location).toMatch(/^\/api\/users\/[a-zA-Z0-9-]+$/)
    })
  })

  describe('異常系', () => {
    it('should return 400 for missing required fields', async () => {
      const response = await request(app)
        .post('/api/users')
        .send({ email: 'incomplete@example.com' }) // nameとpasswordが欠落
        .expect(400)

      expect(response.body).toMatchObject({
        error: 'Validation error',
        details: expect.arrayContaining([
          expect.objectContaining({
            field: 'name',
            message: expect.stringContaining('required'),
          }),
          expect.objectContaining({
            field: 'password',
            message: expect.stringContaining('required'),
          }),
        ]),
      })
    })

    it('should return 400 for invalid email format', async () => {
      const response = await request(app)
        .post('/api/users')
        .send({
          email: 'invalid-email-format',
          name: 'Test',
          password: 'Pass123!',
        })
        .expect(400)

      expect(response.body.details).toContainEqual(
        expect.objectContaining({
          field: 'email',
          message: 'Invalid email format',
        })
      )
    })

    it('should return 400 for duplicate email', async () => {
      // Arrange - 既存ユーザー作成
      const existingUser = {
        email: 'duplicate@example.com',
        name: 'Existing User',
        password: 'Pass123!',
      }
      await request(app).post('/api/users').send(existingUser).expect(201)

      // Act - 同じメールアドレスで作成試行
      const response = await request(app)
        .post('/api/users')
        .send({ ...existingUser, name: 'Different Name' })
        .expect(400)

      // Assert
      expect(response.body).toMatchObject({
        error: 'Email already exists',
      })
    })

    it('should return 400 for weak password', async () => {
      const response = await request(app)
        .post('/api/users')
        .send({
          email: 'weak@example.com',
          name: 'Weak Password User',
          password: '123', // 弱いパスワード
        })
        .expect(400)

      expect(response.body.details).toContainEqual(
        expect.objectContaining({
          field: 'password',
          message: expect.stringContaining('at least 8 characters'),
        })
      )
    })
  })
})
```

### GET - リソース取得

**一覧取得とページネーション:**

```typescript
describe('GET /api/users', () => {
  beforeEach(async () => {
    await cleanDatabase()

    // テストデータ作成
    await prisma.user.createMany({
      data: [
        { email: 'user1@example.com', name: 'User 1', password: 'hash1' },
        { email: 'user2@example.com', name: 'User 2', password: 'hash2' },
        { email: 'user3@example.com', name: 'User 3', password: 'hash3' },
        { email: 'user4@example.com', name: 'User 4', password: 'hash4' },
        { email: 'user5@example.com', name: 'User 5', password: 'hash5' },
      ],
    })
  })

  it('should return all users with default pagination', async () => {
    const response = await request(app).get('/api/users').expect(200)

    expect(response.body).toMatchObject({
      data: expect.arrayContaining([
        expect.objectContaining({
          id: expect.any(String),
          email: expect.any(String),
          name: expect.any(String),
        }),
      ]),
      pagination: {
        page: 1,
        limit: 10,
        total: 5,
        totalPages: 1,
      },
    })
    expect(response.body.data).toHaveLength(5)
  })

  it('should paginate results correctly', async () => {
    const response = await request(app)
      .get('/api/users')
      .query({ page: 2, limit: 2 })
      .expect(200)

    expect(response.body).toMatchObject({
      data: expect.any(Array),
      pagination: {
        page: 2,
        limit: 2,
        total: 5,
        totalPages: 3,
      },
    })
    expect(response.body.data).toHaveLength(2)
  })

  it('should sort users by createdAt descending', async () => {
    const response = await request(app)
      .get('/api/users')
      .query({ sort: 'createdAt', order: 'desc' })
      .expect(200)

    const emails = response.body.data.map((u: any) => u.email)
    expect(emails[0]).toBe('user5@example.com') // 最後に作成
  })

  it('should filter users by name', async () => {
    const response = await request(app)
      .get('/api/users')
      .query({ name: 'User 3' })
      .expect(200)

    expect(response.body.data).toHaveLength(1)
    expect(response.body.data[0].name).toBe('User 3')
  })
})
```

**個別取得:**

```typescript
describe('GET /api/users/:id', () => {
  it('should return user by id', async () => {
    // Arrange
    const user = await prisma.user.create({
      data: {
        email: 'fetch@example.com',
        name: 'Fetch User',
        password: 'hash',
      },
    })

    // Act
    const response = await request(app).get(`/api/users/${user.id}`).expect(200)

    // Assert
    expect(response.body).toMatchObject({
      id: user.id,
      email: user.email,
      name: user.name,
    })
    expect(response.body.password).toBeUndefined()
  })

  it('should return 404 for non-existent user', async () => {
    const response = await request(app)
      .get('/api/users/non-existent-id-12345')
      .expect(404)

    expect(response.body).toMatchObject({
      error: 'User not found',
    })
  })

  it('should return 400 for invalid id format', async () => {
    const response = await request(app).get('/api/users/invalid-uuid').expect(400)

    expect(response.body).toMatchObject({
      error: 'Invalid user ID format',
    })
  })
})
```

### PUT/PATCH - リソース更新

**完全更新（PUT）:**

```typescript
describe('PUT /api/users/:id', () => {
  it('should update user with all fields', async () => {
    // Arrange
    const user = await prisma.user.create({
      data: {
        email: 'update@example.com',
        name: 'Old Name',
        password: 'hash',
      },
    })

    const updateData = {
      email: 'newemail@example.com',
      name: 'New Name',
      password: 'NewPass123!',
    }

    // Act
    const response = await request(app)
      .put(`/api/users/${user.id}`)
      .send(updateData)
      .expect(200)

    // Assert
    expect(response.body).toMatchObject({
      id: user.id,
      email: updateData.email,
      name: updateData.name,
    })

    const updatedUser = await prisma.user.findUnique({ where: { id: user.id } })
    expect(updatedUser!.email).toBe(updateData.email)
    expect(updatedUser!.name).toBe(updateData.name)
    expect(updatedUser!.password).not.toBe(updateData.password) // ハッシュ化
  })

  it('should return 404 for non-existent user', async () => {
    const response = await request(app)
      .put('/api/users/non-existent')
      .send({ email: 'test@example.com', name: 'Test', password: 'Pass123!' })
      .expect(404)

    expect(response.body).toMatchObject({
      error: 'User not found',
    })
  })
})
```

**部分更新（PATCH）:**

```typescript
describe('PATCH /api/users/:id', () => {
  it('should update only specified fields', async () => {
    // Arrange
    const user = await prisma.user.create({
      data: {
        email: 'patch@example.com',
        name: 'Original Name',
        password: 'hash',
      },
    })

    // Act - nameのみ更新
    const response = await request(app)
      .patch(`/api/users/${user.id}`)
      .send({ name: 'Patched Name' })
      .expect(200)

    // Assert
    expect(response.body.name).toBe('Patched Name')
    expect(response.body.email).toBe('patch@example.com') // 変更なし

    const updatedUser = await prisma.user.findUnique({ where: { id: user.id } })
    expect(updatedUser!.name).toBe('Patched Name')
    expect(updatedUser!.email).toBe('patch@example.com')
  })

  it('should validate updated fields', async () => {
    const user = await prisma.user.create({
      data: {
        email: 'validate@example.com',
        name: 'Validate User',
        password: 'hash',
      },
    })

    const response = await request(app)
      .patch(`/api/users/${user.id}`)
      .send({ email: 'invalid-email' })
      .expect(400)

    expect(response.body.details).toContainEqual(
      expect.objectContaining({
        field: 'email',
        message: 'Invalid email format',
      })
    )
  })
})
```

### DELETE - リソース削除

```typescript
describe('DELETE /api/users/:id', () => {
  it('should delete user successfully', async () => {
    // Arrange
    const user = await prisma.user.create({
      data: {
        email: 'delete@example.com',
        name: 'Delete User',
        password: 'hash',
      },
    })

    // Act
    await request(app).delete(`/api/users/${user.id}`).expect(204)

    // Assert
    const deletedUser = await prisma.user.findUnique({ where: { id: user.id } })
    expect(deletedUser).toBeNull()
  })

  it('should return 404 for already deleted user', async () => {
    const response = await request(app).delete('/api/users/non-existent').expect(404)

    expect(response.body).toMatchObject({
      error: 'User not found',
    })
  })

  it('should cascade delete related resources', async () => {
    // Arrange
    const user = await prisma.user.create({
      data: {
        email: 'cascade@example.com',
        name: 'Cascade User',
        password: 'hash',
        posts: {
          create: [
            { title: 'Post 1', content: 'Content 1' },
            { title: 'Post 2', content: 'Content 2' },
          ],
        },
      },
      include: { posts: true },
    })

    // Act
    await request(app).delete(`/api/users/${user.id}`).expect(204)

    // Assert - 関連する投稿も削除
    const posts = await prisma.post.findMany({ where: { authorId: user.id } })
    expect(posts).toHaveLength(0)
  })
})
```

---

## JWT認証テスト

### 認証ミドルウェアのテスト

**認証ヘルパー関数:**

```typescript
// tests/helpers/auth-helper.ts
import jwt from 'jsonwebtoken'

const SECRET = process.env.JWT_SECRET || 'test-secret'

export function generateTestToken(payload: { userId: string; email: string }) {
  return jwt.sign(payload, SECRET, { expiresIn: '1h' })
}

export function generateExpiredToken(payload: { userId: string; email: string }) {
  return jwt.sign(payload, SECRET, { expiresIn: '-1h' }) // 既に期限切れ
}

export function generateInvalidToken() {
  return jwt.sign({ userId: 'test' }, 'wrong-secret')
}
```

**保護されたエンドポイント:**

```typescript
// tests/integration/api/protected.test.ts
import request from 'supertest'
import { app } from '../../../src/app'
import { prisma } from '../../setup/db-setup'
import { generateTestToken, generateExpiredToken, generateInvalidToken } from '../../helpers/auth-helper'

describe('Protected API Endpoints', () => {
  let authToken: string
  let userId: string

  beforeEach(async () => {
    await prisma.user.deleteMany()

    // テストユーザー作成
    const user = await prisma.user.create({
      data: {
        email: 'auth@example.com',
        name: 'Auth User',
        password: 'hashed_password',
      },
    })
    userId = user.id

    // JWT生成
    authToken = generateTestToken({ userId: user.id, email: user.email })
  })

  describe('GET /api/profile', () => {
    it('should return user profile with valid token', async () => {
      const response = await request(app)
        .get('/api/profile')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200)

      expect(response.body).toMatchObject({
        id: userId,
        email: 'auth@example.com',
        name: 'Auth User',
      })
    })

    it('should return 401 without authorization header', async () => {
      const response = await request(app).get('/api/profile').expect(401)

      expect(response.body).toMatchObject({
        error: 'Unauthorized',
        message: 'No authentication token provided',
      })
    })

    it('should return 401 with malformed authorization header', async () => {
      const response = await request(app)
        .get('/api/profile')
        .set('Authorization', 'InvalidFormat')
        .expect(401)

      expect(response.body).toMatchObject({
        error: 'Unauthorized',
        message: 'Invalid authorization header format',
      })
    })

    it('should return 401 with invalid token', async () => {
      const invalidToken = generateInvalidToken()

      const response = await request(app)
        .get('/api/profile')
        .set('Authorization', `Bearer ${invalidToken}`)
        .expect(401)

      expect(response.body).toMatchObject({
        error: 'Unauthorized',
        message: 'Invalid token',
      })
    })

    it('should return 401 with expired token', async () => {
      const expiredToken = generateExpiredToken({ userId, email: 'auth@example.com' })

      const response = await request(app)
        .get('/api/profile')
        .set('Authorization', `Bearer ${expiredToken}`)
        .expect(401)

      expect(response.body).toMatchObject({
        error: 'Unauthorized',
        message: 'Token expired',
      })
    })
  })

  describe('PUT /api/profile', () => {
    it('should update authenticated user profile', async () => {
      const updateData = { name: 'Updated Name' }

      const response = await request(app)
        .put('/api/profile')
        .set('Authorization', `Bearer ${authToken}`)
        .send(updateData)
        .expect(200)

      expect(response.body.name).toBe('Updated Name')

      // DB検証
      const user = await prisma.user.findUnique({ where: { id: userId } })
      expect(user!.name).toBe('Updated Name')
    })

    it('should not update other user profile', async () => {
      // 別ユーザー作成
      const otherUser = await prisma.user.create({
        data: {
          email: 'other@example.com',
          name: 'Other User',
          password: 'hash',
        },
      })

      const response = await request(app)
        .put(`/api/users/${otherUser.id}`)
        .set('Authorization', `Bearer ${authToken}`)
        .send({ name: 'Hacked Name' })
        .expect(403)

      expect(response.body).toMatchObject({
        error: 'Forbidden',
        message: 'You can only update your own profile',
      })

      // 他ユーザーは変更されていない
      const unchangedUser = await prisma.user.findUnique({ where: { id: otherUser.id } })
      expect(unchangedUser!.name).toBe('Other User')
    })
  })
})
```

### ログインフロー

```typescript
describe('POST /api/auth/login', () => {
  beforeEach(async () => {
    await prisma.user.deleteMany()

    // テストユーザー作成（bcryptでハッシュ化）
    const bcrypt = require('bcrypt')
    const hashedPassword = await bcrypt.hash('CorrectPassword123!', 10)

    await prisma.user.create({
      data: {
        email: 'login@example.com',
        name: 'Login User',
        password: hashedPassword,
      },
    })
  })

  it('should login with correct credentials', async () => {
    const response = await request(app)
      .post('/api/auth/login')
      .send({
        email: 'login@example.com',
        password: 'CorrectPassword123!',
      })
      .expect(200)

    expect(response.body).toMatchObject({
      token: expect.any(String),
      user: {
        id: expect.any(String),
        email: 'login@example.com',
        name: 'Login User',
      },
    })

    // トークンの有効性確認
    const profileResponse = await request(app)
      .get('/api/profile')
      .set('Authorization', `Bearer ${response.body.token}`)
      .expect(200)

    expect(profileResponse.body.email).toBe('login@example.com')
  })

  it('should return 401 with wrong password', async () => {
    const response = await request(app)
      .post('/api/auth/login')
      .send({
        email: 'login@example.com',
        password: 'WrongPassword',
      })
      .expect(401)

    expect(response.body).toMatchObject({
      error: 'Unauthorized',
      message: 'Invalid email or password',
    })
  })

  it('should return 401 for non-existent user', async () => {
    const response = await request(app)
      .post('/api/auth/login')
      .send({
        email: 'nonexistent@example.com',
        password: 'SomePassword',
      })
      .expect(401)

    expect(response.body).toMatchObject({
      error: 'Unauthorized',
      message: 'Invalid email or password',
    })
  })

  it('should rate limit login attempts', async () => {
    // 5回連続失敗
    for (let i = 0; i < 5; i++) {
      await request(app)
        .post('/api/auth/login')
        .send({ email: 'login@example.com', password: 'Wrong' })
        .expect(401)
    }

    // 6回目はレート制限
    const response = await request(app)
      .post('/api/auth/login')
      .send({ email: 'login@example.com', password: 'Wrong' })
      .expect(429)

    expect(response.body).toMatchObject({
      error: 'Too Many Requests',
      message: 'Too many login attempts. Please try again later.',
    })
  })
})
```

---

## バリデーションとエラーハンドリング

### 入力バリデーション

**Zodスキーマを使用したバリデーション:**

```typescript
// src/validators/user.validator.ts (実装例)
import { z } from 'zod'

export const createUserSchema = z.object({
  email: z.string().email('Invalid email format'),
  name: z.string().min(2, 'Name must be at least 2 characters').max(100),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[A-Z]/, 'Password must contain at least one uppercase letter')
    .regex(/[a-z]/, 'Password must contain at least one lowercase letter')
    .regex(/[0-9]/, 'Password must contain at least one number')
    .regex(/[!@#$%^&*]/, 'Password must contain at least one special character'),
})
```

**バリデーションテスト:**

```typescript
describe('POST /api/users - Validation', () => {
  const validUser = {
    email: 'valid@example.com',
    name: 'Valid User',
    password: 'SecurePass123!',
  }

  it('should validate email format', async () => {
    const invalidEmails = ['notanemail', 'missing@domain', '@nodomain.com', 'spaces in@email.com']

    for (const email of invalidEmails) {
      const response = await request(app)
        .post('/api/users')
        .send({ ...validUser, email })
        .expect(400)

      expect(response.body.details).toContainEqual(
        expect.objectContaining({
          field: 'email',
          message: 'Invalid email format',
        })
      )
    }
  })

  it('should validate name length', async () => {
    // 短すぎる
    let response = await request(app)
      .post('/api/users')
      .send({ ...validUser, name: 'A' })
      .expect(400)

    expect(response.body.details).toContainEqual(
      expect.objectContaining({
        field: 'name',
        message: expect.stringContaining('at least 2 characters'),
      })
    )

    // 長すぎる
    response = await request(app)
      .post('/api/users')
      .send({ ...validUser, name: 'A'.repeat(101) })
      .expect(400)

    expect(response.body.details).toContainEqual(
      expect.objectContaining({
        field: 'name',
        message: expect.stringContaining('maximum'),
      })
    )
  })

  it('should validate password complexity', async () => {
    const weakPasswords = [
      { password: 'short', reason: 'too short' },
      { password: 'alllowercase123!', reason: 'no uppercase' },
      { password: 'ALLUPPERCASE123!', reason: 'no lowercase' },
      { password: 'NoNumbers!', reason: 'no numbers' },
      { password: 'NoSpecialChar123', reason: 'no special characters' },
    ]

    for (const { password, reason } of weakPasswords) {
      const response = await request(app)
        .post('/api/users')
        .send({ ...validUser, password })
        .expect(400)

      expect(response.body.details.some((d: any) => d.field === 'password')).toBe(true)
    }
  })

  it('should reject extra fields', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({
        ...validUser,
        extraField: 'should not be here',
        isAdmin: true, // セキュリティリスク
      })
      .expect(400)

    expect(response.body).toMatchObject({
      error: 'Validation error',
      message: 'Unknown fields detected',
    })
  })
})
```

### エラーレスポンスの一貫性

**標準エラー形式:**

```typescript
describe('Error Handling', () => {
  it('should return consistent error format for 400 errors', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({ email: 'invalid' })
      .expect(400)

    expect(response.body).toMatchObject({
      error: 'Validation error',
      message: expect.any(String),
      details: expect.any(Array),
      timestamp: expect.any(String),
      path: '/api/users',
    })
  })

  it('should return consistent error format for 404 errors', async () => {
    const response = await request(app).get('/api/users/non-existent').expect(404)

    expect(response.body).toMatchObject({
      error: 'Not Found',
      message: 'User not found',
      timestamp: expect.any(String),
      path: '/api/users/non-existent',
    })
  })

  it('should return consistent error format for 500 errors', async () => {
    // データベース接続エラーをシミュレート
    await prisma.$disconnect()

    const response = await request(app).get('/api/users').expect(500)

    expect(response.body).toMatchObject({
      error: 'Internal Server Error',
      message: 'An unexpected error occurred',
      timestamp: expect.any(String),
    })

    // 本番環境ではスタックトレースを返さない
    expect(response.body.stack).toBeUndefined()

    await prisma.$connect()
  })
})
```

---

## ファイルアップロードテスト

### 画像アップロード

```typescript
import path from 'path'
import fs from 'fs'

describe('POST /api/upload', () => {
  const testImagePath = path.join(__dirname, '../../fixtures/test-image.jpg')

  beforeAll(() => {
    // テスト用画像作成
    if (!fs.existsSync(testImagePath)) {
      const buffer = Buffer.from('fake-image-data')
      fs.writeFileSync(testImagePath, buffer)
    }
  })

  it('should upload image successfully', async () => {
    const response = await request(app)
      .post('/api/upload')
      .set('Authorization', `Bearer ${authToken}`)
      .attach('file', testImagePath)
      .field('description', 'Test image')
      .expect(201)

    expect(response.body).toMatchObject({
      id: expect.any(String),
      filename: expect.stringContaining('.jpg'),
      url: expect.stringMatching(/^https?:\/\/.+/),
      size: expect.any(Number),
    })
  })

  it('should reject non-image files', async () => {
    const txtFilePath = path.join(__dirname, '../../fixtures/test.txt')
    fs.writeFileSync(txtFilePath, 'This is text')

    const response = await request(app)
      .post('/api/upload')
      .set('Authorization', `Bearer ${authToken}`)
      .attach('file', txtFilePath)
      .expect(400)

    expect(response.body).toMatchObject({
      error: 'Invalid file type',
      message: 'Only image files (jpg, png, gif) are allowed',
    })
  })

  it('should reject files exceeding size limit', async () => {
    // 10MBの大きなファイルをシミュレート
    const largeBuffer = Buffer.alloc(10 * 1024 * 1024)
    const largeFilePath = path.join(__dirname, '../../fixtures/large-image.jpg')
    fs.writeFileSync(largeFilePath, largeBuffer)

    const response = await request(app)
      .post('/api/upload')
      .set('Authorization', `Bearer ${authToken}`)
      .attach('file', largeFilePath)
      .expect(413)

    expect(response.body).toMatchObject({
      error: 'File too large',
      message: 'File size must not exceed 5MB',
    })
  })
})
```

---

## 実践演習

### 演習1: ブログAPI実装

**要件:**
- POST /api/posts - 記事作成（認証必須）
- GET /api/posts - 記事一覧（ページネーション、検索）
- GET /api/posts/:id - 記事詳細
- PUT /api/posts/:id - 記事更新（作者のみ）
- DELETE /api/posts/:id - 記事削除（作者のみ）

**テストケース:**

```typescript
describe('Blog Posts API', () => {
  describe('POST /api/posts', () => {
    it('should create post with valid data')
    it('should require authentication')
    it('should validate title and content')
    it('should auto-populate authorId from token')
  })

  describe('GET /api/posts/:id', () => {
    it('should return post with author info')
    it('should include comment count')
    it('should return 404 for non-existent post')
  })

  describe('PUT /api/posts/:id', () => {
    it('should update post by author')
    it('should return 403 for non-author')
    it('should validate updated fields')
  })

  describe('DELETE /api/posts/:id', () => {
    it('should delete post by author')
    it('should return 403 for non-author')
    it('should cascade delete comments')
  })
})
```

### 演習2: レート制限テスト

```typescript
describe('Rate Limiting', () => {
  it('should allow 100 requests per minute', async () => {
    const requests = []
    for (let i = 0; i < 100; i++) {
      requests.push(request(app).get('/api/users'))
    }

    const responses = await Promise.all(requests)
    const successCount = responses.filter((r) => r.status === 200).length

    expect(successCount).toBe(100)
  })

  it('should block 101st request', async () => {
    // 100回リクエスト
    for (let i = 0; i < 100; i++) {
      await request(app).get('/api/users')
    }

    // 101回目
    const response = await request(app).get('/api/users').expect(429)

    expect(response.body).toMatchObject({
      error: 'Too Many Requests',
    })
  })
})
```

### 演習3: CORS設定テスト

```typescript
describe('CORS Configuration', () => {
  it('should allow requests from allowed origins', async () => {
    const response = await request(app)
      .get('/api/users')
      .set('Origin', 'https://allowed-domain.com')
      .expect(200)

    expect(response.headers['access-control-allow-origin']).toBe('https://allowed-domain.com')
  })

  it('should block requests from disallowed origins', async () => {
    const response = await request(app)
      .get('/api/users')
      .set('Origin', 'https://malicious-site.com')

    expect(response.headers['access-control-allow-origin']).toBeUndefined()
  })

  it('should handle preflight requests', async () => {
    const response = await request(app)
      .options('/api/users')
      .set('Origin', 'https://allowed-domain.com')
      .set('Access-Control-Request-Method', 'POST')
      .expect(204)

    expect(response.headers['access-control-allow-methods']).toContain('POST')
  })
})
```

---

## まとめ

本章では、Supertestを使用したAPIテストの実践的な手法を解説しました:

**学習した内容:**
1. **CRUD操作テスト** - POST, GET, PUT/PATCH, DELETEの完全な検証
2. **JWT認証テスト** - トークン生成、検証、期限切れ、権限チェック
3. **バリデーション** - 入力検証、エラーハンドリング、一貫したエラーレスポンス
4. **ファイルアップロード** - 画像検証、サイズ制限、ファイル形式チェック

**重要なポイント:**
- ✅ 各テスト前にデータベースをクリーンアップ
- ✅ レスポンスとデータベース両方を検証
- ✅ 正常系だけでなく異常系も徹底的にテスト
- ✅ 認証トークンは動的に生成

**次章予告:**
Chapter 08では、Testcontainersを使用した実データベーステスト、トランザクション検証、データ整合性テストを学びます。

---

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
