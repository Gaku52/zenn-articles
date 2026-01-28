---
title: "エンドポイント設計と文書化"
---

# エンドポイント設計と文書化

## この章で学ぶこと

エンドポイント設計は、API開発における最も重要な判断の一つです。適切に設計されたエンドポイントは、直感的で使いやすく、長期的なメンテナンスも容易になります。一方、不適切な設計は、利用者の混乱を招き、後から修正することが困難になります。

この章では、RESTful APIの原則に基づいたエンドポイント設計の方法と、その設計をわかりやすく文書化する技術を学びます。具体的には以下のトピックを扱います：

- **RESTful APIの原則**: リソース指向、統一インターフェース、ステートレス性など、REST設計の基本原則
- **エンドポイント命名規則**: 一貫性のある、予測可能な命名パターン
- **HTTPメソッドの選択**: GET、POST、PUT、PATCH、DELETEの適切な使い分け
- **ステータスコードの定義**: 成功、失敗、エラーを正確に表現するステータスコード
- **クエリパラメータとパスパラメータ**: データの取得・フィルタリングの最適な方法
- **バージョニング戦略**: APIの進化を管理する方法

## なぜ重要か

エンドポイント設計の品質は、APIの成功を左右します：

1. **開発効率**: 一貫した設計により、新しいエンドポイントの実装とドキュメント化が迅速になります
2. **利用者体験**: 直感的なエンドポイントは、学習コストを下げ、統合を容易にします
3. **保守性**: 明確な設計原則は、将来の変更や拡張を容易にします
4. **品質保証**: 体系的な設計により、テストの網羅性が向上します
5. **コミュニケーション**: 明確な文書化により、チーム間の誤解が減少します

## 前提知識

この章を理解するには、以下の知識があると有益です：

- HTTPプロトコルの基礎（メソッド、ヘッダー、ステータスコード）
- JSONフォーマットの基本
- RESTの概念（既存の知識がなくても、この章で学べます）
- API仕様書の基本（第8章で扱った内容）

---

## RESTful APIの原則

REST（Representational State Transfer）は、Roy Fieldingによって2000年に提唱されたアーキテクチャスタイルです。RESTful APIは、以下の原則に基づいて設計されます。

### リソース指向設計

RESTの中心概念は「リソース」です。リソースとは、APIが扱うデータや機能の単位を指します。

**リソースの例：**
- ユーザー（Users）
- 記事（Articles）
- コメント（Comments）
- 注文（Orders）
- 商品（Products）

**リソース指向の考え方：**

```
❌ 動詞ベースの設計（非REST）
POST /api/createUser
POST /api/deleteUser
GET /api/getUserList
POST /api/updateUser

✅ 名詞ベースの設計（REST）
POST /api/users          # ユーザーを作成
DELETE /api/users/:id    # ユーザーを削除
GET /api/users           # ユーザー一覧を取得
PUT /api/users/:id       # ユーザーを更新
```

リソース指向設計の利点：
- **予測可能性**: HTTPメソッドとリソースの組み合わせで操作が明確
- **一貫性**: すべてのリソースに同じパターンを適用できる
- **スケーラビリティ**: 新しいリソースの追加が容易

### 統一インターフェース

RESTでは、HTTPメソッドを標準的な操作に対応させます。これにより、すべてのリソースに対して一貫したインターフェースを提供できます。

**標準的なCRUD操作：**

| HTTPメソッド | 操作 | 説明 | 冪等性 |
|-------------|------|------|--------|
| GET | Read | リソースの取得 | ○ |
| POST | Create | リソースの作成 | × |
| PUT | Update/Replace | リソースの完全置換 | ○ |
| PATCH | Update/Modify | リソースの部分更新 | △ |
| DELETE | Delete | リソースの削除 | ○ |

**冪等性（Idempotency）とは：**
同じリクエストを複数回実行しても、結果が変わらない性質を指します。

```typescript
// ✅ GET は冪等（何回実行しても同じ結果）
GET /api/users/123
→ 常に同じユーザー情報を返す

// ✅ DELETE は冪等（何回実行しても結果は同じ）
DELETE /api/users/123
→ 1回目: ユーザーを削除（204 No Content）
→ 2回目以降: すでに存在しない（404 Not Found または 204）

// ❌ POST は非冪等（実行するたびに新しいリソースが作成される）
POST /api/users
{
  "name": "John Doe",
  "email": "john@example.com"
}
→ 1回目: user_001 を作成
→ 2回目: user_002 を作成（重複）
→ 3回目: user_003 を作成（重複）
```

### ステートレス性

各リクエストは、完全に独立している必要があります。サーバーは、リクエスト間でクライアントの状態を保持しません。

**ステートレスな設計：**

```typescript
// ❌ ステートフルな設計（セッション依存）
GET /api/login
→ サーバーがセッションを作成

GET /api/current-user
→ セッション情報を使用してユーザーを特定

GET /api/logout
→ セッションを破棄

// ✅ ステートレスな設計（トークンベース）
POST /api/auth/login
{
  "email": "user@example.com",
  "password": "password123"
}
→ レスポンス: { "token": "eyJhbGc..." }

GET /api/users/me
Authorization: Bearer eyJhbGc...
→ トークンから直接ユーザーを特定
```

ステートレス設計の利点：
- **スケーラビリティ**: どのサーバーインスタンスでもリクエストを処理可能
- **信頼性**: サーバー障害時の影響が限定的
- **キャッシュ可能性**: リクエストが独立しているため、キャッシュが容易

### 階層化システム

クライアントは、直接接続しているのがエンドサーバーか、中間サーバー（プロキシ、ゲートウェイ）かを知る必要がありません。

**階層化の例：**

```
Client → Load Balancer → API Gateway → Application Server → Database
```

この設計により：
- 負荷分散やキャッシュレイヤーを透過的に追加可能
- セキュリティ層の追加が容易
- システムの進化が柔軟に

---

## エンドポイント命名規則

一貫性のある命名規則は、APIの使いやすさを大きく向上させます。

### 基本原則

**1. 名詞を使用する**

```typescript
// ✅ 良い例
GET /api/users
GET /api/articles
GET /api/products

// ❌ 悪い例
GET /api/getUsers
GET /api/fetchArticles
GET /api/retrieveProducts
```

**2. 複数形を使用する**

```typescript
// ✅ 良い例（一貫性がある）
GET /api/users           # 一覧取得
POST /api/users          # 作成
GET /api/users/123       # 個別取得
PUT /api/users/123       # 更新
DELETE /api/users/123    # 削除

// ❌ 悪い例（単数形と複数形が混在）
GET /api/user            # 混乱を招く
POST /api/users
GET /api/user/123        # どちらを使うべきか不明確
```

**例外**: 単数形が適切な場合もあります

```typescript
// ✅ シングルトンリソース（複数存在しない）
GET /api/profile         # 現在のユーザーのプロファイル
GET /api/configuration   # アプリケーション設定
GET /api/status          # システムステータス
```

**3. ケバブケースを使用する**

```typescript
// ✅ 良い例
GET /api/user-profiles
GET /api/blog-posts
GET /api/order-items

// ❌ 悪い例
GET /api/userProfiles    # キャメルケース
GET /api/user_profiles   # スネークケース
GET /api/UserProfiles    # パスカルケース
```

**理由**: URLは大文字小文字を区別する場合があり、ケバブケースが最も広く採用されています。

