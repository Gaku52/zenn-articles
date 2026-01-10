---
title: "REST API設計で後悔しないために"
emoji: "🌐"
type: "tech"
topics: ["api", "backend", "rest", "nodejs"]
published: true
---

# REST API設計で後悔しないために

バックエンド開発を始めると、すぐに直面するのがAPI設計です。「とりあえず動けばいい」と思って作ったAPIが、後から大きな問題を引き起こすことは珍しくありません。この記事では、REST API設計における重要な3つの原則と、実践的な設計例を紹介します。

## よくあるAPI設計の失敗

開発現場でよく見かける問題をいくつか挙げてみます。

```
❌ 悪い設計
POST /createUser
GET  /getUserList
POST /deleteUserById?id=123
GET  /api/user_posts/getUserPostsByID
```

この設計の何が問題なのでしょうか？

- URLに動詞が含まれている（createUser, getUserList）
- HTTPメソッドの使い方が間違っている（削除なのにPOST）
- 命名規則がバラバラ（snake_case、camelCase）
- クエリパラメータの不適切な使用

これらの問題は、API設計の基本原則を理解していれば回避できます。

## REST API設計の3原則

### 1. リソース指向で考える

RESTの最も重要な概念は「リソース」です。URLは「何をする」ではなく「何に対して操作するか」を表現します。

**原則:**
- URLには名詞を使う（動詞は使わない）
- 複数形を使う（コレクションを表現）
- 階層構造で親子関係を明確にする

```
✅ 良い設計
GET    /users              # ユーザー一覧
POST   /users              # ユーザー作成
GET    /users/123          # 特定ユーザー
PUT    /users/123          # ユーザー更新
DELETE /users/123          # ユーザー削除
GET    /users/123/posts    # ユーザーの投稿一覧
```

### 2. HTTPメソッドを正しく使う

HTTPメソッドには、それぞれ明確な役割があります。

| メソッド | 用途 | 冪等性 |
|---------|------|--------|
| GET | リソース取得 | ✅ |
| POST | リソース作成 | ❌ |
| PUT | リソース全体更新 | ✅ |
| PATCH | リソース部分更新 | ❌ |
| DELETE | リソース削除 | ✅ |

**冪等性**とは、「同じリクエストを複数回実行しても結果が同じ」という性質です。

**実装例:**

```typescript
import { Router } from 'express'
import { PrismaClient } from '@prisma/client'

const router = Router()
const prisma = new PrismaClient()

// GET - リソース取得
router.get('/users/:id', async (req, res) => {
  const user = await prisma.user.findUnique({
    where: { id: req.params.id },
  })

  if (!user) {
    return res.status(404).json({
      error: { code: 'USER_NOT_FOUND', message: 'User not found' }
    })
  }

  res.json({ success: true, data: user })
})

// POST - リソース作成
router.post('/users', async (req, res) => {
  const { email, name } = req.body

  const user = await prisma.user.create({
    data: { email, name },
  })

  res.status(201)
    .setHeader('Location', `/users/${user.id}`)
    .json({ success: true, data: user })
})

// PATCH - 部分更新
router.patch('/users/:id', async (req, res) => {
  const updates = req.body

  const user = await prisma.user.update({
    where: { id: req.params.id },
    data: updates,
  })

  res.json({ success: true, data: user })
})

// DELETE - リソース削除
router.delete('/users/:id', async (req, res) => {
  await prisma.user.delete({
    where: { id: req.params.id },
  })

  res.status(204).send() // No Content
})
```

### 3. ステータスコードを適切に選択する

HTTPステータスコードは、クライアントに処理結果を伝える重要な手段です。

**成功レスポンス（2xx）:**
- `200 OK`: GET/PUT/PATCHの成功
- `201 Created`: POSTでリソース作成成功
- `204 No Content`: DELETE成功（レスポンスボディなし）

**クライアントエラー（4xx）:**
- `400 Bad Request`: バリデーションエラー
- `401 Unauthorized`: 認証エラー
- `403 Forbidden`: 権限不足
- `404 Not Found`: リソースが存在しない
- `409 Conflict`: リソースの競合（メールアドレス重複など）

**サーバーエラー（5xx）:**
- `500 Internal Server Error`: 予期しないエラー
- `503 Service Unavailable`: サービス一時停止