**4. 小文字を使用する**

```typescript
// ✅ 良い例
GET /api/users/123/orders

// ❌ 悪い例
GET /api/Users/123/Orders
GET /api/USERS/123/ORDERS
```

### リソースの階層構造

関連するリソースは、階層構造で表現します。

**親子関係の表現：**

```typescript
// ユーザーの記事
GET /api/users/123/articles

// 記事のコメント
GET /api/articles/456/comments

// コメントへの返信
GET /api/comments/789/replies
```

**深すぎる階層を避ける：**

```typescript
// ❌ 悪い例（4階層以上は避ける）
GET /api/users/123/articles/456/comments/789/replies/012

// ✅ 良い例（直接アクセス）
GET /api/replies/012
GET /api/replies?commentId=789

// ✅ 良い例（コメントを経由）
GET /api/comments/789/replies
```

**ガイドライン**: 一般的に、3階層までが推奨されます。

### フィルタリングとソート

クエリパラメータを使用してデータを絞り込みます。

**フィルタリング：**

```typescript
// ✅ 良い例
GET /api/users?status=active
GET /api/users?role=admin&status=active
GET /api/articles?author=123&published=true
GET /api/products?category=electronics&minPrice=100&maxPrice=500

// ❌ 悪い例（パスに含める）
GET /api/users/active
GET /api/users/admin/active
```

**ソート：**

```typescript
// ✅ 良い例
GET /api/users?sort=createdAt
GET /api/users?sort=-createdAt           # 降順（-プレフィックス）
GET /api/users?sort=lastName,firstName   # 複数フィールド

// ✅ 明示的な方法
GET /api/users?sortBy=createdAt&order=desc
```

**ペジネーション：**

```typescript
// ✅ オフセットベース
GET /api/users?limit=20&offset=40

// ✅ ページベース
GET /api/users?page=3&perPage=20

// ✅ カーソルベース（大規模データセット向け）
GET /api/users?cursor=eyJpZCI6MTIzfQ&limit=20
```

**検索：**

```typescript
// ✅ 良い例
GET /api/users?search=john
GET /api/articles?q=REST+API

// ✅ 複雑な検索
GET /api/users?name=john&email=*@example.com
```

### バージョニング

APIのバージョンを明示的に管理します。

**1. URLパスによるバージョニング（推奨）**

```typescript
// ✅ 良い例
GET /api/v1/users
GET /api/v2/users

// 利点:
// - 明確で視覚的
// - ドキュメント化が容易
// - バージョン間の切り替えが簡単
```

**2. ヘッダーによるバージョニング**

```typescript
// ✅ 良い例
GET /api/users
Accept: application/vnd.myapi.v1+json

// 利点:
// - URLが変わらない
// - RESTの原則に忠実
// 欠点:
// - 視認性が低い
// - テストやドキュメントが複雑
```

**3. クエリパラメータによるバージョニング**

```typescript
// △ 許容される例
GET /api/users?version=1

// 欠点:
// - バージョンが「リソースの一部」のように見える
// - あまり一般的ではない
```

**バージョニングのベストプラクティス：**

```typescript
// ✅ メジャーバージョンのみ
GET /api/v1/users
GET /api/v2/users

// ❌ マイナーバージョンまで含める（過度に細かい）
GET /api/v1.2.3/users

// ✅ 下位互換性のある変更は同じバージョン内で
GET /api/v1/users          # name フィールド
GET /api/v1/users          # displayName フィールドを追加（下位互換）

// ✅ 破壊的変更は新しいバージョン
GET /api/v1/users          # name フィールド
GET /api/v2/users          # name を firstName と lastName に分割
```

---

## HTTPメソッドの選択

各HTTPメソッドには明確な目的があります。適切なメソッドを選択することで、APIの意図が明確になります。

### GET - リソースの取得

**特徴：**
- リソースの状態を変更しない（安全性）
- 冪等性がある
- キャッシュ可能
- ボディを含まない（含めるべきではない）

**使用例：**

```typescript
// ✅ 一覧取得
GET /api/users
レスポンス: 200 OK
{
  "data": [
    { "id": "1", "name": "Alice" },
    { "id": "2", "name": "Bob" }
  ],
  "total": 2
}

// ✅ 個別取得
GET /api/users/123
レスポンス: 200 OK
{
  "id": "123",
  "name": "Alice",
  "email": "alice@example.com",
  "createdAt": "2026-01-15T10:00:00Z"
}

// ✅ フィルタリング
GET /api/users?status=active&role=admin
レスポンス: 200 OK
{
  "data": [
    { "id": "1", "name": "Alice", "role": "admin" }
  ],
  "total": 1
}
```

**エラーレスポンス：**

```typescript
// リソースが見つからない
GET /api/users/999
レスポンス: 404 Not Found
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User with ID 999 not found"
  }
}

// 権限がない
GET /api/admin/users
レスポンス: 403 Forbidden
{
  "error": {
    "code": "FORBIDDEN",
    "message": "Insufficient permissions to access this resource"
  }
}
```

### POST - リソースの作成

**特徴：**
- 新しいリソースを作成する
- 非冪等（同じリクエストで複数のリソースが作成される可能性）
- リクエストボディを含む

**使用例：**

```typescript
// ✅ リソースの作成
POST /api/users
Content-Type: application/json

{
  "name": "Charlie",
  "email": "charlie@example.com",
  "password": "SecurePass123!"
}

レスポンス: 201 Created
Location: /api/users/124
{
  "id": "124",
  "name": "Charlie",
  "email": "charlie@example.com",
  "createdAt": "2026-01-28T12:00:00Z"
}
```

**ベストプラクティス：**

```typescript
// ✅ 作成されたリソースのURLを Location ヘッダーに含める
POST /api/articles
レスポンス: 201 Created
Location: /api/articles/456
{
  "id": "456",
  "title": "RESTful API Design",
  "slug": "restful-api-design",
  "createdAt": "2026-01-28T12:00:00Z"
}

// ✅ バリデーションエラー
POST /api/users
{
  "name": "",
  "email": "invalid-email"
}

レスポンス: 400 Bad Request
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      {
        "field": "name",
        "message": "Name is required"
      },
      {
        "field": "email",
        "message": "Email format is invalid"
      }
    ]
  }
}

// ✅ 重複エラー
POST /api/users
{
  "name": "Alice",
  "email": "alice@example.com"
}

レスポンス: 409 Conflict
{
  "error": {
    "code": "DUPLICATE_EMAIL",
    "message": "User with this email already exists"
  }
}
```

**POSTの特殊な用途：**

```typescript
// ✅ 複雑な検索（GETでURLが長くなりすぎる場合）
POST /api/users/search
{
  "filters": {
    "status": ["active", "pending"],
    "roles": ["admin", "editor"],
    "createdAfter": "2026-01-01",
    "tags": ["verified", "premium"]
  },
  "sort": {
    "field": "createdAt",
    "order": "desc"
  }
}

// ✅ アクション（リソースではない操作）
POST /api/users/123/resend-verification
レスポンス: 200 OK
{
  "message": "Verification email sent",
  "sentTo": "user@example.com"
}
```

### PUT - リソースの完全置換

**特徴：**
- リソース全体を置き換える
- 冪等性がある（同じリクエストを複数回実行しても結果は同じ）
- リクエストボディにリソースの完全な表現を含む

**使用例：**

```typescript
// ✅ リソース全体の更新
PUT /api/users/123
Content-Type: application/json

{
  "name": "Alice Updated",
  "email": "alice.new@example.com",
  "bio": "Software Engineer",
  "status": "active"
}

レスポンス: 200 OK
{
  "id": "123",
  "name": "Alice Updated",
  "email": "alice.new@example.com",
  "bio": "Software Engineer",
  "status": "active",
  "updatedAt": "2026-01-28T12:30:00Z"
}

// ✅ リソースが存在しない場合は作成（Upsert）
PUT /api/users/456
{
  "name": "New User",
  "email": "new@example.com"
}

レスポンス: 201 Created
Location: /api/users/456
{
  "id": "456",
  "name": "New User",
  "email": "new@example.com",
  "createdAt": "2026-01-28T12:30:00Z"
}
```

**重要な注意点：**

```typescript
// ❌ 部分的な更新は PUT では不適切
PUT /api/users/123
{
  "name": "Alice Updated"
  // email, bio, status が欠けている
}

レスポンス: 200 OK
{
  "id": "123",
  "name": "Alice Updated",
  "email": null,        // 既存の値が失われる
  "bio": null,          // 意図しない結果
  "status": null
}

// ✅ 完全な表現を送信
PUT /api/users/123
{
  "name": "Alice Updated",
  "email": "alice@example.com",    // 既存の値を保持
  "bio": "Software Engineer",
  "status": "active"
}
```

### PATCH - リソースの部分更新

**特徴：**
- リソースの一部のみを更新する
- 一般的に冪等だが、実装による
- 更新するフィールドのみをリクエストボディに含む

**使用例：**

```typescript
// ✅ 部分更新
PATCH /api/users/123
Content-Type: application/json

{
  "name": "Alice Updated"
}

レスポンス: 200 OK
{
  "id": "123",
  "name": "Alice Updated",          // 更新された
  "email": "alice@example.com",     // 既存の値が保持される
  "bio": "Software Engineer",       // 既存の値が保持される
  "status": "active"                // 既存の値が保持される
}

// ✅ 複数フィールドの更新
PATCH /api/users/123
{
  "name": "Alice Updated",
  "bio": "Senior Software Engineer"
}

// ✅ ネストされたフィールドの更新
PATCH /api/users/123
{
  "settings": {
    "notifications": {
      "email": false
    }
  }
}
```

**PUTとPATCHの使い分け：**

```typescript
// シナリオ: ユーザーのメールアドレスのみを更新したい

// ❌ PUT（すべてのフィールドが必要）
PUT /api/users/123
{
  "name": "Alice",
  "email": "alice.new@example.com",  // これだけ変更したい
  "bio": "Software Engineer",        // 変更しないが送信が必要
  "status": "active",                // 変更しないが送信が必要
  "role": "admin",                   // 変更しないが送信が必要
  "settings": { ... }                // 変更しないが送信が必要
}

// ✅ PATCH（変更するフィールドのみ）
PATCH /api/users/123
{
  "email": "alice.new@example.com"
}
```

**JSON Patch形式（高度な使用例）：**

```typescript
// RFC 6902 JSON Patch 形式
PATCH /api/users/123
Content-Type: application/json-patch+json

[
  {
    "op": "replace",
    "path": "/email",
    "value": "alice.new@example.com"
  },
  {
    "op": "add",
    "path": "/tags/-",
    "value": "verified"
  },
  {
    "op": "remove",
    "path": "/temporaryFlag"
  }
]
```

### DELETE - リソースの削除

**特徴：**
- リソースを削除する
- 冪等性がある（削除済みリソースの削除は通常エラーにならない）
- リクエストボディは通常含まない

**使用例：**

```typescript
// ✅ リソースの削除
DELETE /api/users/123

レスポンス: 204 No Content
（ボディなし）

// ✅ 削除されたリソース情報を返す
DELETE /api/users/123

レスポンス: 200 OK
{
  "id": "123",
  "name": "Alice",
  "deletedAt": "2026-01-28T13:00:00Z"
}

// ✅ 削除済みのリソースを再度削除
DELETE /api/users/123

レスポンス: 204 No Content
または
レスポンス: 404 Not Found
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User not found or already deleted"
  }
}
```

**論理削除 vs 物理削除：**

```typescript
// ✅ 論理削除（ソフトデリート）
DELETE /api/users/123

// 内部処理:
// UPDATE users SET deleted_at = NOW(), status = 'deleted' WHERE id = 123

レスポンス: 204 No Content

// 削除されたユーザーは取得できない
GET /api/users/123
レスポンス: 404 Not Found

// 管理者は削除されたユーザーを確認できる
GET /api/admin/users/123?includeDeleted=true
レスポンス: 200 OK
{
  "id": "123",
  "name": "Alice",
  "status": "deleted",
  "deletedAt": "2026-01-28T13:00:00Z"
}

// ✅ 物理削除（ハードデリート）
DELETE /api/users/123?permanent=true

// 内部処理:
// DELETE FROM users WHERE id = 123

レスポンス: 204 No Content
```

**一括削除：**

```typescript
// △ クエリパラメータによる削除（危険）
DELETE /api/users?status=inactive

// ✅ より安全な方法（明示的なエンドポイント）
POST /api/users/bulk-delete
{
  "ids": ["123", "124", "125"]
}

レスポンス: 200 OK
{
  "deleted": 3,
  "ids": ["123", "124", "125"]
}

// ✅ さらに安全な方法（確認トークン付き）
POST /api/users/bulk-delete
{
  "ids": ["123", "124", "125"],
  "confirmationToken": "abc123"
}
```

---

## ステータスコードの定義

HTTPステータスコードは、リクエストの結果を明確に伝えます。適切なステータスコードの使用は、APIの品質を大きく向上させます。

### 2xx 成功

**200 OK - リクエスト成功（レスポンスボディあり）**

```typescript
// GET リクエスト
GET /api/users/123
レスポンス: 200 OK
{
  "id": "123",
  "name": "Alice"
}

// POST リクエスト（アクション）
POST /api/users/123/resend-verification
レスポンス: 200 OK
{
  "message": "Email sent successfully"
}

// PUT リクエスト
PUT /api/users/123
{
  "name": "Alice Updated"
}
レスポンス: 200 OK
{
  "id": "123",
  "name": "Alice Updated",
  "updatedAt": "2026-01-28T14:00:00Z"
}

// PATCH リクエスト
PATCH /api/users/123
{
  "name": "Alice Updated"
}
レスポンス: 200 OK
{
  "id": "123",
  "name": "Alice Updated"
}
```

**201 Created - リソース作成成功**

```typescript
POST /api/users
{
  "name": "Bob",
  "email": "bob@example.com"
}

レスポンス: 201 Created
Location: /api/users/124
{
  "id": "124",
  "name": "Bob",
  "email": "bob@example.com",
  "createdAt": "2026-01-28T14:00:00Z"
}
```

**204 No Content - 成功（レスポンスボディなし）**

```typescript
// DELETE リクエスト
DELETE /api/users/123
レスポンス: 204 No Content
（ボディなし）

// PUT リクエスト（レスポンスが不要な場合）
PUT /api/users/123/settings
{
  "notifications": false
}
レスポンス: 204 No Content
```

**その他の2xxステータス：**