```typescript
// 404 Not Found
app.get('/users/:id', async (req, res) => {
  const user = await findUser(req.params.id)

  if (!user) {
    return res.status(404).json({
      error: {
        code: 'USER_NOT_FOUND',
        message: 'User not found'
      }
    })
  }

  res.json({ data: user })
})

// 409 Conflict
app.post('/users', async (req, res) => {
  const existing = await findUserByEmail(req.body.email)

  if (existing) {
    return res.status(409).json({
      error: {
        code: 'EMAIL_ALREADY_EXISTS',
        message: 'Email already exists'
      }
    })
  }

  const user = await createUser(req.body)
  res.status(201).json({ data: user })
})
```

## 実践的な設計例：タスク管理API

ここまでの原則を踏まえて、簡単なタスク管理APIを設計してみます。

```typescript
// GET /tasks - タスク一覧（ページネーション付き）
router.get('/tasks', async (req, res) => {
  const page = parseInt(req.query.page as string) || 1
  const limit = parseInt(req.query.limit as string) || 10
  const skip = (page - 1) * limit

  const [tasks, total] = await Promise.all([
    prisma.task.findMany({
      skip,
      take: limit,
      orderBy: { createdAt: 'desc' },
    }),
    prisma.task.count(),
  ])

  res.json({
    success: true,
    data: tasks,
    meta: {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit),
      hasNextPage: page < Math.ceil(total / limit),
    },
  })
})

// POST /tasks - タスク作成
router.post('/tasks', async (req, res) => {
  const { title, description } = req.body

  // バリデーション
  if (!title || title.trim().length === 0) {
    return res.status(400).json({
      error: {
        code: 'VALIDATION_ERROR',
        message: 'Title is required',
      },
    })
  }

  const task = await prisma.task.create({
    data: {
      title,
      description,
      status: 'todo',
    },
  })

  res.status(201)
    .setHeader('Location', `/tasks/${task.id}`)
    .json({ success: true, data: task })
})

// PATCH /tasks/:id - タスク更新
router.patch('/tasks/:id', async (req, res) => {
  try {
    const task = await prisma.task.update({
      where: { id: req.params.id },
      data: req.body,
    })

    res.json({ success: true, data: task })
  } catch (error) {
    res.status(404).json({
      error: {
        code: 'TASK_NOT_FOUND',
        message: 'Task not found',
      },
    })
  }
})
```

この実装のポイント：

1. **ページネーション**: 大量のデータを効率的に取得
2. **統一されたレスポンス形式**: `{ success, data, meta }` で一貫性を保つ
3. **適切なステータスコード**: 201（作成）、204（削除）、404（Not Found）
4. **エラーハンドリング**: わかりやすいエラーコードとメッセージ

## まとめ

REST API設計の3つの原則をおさらいします。

1. **リソース指向**: URLは名詞で、リソースを表現する
2. **HTTPメソッドの正しい使い方**: GET/POST/PUT/PATCH/DELETEを適切に使い分ける
3. **ステータスコードの適切な選択**: 2xx/4xx/5xxで処理結果を明確に伝える

これらの原則を守るだけで、保守性が高く、直感的に使えるAPIを設計できます。

## 🌐 さらに堅牢なAPI設計を学ぶ

### 書籍で学べる本格的なAPI開発

この記事ではREST APIの基礎を解説しましたが、実際のプロダクション環境では、さらに高度な実装が必要です。

✅ **エラーハンドリング完全ガイド**
- RFC 7807準拠のエラーレスポンス設計
- リトライ戦略とサーキットブレーカーパターン
- タイムアウト制御とフォールバック処理

✅ **APIバージョニング戦略**
- URI/ヘッダー/クエリパラメータ方式の比較
- 後方互換性を維持する設計パターン
- 段階的な廃止（Deprecation）の実装

✅ **認証・認可の実装**
- JWT vs Session vs OAuth 2.0の選択基準
- RBAC/ABAC（ロール・属性ベースアクセス制御）
- セキュリティベストプラクティス

✅ **E-commerceバックエンド完全実装**
- 決済API統合（Stripe, PayPal）
- 在庫管理システムの設計
- 注文処理フローの実装

✅ **実測データに基づく最適化**
- API応答時間95ms→28ms（-70%）
- エラー率4.2%→0.3%（-93%）
- 52万字の完全ガイド

📚 **Backend Development完全ガイド 2026**
👉 https://zenn.dev/gaku52/books/backend-development-complete-guide-2026