```typescript
// 202 Accepted - 非同期処理を受け付けた
POST /api/reports/generate
{
  "type": "monthly",
  "format": "pdf"
}

レスポンス: 202 Accepted
{
  "jobId": "job_789",
  "status": "processing",
  "estimatedCompletionTime": "2026-01-28T14:05:00Z",
  "statusUrl": "/api/jobs/job_789"
}
```

### 3xx リダイレクト

**301 Moved Permanently - 永続的なリダイレクト**

```typescript
// 旧エンドポイント
GET /api/v1/users

レスポンス: 301 Moved Permanently
Location: /api/v2/users
{
  "message": "This endpoint has been moved permanently to /api/v2/users"
}
```

**302 Found / 307 Temporary Redirect - 一時的なリダイレクト**

```typescript
GET /api/download/file123

レスポンス: 307 Temporary Redirect
Location: https://cdn.example.com/files/file123.pdf
```

**304 Not Modified - キャッシュが有効**

```typescript
GET /api/users/123
If-None-Match: "33a64df551425fcc55e4d42a148795d9f25f89d4"

レスポンス: 304 Not Modified
（ボディなし、キャッシュされたデータを使用）
```

### 4xx クライアントエラー

**400 Bad Request - リクエストが不正**

```typescript
// バリデーションエラー
POST /api/users
{
  "name": "",
  "email": "invalid-email"
}

レスポンス: 400 Bad Request
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "name",
        "message": "Name must not be empty"
      },
      {
        "field": "email",
        "message": "Email format is invalid"
      }
    ]
  }
}

// JSONパースエラー
POST /api/users
{ "name": "Alice", }  // 末尾カンマが不正

レスポンス: 400 Bad Request
{
  "error": {
    "code": "INVALID_JSON",
    "message": "Request body is not valid JSON"
  }
}
```

**401 Unauthorized - 認証が必要**

```typescript
GET /api/users/me

レスポンス: 401 Unauthorized
WWW-Authenticate: Bearer realm="api"
{
  "error": {
    "code": "AUTHENTICATION_REQUIRED",
    "message": "Authentication token is required"
  }
}

// トークン期限切れ
GET /api/users/me
Authorization: Bearer expired_token

レスポンス: 401 Unauthorized
{
  "error": {
    "code": "TOKEN_EXPIRED",
    "message": "Authentication token has expired",
    "expiredAt": "2026-01-28T10:00:00Z"
  }
}
```

**403 Forbidden - 権限がない**

```typescript
// 認証は成功したが、権限がない
DELETE /api/users/456
Authorization: Bearer valid_token

レスポンス: 403 Forbidden
{
  "error": {
    "code": "INSUFFICIENT_PERMISSIONS",
    "message": "You do not have permission to delete this user",
    "requiredRole": "admin",
    "currentRole": "user"
  }
}
```

**404 Not Found - リソースが見つからない**

```typescript
GET /api/users/999

レスポンス: 404 Not Found
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User with ID 999 not found"
  }
}

// エンドポイント自体が存在しない
GET /api/invalid-endpoint

レスポンス: 404 Not Found
{
  "error": {
    "code": "ENDPOINT_NOT_FOUND",
    "message": "The requested endpoint does not exist"
  }
}
```

**405 Method Not Allowed - HTTPメソッドが許可されていない**

```typescript
// DELETE がサポートされていないエンドポイント
DELETE /api/system/info

レスポンス: 405 Method Not Allowed
Allow: GET, HEAD
{
  "error": {
    "code": "METHOD_NOT_ALLOWED",
    "message": "DELETE method is not allowed for this endpoint",
    "allowedMethods": ["GET", "HEAD"]
  }
}
```

**409 Conflict - リソースの競合**

```typescript
// 重複したメールアドレス
POST /api/users
{
  "name": "Alice",
  "email": "existing@example.com"
}

レスポンス: 409 Conflict
{
  "error": {
    "code": "DUPLICATE_EMAIL",
    "message": "User with this email already exists",
    "conflictingField": "email"
  }
}

// 楽観的ロックの競合
PUT /api/users/123
If-Match: "old-etag"
{
  "name": "Alice Updated"
}

レスポンス: 409 Conflict
{
  "error": {
    "code": "RESOURCE_MODIFIED",
    "message": "Resource has been modified by another request",
    "currentETag": "new-etag"
  }
}
```

**422 Unprocessable Entity - 意味的なエラー**

```typescript
// リクエストは正しいが、ビジネスロジックで処理できない
POST /api/orders
{
  "productId": "123",
  "quantity": 100
}

レスポンス: 422 Unprocessable Entity
{
  "error": {
    "code": "INSUFFICIENT_STOCK",
    "message": "Insufficient stock to fulfill order",
    "requestedQuantity": 100,
    "availableQuantity": 50
  }
}
```

**429 Too Many Requests - レート制限超過**

```typescript
GET /api/users

レスポンス: 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1738072800
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "API rate limit exceeded",
    "limit": 100,
    "resetAt": "2026-01-28T15:00:00Z"
  }
}
```

### 5xx サーバーエラー

**500 Internal Server Error - サーバー内部エラー**

```typescript
GET /api/users/123

レスポンス: 500 Internal Server Error
{
  "error": {
    "code": "INTERNAL_SERVER_ERROR",
    "message": "An unexpected error occurred",
    "requestId": "req_abc123"  // トラブルシューティング用
  }
}

// 本番環境では詳細なエラー情報を隠す
// 開発環境でのみスタックトレースを返す
```

**502 Bad Gateway - ゲートウェイエラー**

```typescript
GET /api/users

レスポンス: 502 Bad Gateway
{
  "error": {
    "code": "BAD_GATEWAY",
    "message": "Upstream service is unavailable",
    "service": "user-service"
  }
}
```

**503 Service Unavailable - サービス利用不可**

```typescript
GET /api/users

レスポンス: 503 Service Unavailable
Retry-After: 120
{
  "error": {
    "code": "SERVICE_UNAVAILABLE",
    "message": "Service is temporarily unavailable due to maintenance",
    "estimatedRecoveryTime": "2026-01-28T16:00:00Z"
  }
}
```

**504 Gateway Timeout - ゲートウェイタイムアウト**

```typescript
GET /api/reports/complex

レスポンス: 504 Gateway Timeout
{
  "error": {
    "code": "GATEWAY_TIMEOUT",
    "message": "Request processing timed out",
    "timeout": 30000
  }
}
```

---

## クエリパラメータとパスパラメータ

パラメータの配置方法は、その意味と使用目的によって決定されます。

### パスパラメータ

パスパラメータは、リソースの識別に使用します。

**基本的な使用例：**

```typescript
// ✅ リソースの識別
GET /api/users/123              # ユーザーID 123
GET /api/articles/456           # 記事ID 456
GET /api/products/abc-123       # 商品コード abc-123

// ✅ 階層的なリソース
GET /api/users/123/articles     # ユーザー123の記事一覧
GET /api/articles/456/comments  # 記事456のコメント一覧
```

**パスパラメータの命名：**

```typescript
// ✅ 良い例（説明的な名前）
GET /api/users/:userId
GET /api/users/:userId/articles/:articleId

// ❌ 悪い例（汎用的すぎる）
GET /api/users/:id/articles/:id  # 同じ名前で混乱

// ✅ 改善例
GET /api/users/:userId/articles/:articleId
```

**TypeScriptでの型定義例：**

```typescript
// Express.jsの例
interface UserParams {
  userId: string;
}

interface ArticleParams {
  userId: string;
  articleId: string;
}

app.get('/api/users/:userId', (req: Request<UserParams>, res) => {
  const { userId } = req.params;
  // userId は型安全
});

app.get('/api/users/:userId/articles/:articleId',
  (req: Request<ArticleParams>, res) => {
    const { userId, articleId } = req.params;
    // 両方とも型安全
  }
);
```

### クエリパラメータ

クエリパラメータは、フィルタリング、ソート、ページング、検索など、リソースの取得方法をカスタマイズするために使用します。

**フィルタリング：**

```typescript
// ✅ 単一条件
GET /api/users?status=active
GET /api/articles?published=true

// ✅ 複数条件
GET /api/users?status=active&role=admin
GET /api/products?category=electronics&inStock=true

// ✅ 範囲指定
GET /api/products?minPrice=100&maxPrice=500
GET /api/articles?publishedAfter=2026-01-01&publishedBefore=2026-01-31

// ✅ 配列値
GET /api/users?roles=admin,editor,viewer
GET /api/users?roles[]=admin&roles[]=editor&roles[]=viewer

// ✅ ネストされた条件
GET /api/users?filter[status]=active&filter[role]=admin
```

**ソート：**

```typescript
// ✅ 単一フィールド
GET /api/users?sort=createdAt
GET /api/users?sortBy=createdAt&order=asc

// ✅ 降順（マイナスプレフィックス）
GET /api/users?sort=-createdAt

// ✅ 複数フィールド
GET /api/users?sort=lastName,firstName
GET /api/users?sort=-createdAt,name
```

**ページング：**

```typescript
// ✅ オフセットベース（小規模データセット）
GET /api/users?limit=20&offset=40
レスポンス:
{
  "data": [...],
  "pagination": {
    "limit": 20,
    "offset": 40,
    "total": 1000,
    "hasMore": true
  }
}

// ✅ ページベース（わかりやすい）
GET /api/users?page=3&perPage=20
レスポンス:
{
  "data": [...],
  "pagination": {
    "page": 3,
    "perPage": 20,
    "totalPages": 50,
    "totalItems": 1000
  }
}

// ✅ カーソルベース（大規模データセット、リアルタイム更新）
GET /api/users?cursor=eyJpZCI6MTIzfQ&limit=20
レスポンス:
{
  "data": [...],
  "pagination": {
    "nextCursor": "eyJpZCI6MTQzfQ",
    "hasMore": true
  }
}
```

**検索：**

```typescript
// ✅ 全文検索
GET /api/articles?q=REST+API
GET /api/users?search=john

// ✅ フィールド指定検索
GET /api/users?name=john&email=*@example.com
GET /api/articles?title=REST&author=Alice
```

**フィールド選択（Sparse Fieldsets）：**

```typescript
// ✅ 必要なフィールドのみ取得
GET /api/users?fields=id,name,email
レスポンス:
{
  "data": [
    {
      "id": "123",
      "name": "Alice",
      "email": "alice@example.com"
    }
  ]
}

// ✅ 関連リソースの展開
GET /api/articles?include=author,comments
レスポンス:
{
  "data": {
    "id": "456",
    "title": "RESTful API",
    "author": {
      "id": "123",
      "name": "Alice"
    },
    "comments": [...]
  }
}
```

**クエリパラメータの検証：**

```typescript
// TypeScriptでの型定義例
interface UserQueryParams {
  status?: 'active' | 'inactive' | 'pending';
  role?: 'admin' | 'editor' | 'viewer';
  sort?: string;
  limit?: number;
  offset?: number;
  search?: string;
}

// Zodを使用した検証例
import { z } from 'zod';

const userQuerySchema = z.object({
  status: z.enum(['active', 'inactive', 'pending']).optional(),
  role: z.enum(['admin', 'editor', 'viewer']).optional(),
  sort: z.string().optional(),
  limit: z.number().min(1).max(100).default(20),
  offset: z.number().min(0).default(0),
  search: z.string().optional(),
});

app.get('/api/users', (req, res) => {
  try {
    const params = userQuerySchema.parse(req.query);
    // 検証済みのパラメータを使用
  } catch (error) {
    res.status(400).json({
      error: {
        code: 'INVALID_QUERY_PARAMETERS',
        message: 'Query parameters validation failed',
        details: error.errors
      }
    });
  }
});
```

### パスパラメータ vs クエリパラメータの判断

**パスパラメータを使用する場合：**

```typescript
// ✅ リソースの識別（必須）
GET /api/users/123
GET /api/articles/456

// ✅ 階層構造（親子関係）
GET /api/users/123/articles
GET /api/articles/456/comments

// ❌ オプショナルな識別子（パスパラメータは不適切）
GET /api/users/123?includeDeleted=true  # 👈 クエリパラメータが適切
```

**クエリパラメータを使用する場合：**

```typescript
// ✅ フィルタリング（オプショナル）
GET /api/users?status=active

// ✅ ソート（オプショナル）
GET /api/users?sort=createdAt

// ✅ ページング（オプショナル）
GET /api/users?page=2&perPage=20

// ✅ 検索（オプショナル）
GET /api/users?search=john

// ❌ リソースの識別（クエリパラメータは不適切）
GET /api/users?id=123  # 👈 パスパラメータが適切
```

---

## リクエストとレスポンスの設計

### リクエストボディの設計

**基本原則：**

```typescript
// ✅ 良い例（明確な構造）
POST /api/users
{
  "name": "Alice",
  "email": "alice@example.com",
  "password": "SecurePass123!",
  "profile": {
    "bio": "Software Engineer",
    "location": "Tokyo"
  }
}

// ❌ 悪い例（フラットすぎる）
POST /api/users
{
  "name": "Alice",
  "email": "alice@example.com",
  "password": "SecurePass123!",
  "bio": "Software Engineer",
  "location": "Tokyo"
}

// ✅ 改善例（構造化）
POST /api/users
{
  "account": {
    "name": "Alice",
    "email": "alice@example.com",
    "password": "SecurePass123!"
  },
  "profile": {
    "bio": "Software Engineer",
    "location": "Tokyo"
  }
}
```

**命名規則：**

```typescript
// ✅ キャメルケース（推奨）
{
  "firstName": "Alice",
  "lastName": "Smith",
  "emailAddress": "alice@example.com",
  "phoneNumber": "+81-90-1234-5678"
}

// △ スネークケース（Pythonバックエンドで一般的）
{
  "first_name": "Alice",
  "last_name": "Smith",
  "email_address": "alice@example.com",
  "phone_number": "+81-90-1234-5678"
}

// ❌ ケバブケース（JSONでは使用しない）
{
  "first-name": "Alice",      // プロパティアクセスが困難
  "last-name": "Smith"
}

// ❌ 混在（一貫性がない）
{
  "firstName": "Alice",
  "last_name": "Smith",        // NG: キャメルケースと混在
  "email-address": "alice@example.com"  // NG: ケバブケースと混在
}
```

**データ型の選択：**

```typescript
// ✅ 適切なデータ型
{
  "id": "user_123",              // string: UUID や識別子
  "name": "Alice",               // string: テキスト
  "age": 30,                     // number: 整数
  "balance": 1234.56,            // number: 小数
  "isActive": true,              // boolean: フラグ
  "tags": ["admin", "verified"], // array: リスト
  "metadata": { ... },           // object: ネストされたデータ
  "createdAt": "2026-01-28T12:00:00Z",  // string: ISO 8601形式の日時
  "deletedAt": null              // null: 値がない
}

// ❌ 不適切なデータ型
{
  "id": 123,                     // NG: IDを数値にすると桁あふれのリスク
  "age": "30",                   // NG: 数値を文字列にすると計算が困難
  "isActive": "true",            // NG: ブール値を文字列にすると判定が複雑
  "createdAt": 1738072800000     // NG: Unixタイムスタンプは可読性が低い
}
```

**日時の表現：**

```typescript
// ✅ ISO 8601形式（推奨）
{
  "createdAt": "2026-01-28T12:00:00Z",           // UTC
  "updatedAt": "2026-01-28T12:30:00+09:00",      // タイムゾーン付き
  "scheduledAt": "2026-01-28T15:00:00.000Z"      // ミリ秒付き
}

// △ Unixタイムスタンプ（可読性が低い）
{
  "createdAt": 1738072800
}

// ❌ 独自フォーマット（避ける）
{
  "createdAt": "28/01/2026 12:00:00",  // 地域によって解釈が異なる
  "createdAt": "2026-01-28 12:00:00"   // タイムゾーン情報がない
}
```

### レスポンスボディの設計

**基本構造：**

```typescript
// ✅ シンプルなリソース
GET /api/users/123
{
  "id": "123",
  "name": "Alice",
  "email": "alice@example.com",
  "createdAt": "2026-01-15T10:00:00Z"
}

// ✅ リスト（データとメタデータを分離）
GET /api/users
{
  "data": [
    { "id": "123", "name": "Alice" },
    { "id": "124", "name": "Bob" }
  ],
  "pagination": {
    "page": 1,
    "perPage": 20,
    "total": 2,
    "totalPages": 1
  }
}

// ❌ 悪い例（データとメタデータが混在）
GET /api/users
{
  "users": [...],
  "page": 1,
  "perPage": 20,
  "total": 2
}
```

**エラーレスポンスの統一：**

```typescript
// ✅ 統一されたエラー構造
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "message": "Email format is invalid",
        "value": "invalid-email"
      }
    ],
    "requestId": "req_abc123",
    "timestamp": "2026-01-28T12:00:00Z"
  }
}

// エラーコードのパターン
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",      // リソースが見つからない
    "code": "UNAUTHORIZED",            // 認証エラー
    "code": "FORBIDDEN",               // 権限エラー
    "code": "VALIDATION_ERROR",        // バリデーションエラー
    "code": "RATE_LIMIT_EXCEEDED",     // レート制限
    "code": "INTERNAL_SERVER_ERROR"    // サーバーエラー
  }
}
```

**ネストされたリソース：**

```typescript
// ✅ シャロー（参照のみ）
GET /api/articles/456
{
  "id": "456",
  "title": "RESTful API Design",
  "authorId": "123",               // 👈 IDのみ
  "createdAt": "2026-01-28T12:00:00Z"
}

// ✅ 埋め込み（詳細情報）
GET /api/articles/456?include=author
{
  "id": "456",
  "title": "RESTful API Design",
  "author": {                      // 👈 完全なオブジェクト
    "id": "123",
    "name": "Alice",
    "email": "alice@example.com"
  },
  "createdAt": "2026-01-28T12:00:00Z"
}

// ✅ ハイパーメディア（リンク）
GET /api/articles/456
{
  "id": "456",
  "title": "RESTful API Design",
  "author": {
    "id": "123",
    "name": "Alice",
    "links": {
      "self": "/api/users/123",
      "articles": "/api/users/123/articles"
    }
  },
  "links": {
    "self": "/api/articles/456",
    "author": "/api/users/123",
    "comments": "/api/articles/456/comments"
  }
}
```

**HATEOAS（Hypermedia as the Engine of Application State）：**

```typescript
// ✅ レスポンスに次のアクションを含める
GET /api/orders/789
{
  "id": "789",
  "status": "pending",
  "total": 5000,
  "items": [...],
  "links": {
    "self": "/api/orders/789",
    "cancel": {
      "href": "/api/orders/789/cancel",
      "method": "POST"
    },
    "pay": {
      "href": "/api/orders/789/pay",
      "method": "POST"
    }
  }
}

// 注文が支払い済みの場合
GET /api/orders/789
{
  "id": "789",
  "status": "paid",
  "total": 5000,
  "items": [...],
  "links": {
    "self": "/api/orders/789",
    "invoice": {
      "href": "/api/orders/789/invoice",
      "method": "GET"
    }
    // 👈 "cancel" と "pay" は表示されない（実行不可能なアクション）
  }
}
```

---

## API文書化のベストプラクティス

### エンドポイントの説明

各エンドポイントには、以下の情報を含めます：

**基本情報：**

```markdown
## ユーザー一覧の取得

### エンドポイント
GET /api/v1/users

### 説明
登録されているユーザーの一覧を取得します。フィルタリング、ソート、ページングをサポートします。

### 認証
必要（Bearer Token）

### レート制限
100 リクエスト / 分
```

**パラメータの説明：**

```markdown
### クエリパラメータ

| パラメータ | 型 | 必須 | デフォルト | 説明 |
|-----------|-----|------|-----------|------|
| status | string | × | - | ユーザーステータス（`active`, `inactive`, `pending`） |
| role | string | × | - | ユーザーロール（`admin`, `editor`, `viewer`） |
| sort | string | × | `createdAt` | ソートフィールド（`-` で降順） |
| page | number | × | 1 | ページ番号（1から開始） |
| perPage | number | × | 20 | 1ページあたりのアイテム数（最大100） |
| search | string | × | - | 名前またはメールで検索 |

### パスパラメータ

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| userId | string | ○ | ユーザーID（UUID形式） |
```

**リクエスト例：**

```markdown
### リクエスト例

#### 基本的な取得
\`\`\`bash
curl -X GET "https://api.example.com/api/v1/users" \
  -H "Authorization: Bearer YOUR_TOKEN"
\`\`\`

#### フィルタリングとソート
\`\`\`bash
curl -X GET "https://api.example.com/api/v1/users?status=active&sort=-createdAt&page=1&perPage=20" \
  -H "Authorization: Bearer YOUR_TOKEN"
\`\`\`

#### 検索
\`\`\`bash
curl -X GET "https://api.example.com/api/v1/users?search=alice" \
  -H "Authorization: Bearer YOUR_TOKEN"
\`\`\`
```

**レスポンス例：**

```markdown
### レスポンス例

#### 成功（200 OK）
\`\`\`json
{
  "data": [
    {
      "id": "123",
      "name": "Alice",
      "email": "alice@example.com",
      "role": "admin",
      "status": "active",
      "createdAt": "2026-01-15T10:00:00Z",
      "updatedAt": "2026-01-28T12:00:00Z"
    },
    {
      "id": "124",
      "name": "Bob",
      "email": "bob@example.com",
      "role": "editor",
      "status": "active",
      "createdAt": "2026-01-20T14:00:00Z",
      "updatedAt": "2026-01-28T12:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "perPage": 20,
    "total": 2,
    "totalPages": 1
  }
}
\`\`\`

#### エラー（401 Unauthorized）
\`\`\`json
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Authentication token is required or invalid"
  }
}
\`\`\`

#### エラー（400 Bad Request）
\`\`\`json
{
  "error": {
    "code": "INVALID_QUERY_PARAMETER",
    "message": "Invalid query parameter value",
    "details": [
      {
        "parameter": "status",
        "message": "Must be one of: active, inactive, pending",
        "received": "invalid_status"
      }
    ]
  }
}
\`\`\`
```

### リファレンスドキュメントのテンプレート

**完全なエンドポイント文書化の例：**

```markdown
# ユーザーAPI

## ユーザー一覧の取得

**エンドポイント:** `GET /api/v1/users`

**説明:** 登録されているユーザーの一覧を取得します。

**認証:** 必要（Bearer Token）

**権限:** `users:read`

**レート制限:** 100 リクエスト / 分

### クエリパラメータ

| パラメータ | 型 | 必須 | デフォルト | 説明 |
|-----------|-----|------|-----------|------|
| status | string | × | - | `active`, `inactive`, `pending` |
| page | number | × | 1 | ページ番号 |
| perPage | number | × | 20 | 1ページあたりのアイテム数（最大100） |

### リクエスト例

\`\`\`bash
curl -X GET "https://api.example.com/api/v1/users?status=active&page=1" \
  -H "Authorization: Bearer YOUR_TOKEN"
\`\`\`

### レスポンス

#### 成功（200 OK）

\`\`\`json
{
  "data": [
    {
      "id": "123",
      "name": "Alice",
      "email": "alice@example.com",
      "status": "active"
    }
  ],
  "pagination": {
    "page": 1,
    "perPage": 20,
    "total": 1,
    "totalPages": 1
  }
}
\`\`\`

#### エラー

| ステータスコード | 説明 |
|----------------|------|
| 400 Bad Request | クエリパラメータが不正 |
| 401 Unauthorized | 認証が必要 |
| 403 Forbidden | 権限がない |
| 429 Too Many Requests | レート制限超過 |
| 500 Internal Server Error | サーバーエラー |

---

## ユーザーの作成

**エンドポイント:** `POST /api/v1/users`

**説明:** 新しいユーザーを作成します。

**認証:** 必要（Bearer Token）

**権限:** `users:write`

### リクエストボディ

\`\`\`typescript
{
  name: string;         // 1-100文字
  email: string;        // 有効なメールアドレス
  password: string;     // 8文字以上、英数字記号を含む
  role?: string;        // "admin" | "editor" | "viewer"（デフォルト: "viewer"）
}
\`\`\`

### リクエスト例

\`\`\`bash
curl -X POST "https://api.example.com/api/v1/users" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Charlie",
    "email": "charlie@example.com",
    "password": "SecurePass123!",
    "role": "editor"
  }'
\`\`\`

### レスポンス

#### 成功（201 Created）

\`\`\`json
{
  "id": "125",
  "name": "Charlie",
  "email": "charlie@example.com",
  "role": "editor",
  "status": "pending",
  "createdAt": "2026-01-28T15:00:00Z"
}
\`\`\`

#### エラー（400 Bad Request）

\`\`\`json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      {
        "field": "email",
        "message": "Email format is invalid"
      }
    ]
  }
}
\`\`\`

#### エラー（409 Conflict）

\`\`\`json
{
  "error": {
    "code": "DUPLICATE_EMAIL",
    "message": "User with this email already exists"
  }
}
\`\`\`
```

---

## 実践例：ブログAPIの設計

実際のユースケースとして、ブログAPIを設計します。

### リソース構造

```
ユーザー (Users)
├─ 記事 (Articles)
│  ├─ コメント (Comments)
│  │  └─ 返信 (Replies)
│  └─ タグ (Tags)
└─ プロフィール (Profile)
```

### エンドポイント一覧

**ユーザー関連：**

```typescript
// ユーザー管理
POST   /api/v1/users                    # ユーザー登録
GET    /api/v1/users                    # ユーザー一覧
GET    /api/v1/users/:userId            # ユーザー詳細
PUT    /api/v1/users/:userId            # ユーザー更新
DELETE /api/v1/users/:userId            # ユーザー削除

// 認証
POST   /api/v1/auth/login               # ログイン
POST   /api/v1/auth/logout              # ログアウト
POST   /api/v1/auth/refresh             # トークン更新
POST   /api/v1/auth/forgot-password     # パスワードリセット要求
POST   /api/v1/auth/reset-password      # パスワードリセット

// プロフィール
GET    /api/v1/profile                  # 現在のユーザーのプロフィール
PUT    /api/v1/profile                  # プロフィール更新
```

**記事関連：**

```typescript
// 記事管理
GET    /api/v1/articles                 # 記事一覧
POST   /api/v1/articles                 # 記事作成
GET    /api/v1/articles/:articleId      # 記事詳細
PUT    /api/v1/articles/:articleId      # 記事更新
PATCH  /api/v1/articles/:articleId      # 記事部分更新
DELETE /api/v1/articles/:articleId      # 記事削除

// ユーザーの記事
GET    /api/v1/users/:userId/articles   # 特定ユーザーの記事一覧

// 記事の公開状態
POST   /api/v1/articles/:articleId/publish    # 記事を公開
POST   /api/v1/articles/:articleId/unpublish  # 記事を非公開
```

**コメント関連：**

```typescript
// コメント管理
GET    /api/v1/articles/:articleId/comments           # コメント一覧
POST   /api/v1/articles/:articleId/comments           # コメント作成
GET    /api/v1/comments/:commentId                    # コメント詳細
PUT    /api/v1/comments/:commentId                    # コメント更新
DELETE /api/v1/comments/:commentId                    # コメント削除

// 返信
GET    /api/v1/comments/:commentId/replies            # 返信一覧
POST   /api/v1/comments/:commentId/replies            # 返信作成
```

**タグ関連：**

```typescript
// タグ管理
GET    /api/v1/tags                     # タグ一覧
POST   /api/v1/tags                     # タグ作成
GET    /api/v1/tags/:tagId              # タグ詳細
PUT    /api/v1/tags/:tagId              # タグ更新
DELETE /api/v1/tags/:tagId              # タグ削除

// タグ別記事
GET    /api/v1/tags/:tagId/articles     # タグ別記事一覧
```

### 詳細な設計例

**記事一覧の取得：**

```typescript
// エンドポイント
GET /api/v1/articles

// クエリパラメータ
interface ArticleQueryParams {
  // フィルタリング
  status?: 'draft' | 'published' | 'archived';
  authorId?: string;
  tag?: string;
  publishedAfter?: string;  // ISO 8601形式
  publishedBefore?: string;

  // 検索
  q?: string;               // タイトルまたは本文で検索

  // ソート
  sort?: 'createdAt' | '-createdAt' | 'publishedAt' | '-publishedAt' | 'title';

  // ページング
  page?: number;
  perPage?: number;

  // 関連リソース
  include?: 'author' | 'tags' | 'comments' | 'author,tags';
}

// リクエスト例
GET /api/v1/articles?status=published&sort=-publishedAt&page=1&perPage=10&include=author,tags

// レスポンス
{
  "data": [
    {
      "id": "article_123",
      "title": "RESTful API設計ガイド",
      "slug": "restful-api-design-guide",
      "excerpt": "RESTful APIの設計原則と実践的なテクニックを解説します。",
      "content": "...",
      "status": "published",
      "author": {
        "id": "user_456",
        "name": "Alice",
        "email": "alice@example.com",
        "avatar": "https://cdn.example.com/avatars/user_456.jpg"
      },
      "tags": [
        {
          "id": "tag_1",
          "name": "API",
          "slug": "api"
        },
        {
          "id": "tag_2",
          "name": "REST",
          "slug": "rest"
        }
      ],
      "viewCount": 1234,
      "likeCount": 56,
      "commentCount": 12,
      "publishedAt": "2026-01-28T10:00:00Z",
      "createdAt": "2026-01-25T15:00:00Z",
      "updatedAt": "2026-01-28T09:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "perPage": 10,
    "total": 100,
    "totalPages": 10,
    "hasNext": true,
    "hasPrev": false
  },
  "links": {
    "self": "/api/v1/articles?page=1&perPage=10",
    "first": "/api/v1/articles?page=1&perPage=10",
    "last": "/api/v1/articles?page=10&perPage=10",
    "next": "/api/v1/articles?page=2&perPage=10",
    "prev": null
  }
}
```

**記事の作成：**

```typescript
// エンドポイント
POST /api/v1/articles

// リクエストボディ
{
  "title": "RESTful API設計ガイド",
  "content": "RESTful APIの設計原則と実践的なテクニックを解説します...",
  "excerpt": "RESTful APIの設計原則と実践的なテクニックを解説します。",
  "tags": ["api", "rest", "web-development"],
  "status": "draft",  // "draft" | "published"
  "publishedAt": null  // 公開日時（nullの場合は即時公開）
}

// レスポンス（201 Created）
{
  "id": "article_789",
  "title": "RESTful API設計ガイド",
  "slug": "restful-api-design-guide",
  "excerpt": "RESTful APIの設計原則と実践的なテクニックを解説します。",
  "content": "RESTful APIの設計原則と実践的なテクニックを解説します...",
  "status": "draft",
  "author": {
    "id": "user_456",
    "name": "Alice"
  },
  "tags": [
    { "id": "tag_1", "name": "API", "slug": "api" },
    { "id": "tag_2", "name": "REST", "slug": "rest" },
    { "id": "tag_3", "name": "Web Development", "slug": "web-development" }
  ],
  "viewCount": 0,
  "likeCount": 0,
  "commentCount": 0,
  "publishedAt": null,
  "createdAt": "2026-01-28T16:00:00Z",
  "updatedAt": "2026-01-28T16:00:00Z"
}
```

**記事の更新：**

```typescript
// 完全更新（PUT）
PUT /api/v1/articles/article_789
{
  "title": "RESTful API設計完全ガイド",  // 更新
  "content": "...",
  "excerpt": "...",
  "tags": ["api", "rest"],
  "status": "draft"
}

// 部分更新（PATCH）
PATCH /api/v1/articles/article_789
{
  "title": "RESTful API設計完全ガイド",  // タイトルのみ更新
  "status": "published"                  // ステータスのみ更新
}

// レスポンス（200 OK）
{
  "id": "article_789",
  "title": "RESTful API設計完全ガイド",
  "slug": "restful-api-design-complete-guide",
  // ... その他のフィールド
  "updatedAt": "2026-01-28T16:30:00Z"
}
```

**コメントの作成：**

```typescript
// エンドポイント
POST /api/v1/articles/article_789/comments

// リクエストボディ
{
  "content": "とても参考になりました。ありがとうございます！"
}

// レスポンス（201 Created）
{
  "id": "comment_001",
  "articleId": "article_789",
  "content": "とても参考になりました。ありがとうございます！",
  "author": {
    "id": "user_123",
    "name": "Bob",
    "avatar": "https://cdn.example.com/avatars/user_123.jpg"
  },
  "likeCount": 0,
  "replyCount": 0,
  "createdAt": "2026-01-28T17:00:00Z",
  "updatedAt": "2026-01-28T17:00:00Z"
}
```

---

## チェックリスト

この章で学んだことを確認しましょう：

### RESTful APIの原則
- [ ] リソース指向設計を理解し、実践できる
- [ ] 統一インターフェース（HTTPメソッド）を適切に選択できる
- [ ] ステートレス性の意味と利点を理解している
- [ ] 冪等性の概念を理解し、設計に反映できる

### エンドポイント設計
- [ ] 一貫性のある命名規則を適用できる（名詞、複数形、ケバブケース、小文字）
- [ ] リソースの階層構造を適切に表現できる（深すぎない階層）
- [ ] クエリパラメータとパスパラメータを適切に使い分けられる
- [ ] フィルタリング、ソート、ページングを実装できる
- [ ] バージョニング戦略を選択し、適用できる

### HTTPメソッド
- [ ] GET、POST、PUT、PATCH、DELETEの使い分けを理解している
- [ ] 各メソッドの冪等性を考慮した設計ができる
- [ ] PUTとPATCHの違いを理解し、適切に選択できる

### ステータスコード
- [ ] 2xx（成功）、4xx（クライアントエラー）、5xx（サーバーエラー）を適切に使用できる
- [ ] 200、201、204の使い分けができる
- [ ] 400、401、403、404の違いを理解している
- [ ] エラーレスポンスに適切な情報を含められる

### リクエスト・レスポンス設計
- [ ] リクエストボディを明確に構造化できる
- [ ] 一貫性のある命名規則（キャメルケース）を適用できる
- [ ] 適切なデータ型を選択できる（文字列、数値、ブール値、配列、オブジェクト）
- [ ] 日時をISO 8601形式で表現できる
- [ ] レスポンスボディを適切に設計できる（データとメタデータの分離）
- [ ] エラーレスポンスを統一された形式で提供できる

### API文書化
- [ ] エンドポイントの説明に必要な情報を含められる
- [ ] パラメータを明確に文書化できる
- [ ] リクエスト例とレスポンス例を提供できる
- [ ] エラーケースを網羅的に文書化できる

---

## 次のステップ

この章では、RESTful APIの原則に基づいたエンドポイント設計と文書化の方法を学びました。エンドポイントの設計が完了したら、次のステップとして以下を学習します：

**第10章: リクエスト・レスポンス例**
- より詳細なリクエスト・レスポンスの例
- バリデーションルールの明記
- エラーハンドリングのベストプラクティス
- 実際に動作するサンプルコードの提供

**第11章: OpenAPI/Swagger活用**
- OpenAPI Specificationによる仕様書の自動化
- Swagger UIを使用した対話的なドキュメント
- コードからの自動生成とその逆
- API開発ワークフローの効率化

エンドポイント設計は、APIの品質を決定する重要な要素です。この章で学んだ原則とベストプラクティスを実践することで、直感的で使いやすく、長期的に保守可能なAPIを構築できます。

### 関連リソース

**公式ドキュメント：**
- [RFC 7231 - HTTP/1.1 Semantics and Content](https://www.rfc-editor.org/rfc/rfc7231)
- [RFC 7230 - HTTP/1.1 Message Syntax and Routing](https://www.rfc-editor.org/rfc/rfc7230)
- [REST API Design Rulebook by Mark Masse (O'Reilly)](https://www.oreilly.com/library/view/rest-api-design/9781449317904/)

**参考記事：**
- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)
- [Google API Design Guide](https://cloud.google.com/apis/design)
- [Stripe API Documentation](https://stripe.com/docs/api)（優れたAPI設計の実例）

**ツール：**
- [OpenAPI Generator](https://openapi-generator.tech/)
- [Swagger Editor](https://editor.swagger.io/)
- [Postman](https://www.postman.com/)（APIのテストと文書化）
