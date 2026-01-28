---
title: "第10章 リクエスト・レスポンス例"
---

# 第10章 リクエスト・レスポンス例

## この章で学ぶこと

API仕様書の中で、**最も重要なのはリクエスト・レスポンス例**です。エンドポイントの説明がどれだけ詳細でも、実際に動く例がなければ、開発者は使い方を理解できません。

この章では、以下を学びます：

- わかりやすいリクエスト例の書き方
- 実用的なレスポンス例の作成方法
- エラーレスポンスの文書化
- バリデーションルールの明記
- データ型とフォーマットの説明
- 実際のユースケースに基づいた例

**最も重要な原則：** リクエスト・レスポンス例は、**コピー&ペーストで動作するべき**です。開発者が実際に試せなければ、ドキュメントとしての価値は半減します。

---

## なぜリクエスト・レスポンス例が重要なのか

### 開発者が最も見る部分

API仕様書を読む開発者の行動パターンは、一般的に以下のようになります：

1. エンドポイント一覧を確認
2. **必要なエンドポイントのリクエスト例を見る**
3. **レスポンス例を確認**
4. 実装を開始
5. エラーが出たらエラーレスポンス例を見る

つまり、**リクエスト・レスポンス例は、最も頻繁に参照される部分**です。

### ❌ 悪い例：説明だけでは不十分

```markdown
### POST /api/users

新しいユーザーを作成します。

**パラメータ:**
- name: ユーザー名（必須）
- email: メールアドレス（必須）
- role: ロール（任意）

**レスポンス:**
作成されたユーザーオブジェクトを返します。
```

**何が問題か:**
- 実際のリクエスト形式がわからない
- レスポンスの構造が不明
- データ型が不明
- バリデーションルールが不明
- エラーケースがわからない

**開発者の反応:**
- 「JSONの形式は？」
- 「roleには何が入るの？」
- 「emailのバリデーションは？」
- 「エラー時のレスポンスは？」

結果として、**チャットやSlackで質問が来る**ことになります。

### ✅ 良い例：実際に動く例を示す

```markdown
### POST /api/users

新しいユーザーを作成します。

#### リクエスト例

```bash
curl -X POST https://api.example.com/v1/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "name": "田中太郎",
    "email": "tanaka@example.com",
    "role": "member"
  }'
```

```json
{
  "name": "田中太郎",
  "email": "tanaka@example.com",
  "role": "member"
}
```

#### レスポンス例

**成功時（201 Created）:**

```json
{
  "id": "usr_2Nq8xQp1jK9L",
  "name": "田中太郎",
  "email": "tanaka@example.com",
  "role": "member",
  "status": "active",
  "createdAt": "2026-01-28T12:00:00Z",
  "updatedAt": "2026-01-28T12:00:00Z"
}
```

**エラー時（400 Bad Request）:**

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "リクエストパラメータが不正です",
    "details": [
      {
        "field": "email",
        "message": "有効なメールアドレスを入力してください"
      }
    ]
  }
}
```

#### パラメータ

| フィールド | 型 | 必須 | 説明 | バリデーション |
|-----------|-----|------|------|---------------|
| name | string | ✅ | ユーザー名 | 1-50文字 |
| email | string | ✅ | メールアドレス | RFC 5322準拠 |
| role | string | ❌ | ロール | `admin`, `member`, `viewer`のいずれか（デフォルト: `member`） |

#### レスポンスフィールド

| フィールド | 型 | 説明 |
|-----------|-----|------|
| id | string | ユーザーID（`usr_`プレフィックス付き） |
| name | string | ユーザー名 |
| email | string | メールアドレス |
| role | string | ロール |
| status | string | アカウント状態（`active`, `suspended`, `deleted`） |
| createdAt | string | 作成日時（ISO 8601形式） |
| updatedAt | string | 更新日時（ISO 8601形式） |
```

**何が優れているか:**
- コピー&ペーストで動くcurlコマンド
- 実際のJSON形式が明確
- 成功時とエラー時の両方を示している
- データ型とバリデーションルールが明記されている
- レスポンスの各フィールドが説明されている

---

## リクエスト例の書き方

### 基本構成

リクエスト例は、以下の要素を含むべきです：

1. **curlコマンド例**（すぐに試せる）
2. **JSONボディ例**（見やすい形式で）
3. **パラメータ説明**（型、必須/任意、バリデーション）
4. **認証情報の扱い**（どこにトークンを入れるか）

### curlコマンドの書き方

#### ✅ 推奨形式

```bash
curl -X POST https://api.example.com/v1/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "name": "田中太郎",
    "email": "tanaka@example.com"
  }'
```

**ポイント:**
- 改行してオプションを見やすく配置
- ヘッダーを明示
- JSONを整形して記載
- APIキーはプレースホルダー（`YOUR_API_KEY`）を使用

#### ❌ 避けるべき形式

```bash
# 悪い例1: 1行で長すぎる
curl -X POST https://api.example.com/v1/users -H "Content-Type: application/json" -H "Authorization: Bearer YOUR_API_KEY" -d '{"name":"田中太郎","email":"tanaka@example.com"}'

# 悪い例2: 実際のAPIキーを記載
curl -X POST https://api.example.com/v1/users \
  -H "Authorization: Bearer sk_live_1234567890abcdef"

# 悪い例3: JSONが圧縮されていて読めない
curl -X POST https://api.example.com/v1/users \
  -d '{"name":"田中太郎","email":"tanaka@example.com","role":"member","preferences":{"language":"ja","timezone":"Asia/Tokyo"}}'
```

### 複数の言語での例

開発者が使用する言語は様々です。可能であれば、複数の言語での例を提供すると親切です。

#### JavaScript/TypeScript

```typescript
// fetch APIを使用
const response = await fetch('https://api.example.com/v1/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer YOUR_API_KEY'
  },
  body: JSON.stringify({
    name: '田中太郎',
    email: 'tanaka@example.com',
    role: 'member'
  })
});

const data = await response.json();
console.log(data);
```

#### Python

```python
import requests

response = requests.post(
    'https://api.example.com/v1/users',
    headers={
        'Content-Type': 'application/json',
        'Authorization': 'Bearer YOUR_API_KEY'
    },
    json={
        'name': '田中太郎',
        'email': 'tanaka@example.com',
        'role': 'member'
    }
)

data = response.json()
print(data)
```

#### Swift

```swift
struct CreateUserRequest: Codable {
    let name: String
    let email: String
    let role: String
}

let request = CreateUserRequest(
    name: "田中太郎",
    email: "tanaka@example.com",
    role: "member"
)

var urlRequest = URLRequest(url: URL(string: "https://api.example.com/v1/users")!)
urlRequest.httpMethod = "POST"
urlRequest.addValue("application/json", forHTTPHeaderField: "Content-Type")
urlRequest.addValue("Bearer YOUR_API_KEY", forHTTPHeaderField: "Authorization")
urlRequest.httpBody = try JSONEncoder().encode(request)

let (data, response) = try await URLSession.shared.data(for: urlRequest)
let user = try JSONDecoder().decode(User.self, from: data)
```

### パラメータの詳細な説明

パラメータは、**表形式**で説明すると見やすくなります。

#### 基本的な表形式

| フィールド | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| name | string | ✅ | ユーザー名 |
| email | string | ✅ | メールアドレス |
| role | string | ❌ | ロール（デフォルト: `member`） |

#### より詳細な表形式

| フィールド | 型 | 必須 | デフォルト | 説明 | バリデーション | 例 |
|-----------|-----|------|----------|------|---------------|-----|
| name | string | ✅ | - | ユーザー名 | 1-50文字 | `"田中太郎"` |
| email | string | ✅ | - | メールアドレス | RFC 5322準拠 | `"tanaka@example.com"` |
| role | string | ❌ | `"member"` | ロール | `admin`, `member`, `viewer`のいずれか | `"member"` |
| phoneNumber | string | ❌ | `null` | 電話番号 | E.164形式 | `"+81-90-1234-5678"` |
| birthDate | string | ❌ | `null` | 生年月日 | ISO 8601形式（日付のみ） | `"1990-05-15"` |
| tags | string[] | ❌ | `[]` | タグリスト | 各タグは1-20文字、最大10個 | `["premium", "early-adopter"]` |

### ネストされたオブジェクトの説明

複雑なオブジェクト構造の場合は、ドット記法で説明します。

#### リクエスト例

```json
{
  "name": "田中太郎",
  "email": "tanaka@example.com",
  "address": {
    "zipCode": "100-0001",
    "prefecture": "東京都",
    "city": "千代田区",
    "line1": "千代田1-1",
    "line2": "パレスサイドビル 5F"
  },
  "preferences": {
    "language": "ja",
    "timezone": "Asia/Tokyo",
    "notifications": {
      "email": true,
      "push": false,
      "sms": false
    }
  }
}
```

#### パラメータ説明

| フィールド | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| name | string | ✅ | ユーザー名 |
| email | string | ✅ | メールアドレス |
| address | object | ❌ | 住所情報 |
| address.zipCode | string | ✅ | 郵便番号（ハイフン含む） |
| address.prefecture | string | ✅ | 都道府県 |
| address.city | string | ✅ | 市区町村 |
| address.line1 | string | ✅ | 住所1行目 |
| address.line2 | string | ❌ | 住所2行目（建物名、部屋番号など） |
| preferences | object | ❌ | ユーザー設定 |
| preferences.language | string | ❌ | 言語コード（ISO 639-1）デフォルト: `"ja"` |
| preferences.timezone | string | ❌ | タイムゾーン（IANA形式）デフォルト: `"Asia/Tokyo"` |
| preferences.notifications | object | ❌ | 通知設定 |
| preferences.notifications.email | boolean | ❌ | メール通知の有効/無効 デフォルト: `true` |
| preferences.notifications.push | boolean | ❌ | プッシュ通知の有効/無効 デフォルト: `false` |
| preferences.notifications.sms | boolean | ❌ | SMS通知の有効/無効 デフォルト: `false` |

### 配列パラメータの説明

#### リクエスト例

```json
{
  "name": "プロジェクトX",
  "members": [
    {
      "userId": "usr_2Nq8xQp1jK9L",
      "role": "owner"
    },
    {
      "userId": "usr_3Mp7yRo2kL8M",
      "role": "editor"
    }
  ],
  "tags": ["urgent", "client-request", "q1-2026"]
}
```

#### パラメータ説明

| フィールド | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| name | string | ✅ | プロジェクト名 |
| members | array | ✅ | メンバーリスト（最低1人、最大50人） |
| members[].userId | string | ✅ | ユーザーID |
| members[].role | string | ✅ | 権限（`owner`, `editor`, `viewer`のいずれか） |
| tags | string[] | ❌ | タグリスト（最大20個） |

---

## レスポンス例の書き方

### 成功レスポンスの基本

成功レスポンスには、以下を含めるべきです：

1. **HTTPステータスコード**
2. **レスポンスボディ**（整形済みJSON）
3. **各フィールドの説明**
4. **レスポンスヘッダー**（重要な場合）

### CRUD操作別のレスポンス例

#### CREATE（POST）- 201 Created

```http
POST /api/v1/users
```

**リクエスト:**

```json
{
  "name": "田中太郎",
  "email": "tanaka@example.com"
}
```

**レスポンス（201 Created）:**

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/v1/users/usr_2Nq8xQp1jK9L
```

```json
{
  "id": "usr_2Nq8xQp1jK9L",
  "name": "田中太郎",
  "email": "tanaka@example.com",
  "role": "member",
  "status": "active",
  "createdAt": "2026-01-28T12:00:00Z",
  "updatedAt": "2026-01-28T12:00:00Z"
}
```

**ポイント:**
- `201 Created`を使用
- `Location`ヘッダーで作成されたリソースのURLを示す
- リクエストにないフィールド（`id`, `role`, `status`, `createdAt`, `updatedAt`）が追加されている

#### READ（GET）- 200 OK

##### 単一リソース取得

```http
GET /api/v1/users/usr_2Nq8xQp1jK9L
```

**レスポンス（200 OK）:**

```json
{
  "id": "usr_2Nq8xQp1jK9L",
  "name": "田中太郎",
  "email": "tanaka@example.com",
  "role": "member",
  "status": "active",
  "lastLoginAt": "2026-01-28T10:30:00Z",
  "createdAt": "2026-01-28T12:00:00Z",
  "updatedAt": "2026-01-28T12:00:00Z"
}
```

##### リスト取得（ページネーション付き）

```http
GET /api/v1/users?page=1&limit=20&sort=createdAt:desc
```

**レスポンス（200 OK）:**

```json
{
  "data": [
    {
      "id": "usr_2Nq8xQp1jK9L",
      "name": "田中太郎",
      "email": "tanaka@example.com",
      "role": "member",
      "status": "active",
      "createdAt": "2026-01-28T12:00:00Z"
    },
    {
      "id": "usr_3Mp7yRo2kL8M",
      "name": "佐藤花子",
      "email": "sato@example.com",
      "role": "admin",
      "status": "active",
      "createdAt": "2026-01-27T15:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 156,
    "totalPages": 8,
    "hasNext": true,
    "hasPrev": false
  }
}
```

**ポイント:**
- リストは`data`配列に格納
- ページネーション情報を`pagination`オブジェクトで提供
- `hasNext`, `hasPrev`で次/前のページの有無を示す

#### UPDATE（PUT/PATCH）- 200 OK

##### PUT（完全更新）

```http
PUT /api/v1/users/usr_2Nq8xQp1jK9L
```

**リクエスト:**

```json
{
  "name": "田中太郎（更新後）",
  "email": "tanaka.updated@example.com",
  "role": "admin"
}
```

**レスポンス（200 OK）:**

```json
{
  "id": "usr_2Nq8xQp1jK9L",
  "name": "田中太郎（更新後）",
  "email": "tanaka.updated@example.com",
  "role": "admin",
  "status": "active",
  "createdAt": "2026-01-28T12:00:00Z",
  "updatedAt": "2026-01-28T14:30:00Z"
}
```

##### PATCH（部分更新）

```http
PATCH /api/v1/users/usr_2Nq8xQp1jK9L
```

**リクエスト:**

```json
{
  "name": "田中太郎（更新後）"
}
```

**レスポンス（200 OK）:**

```json
{
  "id": "usr_2Nq8xQp1jK9L",
  "name": "田中太郎（更新後）",
  "email": "tanaka@example.com",
  "role": "member",
  "status": "active",
  "createdAt": "2026-01-28T12:00:00Z",
  "updatedAt": "2026-01-28T14:30:00Z"
}
```

**ポイント:**
- `updatedAt`が更新されている
- PATCHでは指定されたフィールドのみ更新

#### DELETE（DELETE）- 204 No Content

```http
DELETE /api/v1/users/usr_2Nq8xQp1jK9L
```

**レスポンス（204 No Content）:**

```http
HTTP/1.1 204 No Content
```

（レスポンスボディなし）

**ポイント:**
- `204 No Content`はボディを返さない
- 削除が成功したことを示すだけ

##### 代替パターン: 200 OK with Body

一部のAPIでは、削除されたリソースの情報を返すこともあります：

```http
DELETE /api/v1/users/usr_2Nq8xQp1jK9L
```

**レスポンス（200 OK）:**

```json
{
  "id": "usr_2Nq8xQp1jK9L",
  "name": "田中太郎",
  "email": "tanaka@example.com",
  "status": "deleted",
  "deletedAt": "2026-01-28T15:00:00Z"
}
```

### レスポンスフィールドの説明

レスポンスの各フィールドについても、詳細な説明を提供します。

#### 基本的な説明

| フィールド | 型 | 説明 |
|-----------|-----|------|
| id | string | ユーザーID（`usr_`プレフィックス付き、16文字） |
| name | string | ユーザー名 |
| email | string | メールアドレス |
| role | string | ロール（`admin`, `member`, `viewer`のいずれか） |
| status | string | アカウント状態（`active`, `suspended`, `deleted`） |
| createdAt | string | 作成日時（ISO 8601形式、UTC） |
| updatedAt | string | 更新日時（ISO 8601形式、UTC） |

#### 詳細な説明（値の例を含む）

| フィールド | 型 | Nullable | 説明 | 値の例 |
|-----------|-----|----------|------|--------|
| id | string | ❌ | ユーザーID | `"usr_2Nq8xQp1jK9L"` |
| name | string | ❌ | ユーザー名（1-50文字） | `"田中太郎"` |
| email | string | ❌ | メールアドレス（RFC 5322準拠） | `"tanaka@example.com"` |
| role | string | ❌ | ロール | `"admin"`, `"member"`, `"viewer"` |
| status | string | ❌ | アカウント状態 | `"active"`, `"suspended"`, `"deleted"` |
| lastLoginAt | string | ✅ | 最終ログイン日時（未ログインの場合は`null`） | `"2026-01-28T10:30:00Z"` |
| createdAt | string | ❌ | 作成日時（ISO 8601、UTC） | `"2026-01-28T12:00:00Z"` |
| updatedAt | string | ❌ | 更新日時（ISO 8601、UTC） | `"2026-01-28T12:00:00Z"` |

### 条件付きフィールド

レスポンスに含まれるフィールドが条件によって変わる場合は、その旨を明記します。

#### 例：ユーザーの権限によって返されるフィールドが異なる

```markdown
### GET /api/v1/users/:id

#### レスポンス

**基本フィールド（すべてのユーザーに返される）:**

| フィールド | 型 | 説明 |
|-----------|-----|------|
| id | string | ユーザーID |
| name | string | ユーザー名 |
| role | string | ロール |

**管理者のみに返されるフィールド:**

| フィールド | 型 | 説明 |
|-----------|-----|------|
| email | string | メールアドレス |
| phoneNumber | string | 電話番号 |
| lastLoginAt | string | 最終ログイン日時 |

**本人または管理者のみに返されるフィールド:**

| フィールド | 型 | 説明 |
|-----------|-----|------|
| preferences | object | ユーザー設定 |
| address | object | 住所情報 |
```

---

## エラーレスポンスの文書化

### エラーレスポンスが重要な理由

開発中、**成功よりもエラーの方が頻繁に発生します**：

- バリデーションエラー
- 認証エラー
- 認可エラー
- リソースが見つからない
- サーバーエラー

エラーレスポンスが明確に文書化されていないと、開発者は：

- エラーの原因がわからない
- 適切なエラーハンドリングができない
- デバッグに時間がかかる

### エラーレスポンスの統一フォーマット

エラーレスポンスは、**API全体で統一したフォーマット**を使うべきです。

#### 推奨フォーマット

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "人間が読めるエラーメッセージ",
    "details": [
      {
        "field": "email",
        "message": "有効なメールアドレスを入力してください"
      }
    ],
    "requestId": "req_3Np8yRo2kL8M",
    "timestamp": "2026-01-28T12:00:00Z"
  }
}
```

**フィールドの説明:**

| フィールド | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| error | object | ✅ | エラー情報を格納するオブジェクト |
| error.code | string | ✅ | マシンリーダブルなエラーコード（大文字スネークケース） |
| error.message | string | ✅ | 人間が読めるエラーメッセージ |
| error.details | array | ❌ | 詳細なエラー情報（バリデーションエラーなど） |
| error.requestId | string | ❌ | リクエストID（サポートへの問い合わせ時に使用） |
| error.timestamp | string | ❌ | エラー発生日時 |

### HTTPステータスコード別のエラー例

#### 400 Bad Request - バリデーションエラー

```http
POST /api/v1/users
```

**リクエスト:**

```json
{
  "name": "",
  "email": "invalid-email"
}
```

**レスポンス（400 Bad Request）:**

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "リクエストパラメータが不正です",
    "details": [
      {
        "field": "name",
        "message": "ユーザー名は1文字以上50文字以下で入力してください",
        "value": ""
      },
      {
        "field": "email",
        "message": "有効なメールアドレスを入力してください",
        "value": "invalid-email"
      }
    ]
  }
}
```

#### 401 Unauthorized - 認証エラー

```http
GET /api/v1/users
Authorization: Bearer invalid_token
```

**レスポンス（401 Unauthorized）:**

```json
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "認証に失敗しました。有効なAPIキーを提供してください"
  }
}
```

**別パターン: トークン期限切れ**

```json
{
  "error": {
    "code": "TOKEN_EXPIRED",
    "message": "アクセストークンの有効期限が切れています",
    "details": [
      {
        "field": "token",
        "message": "リフレッシュトークンを使用して新しいアクセストークンを取得してください"
      }
    ]
  }
}
```

#### 403 Forbidden - 認可エラー

```http
DELETE /api/v1/users/usr_3Mp7yRo2kL8M
Authorization: Bearer valid_token_but_insufficient_permission
```

**レスポンス（403 Forbidden）:**

```json
{
  "error": {
    "code": "FORBIDDEN",
    "message": "このリソースにアクセスする権限がありません",
    "details": [
      {
        "resource": "users",
        "action": "delete",
        "requiredRole": "admin",
        "currentRole": "member"
      }
    ]
  }
}
```

#### 404 Not Found - リソースが見つからない

```http
GET /api/v1/users/usr_nonexistent
```

**レスポンス（404 Not Found）:**

```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "指定されたユーザーが見つかりません",
    "details": [
      {
        "resource": "users",
        "id": "usr_nonexistent"
      }
    ]
  }
}
```

#### 409 Conflict - リソースの競合

```http
POST /api/v1/users
```

**リクエスト:**

```json
{
  "name": "田中太郎",
  "email": "tanaka@example.com"
}
```

**レスポンス（409 Conflict）:**

```json
{
  "error": {
    "code": "RESOURCE_CONFLICT",
    "message": "このメールアドレスは既に使用されています",
    "details": [
      {
        "field": "email",
        "message": "tanaka@example.comは既に登録されています",
        "conflictingResource": {
          "type": "user",
          "id": "usr_2Nq8xQp1jK9L"
        }
      }
    ]
  }
}
```

#### 422 Unprocessable Entity - ビジネスロジックエラー

```http
POST /api/v1/projects/:projectId/members
```

**リクエスト:**

```json
{
  "userId": "usr_2Nq8xQp1jK9L",
  "role": "owner"
}
```

**レスポンス（422 Unprocessable Entity）:**

```json
{
  "error": {
    "code": "BUSINESS_RULE_VIOLATION",
    "message": "プロジェクトには既にオーナーが存在します",
    "details": [
      {
        "rule": "single_owner_per_project",
        "message": "1つのプロジェクトに設定できるオーナーは1人までです",
        "currentOwner": {
          "userId": "usr_3Mp7yRo2kL8M",
          "name": "佐藤花子"
        }
      }
    ]
  }
}
```

#### 429 Too Many Requests - レート制限

```http
POST /api/v1/users
```

**レスポンス（429 Too Many Requests）:**

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1706443200
Retry-After: 60
```

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "リクエスト数が制限を超えました。しばらく待ってから再試行してください",
    "details": [
      {
        "limit": 100,
        "window": "1分",
        "remaining": 0,
        "resetAt": "2026-01-28T12:00:00Z",
        "retryAfter": 60
      }
    ]
  }
}
```

#### 500 Internal Server Error - サーバーエラー

```http
GET /api/v1/users
```

**レスポンス（500 Internal Server Error）:**

```json
{
  "error": {
    "code": "INTERNAL_SERVER_ERROR",
    "message": "サーバーエラーが発生しました。問題が解決しない場合はサポートにお問い合わせください",
    "requestId": "req_3Np8yRo2kL8M"
  }
}
```

**ポイント:**
- 内部実装の詳細は公開しない（セキュリティ上の理由）
- `requestId`を提供してサポートが調査できるようにする
- 一般的なメッセージのみを返す

#### 503 Service Unavailable - メンテナンス中

```http
GET /api/v1/users
```

**レスポンス（503 Service Unavailable）:**

```http
HTTP/1.1 503 Service Unavailable
Retry-After: 3600
```

```json
{
  "error": {
    "code": "SERVICE_UNAVAILABLE",
    "message": "現在メンテナンス中です。しばらくお待ちください",
    "details": [
      {
        "maintenanceWindow": {
          "start": "2026-01-28T12:00:00Z",
          "end": "2026-01-28T13:00:00Z"
        },
        "retryAfter": 3600
      }
    ]
  }
}
```

### エラーコードの一覧表

すべてのエラーコードを一覧にしておくと、開発者がエラーハンドリングを実装しやすくなります。

| HTTPステータス | エラーコード | 説明 | 対処方法 |
|---------------|-------------|------|---------|
| 400 | `VALIDATION_ERROR` | リクエストパラメータが不正 | `details`を確認して修正 |
| 400 | `INVALID_REQUEST` | リクエスト形式が不正 | JSON形式やヘッダーを確認 |
| 401 | `UNAUTHORIZED` | 認証情報が無効 | 有効なAPIキーを提供 |
| 401 | `TOKEN_EXPIRED` | トークンの有効期限切れ | リフレッシュトークンで更新 |
| 403 | `FORBIDDEN` | アクセス権限がない | 必要な権限を持つユーザーで実行 |
| 404 | `RESOURCE_NOT_FOUND` | リソースが見つからない | IDを確認するか、リソースを作成 |
| 409 | `RESOURCE_CONFLICT` | リソースが既に存在 | 既存リソースを更新するか、別の値を使用 |
| 422 | `BUSINESS_RULE_VIOLATION` | ビジネスルール違反 | `details`を確認してルールに従う |
| 429 | `RATE_LIMIT_EXCEEDED` | レート制限超過 | `retryAfter`秒後に再試行 |
| 500 | `INTERNAL_SERVER_ERROR` | サーバーエラー | `requestId`を控えてサポートに連絡 |
| 503 | `SERVICE_UNAVAILABLE` | サービス利用不可 | メンテナンス終了後に再試行 |

---

## バリデーションルールの明記

### なぜバリデーションルールが重要なのか

バリデーションルールが明記されていないと：

- 開発者は試行錯誤でルールを探る必要がある
- エラーが出てから初めてルールがわかる
- クライアント側でバリデーションを実装できない

### 基本的なバリデーションルール

#### 文字列の制約

| フィールド | 型 | バリデーション | 説明 |
|-----------|-----|---------------|------|
| name | string | 1-50文字、必須 | 空白のみは不可 |
| email | string | RFC 5322準拠、必須 | 有効なメールアドレス形式 |
| phoneNumber | string | E.164形式、任意 | 例: `+81-90-1234-5678` |
| zipCode | string | 正規表現: `^\d{3}-\d{4}$`、任意 | 例: `100-0001` |
| url | string | RFC 3986準拠、任意 | `http://`または`https://`で始まる |

#### 数値の制約

| フィールド | 型 | バリデーション | 説明 |
|-----------|-----|---------------|------|
| age | number | 0-150の整数、必須 | 年齢 |
| price | number | 0以上の数値、必須 | 小数点以下2桁まで |
| quantity | number | 1-9999の整数、必須 | 在庫数 |
| percentage | number | 0.0-1.0の小数、必須 | 割合（0.5 = 50%） |

#### 配列の制約

| フィールド | 型 | バリデーション | 説明 |
|-----------|-----|---------------|------|
| tags | string[] | 最大10要素、各要素1-20文字、任意 | タグリスト |
| members | object[] | 1-50要素、必須 | メンバーリスト（最低1人） |
| attachments | string[] | 最大5要素、各要素はURL形式、任意 | 添付ファイルURL |

#### 列挙型の制約

| フィールド | 型 | 許可される値 | デフォルト |
|-----------|-----|-------------|----------|
| role | string | `admin`, `member`, `viewer` | `member` |
| status | string | `draft`, `published`, `archived` | `draft` |
| priority | string | `low`, `medium`, `high`, `urgent` | `medium` |

### 複雑なバリデーションルール

#### 相互依存するフィールド

```markdown
#### バリデーションルール

**基本ルール:**
- `startDate`と`endDate`はISO 8601形式の日付文字列（`YYYY-MM-DD`）
- `startDate`は必須、`endDate`は任意

**相互依存ルール:**
- `endDate`を指定する場合、`startDate`より後の日付である必要があります
- `endDate`を省略した場合、期限なしとして扱われます

**例:**

✅ 有効な例:
```json
{
  "startDate": "2026-01-28",
  "endDate": "2026-02-28"
}
```

```json
{
  "startDate": "2026-01-28"
  // endDateなし = 期限なし
}
```

❌ 無効な例:
```json
{
  "startDate": "2026-02-28",
  "endDate": "2026-01-28"  // startDateより前
}
```
```

#### 条件付きバリデーション

```markdown
#### バリデーションルール

**`paymentMethod`による条件分岐:**

**`paymentMethod`が`credit_card`の場合:**
- `cardNumber`: 必須（14-16桁の数字）
- `cardExpiry`: 必須（`MM/YY`形式）
- `cardCvc`: 必須（3-4桁の数字）

**`paymentMethod`が`bank_transfer`の場合:**
- `bankName`: 必須
- `accountNumber`: 必須（7-12桁の数字）
- `accountHolder`: 必須

**例:**

✅ クレジットカード決済:
```json
{
  "paymentMethod": "credit_card",
  "cardNumber": "4111111111111111",
  "cardExpiry": "12/28",
  "cardCvc": "123"
}
```

✅ 銀行振込:
```json
{
  "paymentMethod": "bank_transfer",
  "bankName": "三菱UFJ銀行",
  "accountNumber": "1234567",
  "accountHolder": "田中太郎"
}
```

❌ 無効な例（必須フィールドが不足）:
```json
{
  "paymentMethod": "credit_card"
  // cardNumber, cardExpiry, cardCvcが不足
}
```
```

### 正規表現によるバリデーション

正規表現を使用する場合は、説明と例を併記します。

| フィールド | 正規表現 | 説明 | 有効な例 | 無効な例 |
|-----------|---------|------|---------|---------|
| zipCode | `^\d{3}-\d{4}$` | 郵便番号（ハイフン区切り） | `100-0001` | `1000001`, `100-00001` |
| phoneNumber | `^\+81-\d{1,4}-\d{4}-\d{4}$` | 日本の電話番号（国際形式） | `+81-90-1234-5678` | `090-1234-5678`, `+81901234567` |
| slug | `^[a-z0-9]+(?:-[a-z0-9]+)*$` | URL用スラッグ（小文字、数字、ハイフン） | `my-project-name` | `My Project`, `my_project` |
| hexColor | `^#[0-9A-Fa-f]{6}$` | 16進数カラーコード | `#FF5733`, `#ff5733` | `FF5733`, `#F57` |

---

## データ型とフォーマットの説明

### 基本的なデータ型

| 型 | JSON型 | 説明 | 例 |
|----|--------|------|-----|
| string | string | 文字列 | `"田中太郎"` |
| number | number | 数値（整数または小数） | `123`, `123.45` |
| boolean | boolean | 真偽値 | `true`, `false` |
| null | null | 値が存在しない | `null` |
| object | object | オブジェクト | `{"key": "value"}` |
| array | array | 配列 | `[1, 2, 3]` |

### 日時フォーマット

#### ISO 8601形式（推奨）

```markdown
#### 日時フィールドのフォーマット

すべての日時フィールドは**ISO 8601形式**で表現されます。

**日時（タイムゾーン付き）:**
- フォーマット: `YYYY-MM-DDTHH:mm:ssZ`
- タイムゾーン: UTC
- 例: `2026-01-28T12:00:00Z`

**日付のみ:**
- フォーマット: `YYYY-MM-DD`
- 例: `2026-01-28`

**時刻のみ:**
- フォーマット: `HH:mm:ss`
- 例: `12:00:00`

**タイムゾーンを含む日時（RFC 3339）:**
- フォーマット: `YYYY-MM-DDTHH:mm:ss+09:00`
- 例: `2026-01-28T12:00:00+09:00`（日本時間）

**よくある間違い:**

❌ `2026/01/28` - スラッシュ区切り（ISO 8601ではない）
❌ `2026-1-28` - 月・日がゼロパディングされていない
❌ `2026-01-28 12:00:00` - `T`区切りがない
```

#### タイムスタンプ形式

```markdown
#### Unixタイムスタンプ

一部のフィールドでは、Unixタイムスタンプ（秒単位）を使用します。

**フォーマット:**
- 型: `number`
- 説明: 1970年1月1日00:00:00 UTCからの経過秒数
- 例: `1706443200`（2026-01-28T12:00:00Z）

**ミリ秒タイムスタンプ:**
- 型: `number`
- 説明: 1970年1月1日00:00:00 UTCからの経過ミリ秒数
- 例: `1706443200000`

**JavaScriptでの変換:**

```typescript
// ISO 8601 → Unixタイムスタンプ（秒）
const timestamp = Math.floor(new Date('2026-01-28T12:00:00Z').getTime() / 1000);

// Unixタイムスタンプ（秒） → ISO 8601
const isoString = new Date(timestamp * 1000).toISOString();
```
```

### 通貨フォーマット

```markdown
#### 通貨フィールド

**基本フォーマット:**
- 型: `number`
- 単位: 各通貨の最小単位（例: 日本円は「円」、米ドルは「セント」）
- 例:
  - `1000` = 1,000円
  - `299` = 2.99ドル（299セント）

**理由:**
小数点を避けることで、浮動小数点演算の誤差を防ぎます。

**オブジェクト形式（推奨）:**

```json
{
  "amount": 1000,
  "currency": "JPY"
}
```

| フィールド | 型 | 説明 |
|-----------|-----|------|
| amount | number | 金額（最小単位） |
| currency | string | 通貨コード（ISO 4217） |

**通貨コード例:**
- `JPY`: 日本円
- `USD`: 米ドル
- `EUR`: ユーロ
- `GBP`: 英ポンド

**JavaScriptでの表示:**

```typescript
const price = {
  amount: 1000,
  currency: 'JPY'
};

// 日本円の場合、そのまま表示
console.log(`${price.amount}円`); // 1000円

// ドルの場合、100で割る
const priceUSD = {
  amount: 299,
  currency: 'USD'
};
console.log(`$${priceUSD.amount / 100}`); // $2.99
```
```

### ID形式

```markdown
#### IDフィールドのフォーマット

すべてのリソースIDは、**プレフィックス + アンダースコア + ランダム文字列**の形式です。

| リソース | プレフィックス | 例 |
|---------|--------------|-----|
| User | `usr_` | `usr_2Nq8xQp1jK9L` |
| Project | `prj_` | `prj_3Mp7yRo2kL8M` |
| Task | `tsk_` | `tsk_4No8zSp3mM9N` |
| Comment | `cmt_` | `cmt_5Op9aTq4nN0O` |

**長さ:** プレフィックス（4文字） + ランダム文字列（12文字） = 合計16文字

**ランダム文字列:** Base58エンコード（数字と大小英字、紛らわしい文字を除外）

**利点:**
- リソースタイプがIDから判別できる
- URLやログで見たときに識別しやすい
- 大小英字・数字のみなので扱いやすい
```

---

## 実際のユースケースに基づいた例

### ユースケース1: ユーザー登録フロー

#### 1. ユーザー登録

```http
POST /api/v1/auth/register
```

**リクエスト:**

```json
{
  "email": "tanaka@example.com",
  "password": "SecurePassword123!",
  "name": "田中太郎"
}
```

**レスポンス（201 Created）:**

```json
{
  "user": {
    "id": "usr_2Nq8xQp1jK9L",
    "email": "tanaka@example.com",
    "name": "田中太郎",
    "status": "pending_verification",
    "createdAt": "2026-01-28T12:00:00Z"
  },
  "message": "確認メールを送信しました。メール内のリンクをクリックしてアカウントを有効化してください"
}
```

#### 2. メールアドレス確認

```http
POST /api/v1/auth/verify-email
```

**リクエスト:**

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**レスポンス（200 OK）:**

```json
{
  "user": {
    "id": "usr_2Nq8xQp1jK9L",
    "email": "tanaka@example.com",
    "name": "田中太郎",
    "status": "active",
    "verifiedAt": "2026-01-28T12:05:00Z"
  },
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresIn": 3600
}
```

#### 3. ログイン

```http
POST /api/v1/auth/login
```

**リクエスト:**

```json
{
  "email": "tanaka@example.com",
  "password": "SecurePassword123!"
}
```

**レスポンス（200 OK）:**

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresIn": 3600,
  "user": {
    "id": "usr_2Nq8xQp1jK9L",
    "email": "tanaka@example.com",
    "name": "田中太郎",
    "role": "member"
  }
}
```

### ユースケース2: プロジェクト作成と管理

#### 1. プロジェクト作成

```http
POST /api/v1/projects
```

**リクエスト:**

```json
{
  "name": "Webサイトリニューアル",
  "description": "コーポレートサイトの全面リニューアルプロジェクト",
  "startDate": "2026-02-01",
  "endDate": "2026-05-31",
  "members": [
    {
      "userId": "usr_2Nq8xQp1jK9L",
      "role": "owner"
    },
    {
      "userId": "usr_3Mp7yRo2kL8M",
      "role": "editor"
    }
  ],
  "tags": ["web", "design", "urgent"]
}
```

**レスポンス（201 Created）:**

```json
{
  "id": "prj_4No8zSp3mM9N",
  "name": "Webサイトリニューアル",
  "description": "コーポレートサイトの全面リニューアルプロジェクト",
  "status": "active",
  "startDate": "2026-02-01",
  "endDate": "2026-05-31",
  "progress": 0,
  "members": [
    {
      "userId": "usr_2Nq8xQp1jK9L",
      "name": "田中太郎",
      "email": "tanaka@example.com",
      "role": "owner"
    },
    {
      "userId": "usr_3Mp7yRo2kL8M",
      "name": "佐藤花子",
      "email": "sato@example.com",
      "role": "editor"
    }
  ],
  "tags": ["web", "design", "urgent"],
  "createdAt": "2026-01-28T12:00:00Z",
  "updatedAt": "2026-01-28T12:00:00Z"
}
```

#### 2. タスク作成

```http
POST /api/v1/projects/prj_4No8zSp3mM9N/tasks
```

**リクエスト:**

```json
{
  "title": "デザインモックアップ作成",
  "description": "トップページとサブページのデザインモックアップを作成",
  "assigneeId": "usr_3Mp7yRo2kL8M",
  "priority": "high",
  "dueDate": "2026-02-15",
  "estimatedHours": 40,
  "tags": ["design", "mockup"]
}
```

**レスポンス（201 Created）:**

```json
{
  "id": "tsk_5Op9aTq4nN0O",
  "projectId": "prj_4No8zSp3mM9N",
  "title": "デザインモックアップ作成",
  "description": "トップページとサブページのデザインモックアップを作成",
  "status": "todo",
  "priority": "high",
  "assignee": {
    "userId": "usr_3Mp7yRo2kL8M",
    "name": "佐藤花子"
  },
  "dueDate": "2026-02-15",
  "estimatedHours": 40,
  "actualHours": 0,
  "progress": 0,
  "tags": ["design", "mockup"],
  "createdBy": "usr_2Nq8xQp1jK9L",
  "createdAt": "2026-01-28T12:10:00Z",
  "updatedAt": "2026-01-28T12:10:00Z"
}
```

#### 3. タスクステータス更新

```http
PATCH /api/v1/tasks/tsk_5Op9aTq4nN0O
```

**リクエスト:**

```json
{
  "status": "in_progress"
}
```

**レスポンス（200 OK）:**

```json
{
  "id": "tsk_5Op9aTq4nN0O",
  "projectId": "prj_4No8zSp3mM9N",
  "title": "デザインモックアップ作成",
  "status": "in_progress",
  "startedAt": "2026-01-28T13:00:00Z",
  "updatedAt": "2026-01-28T13:00:00Z"
}
```

#### 4. コメント追加

```http
POST /api/v1/tasks/tsk_5Op9aTq4nN0O/comments
```

**リクエスト:**

```json
{
  "body": "トップページのモックアップが完成しました。レビューお願いします。",
  "attachments": [
    "https://example.com/files/mockup-top.png"
  ]
}
```

**レスポンス（201 Created）:**

```json
{
  "id": "cmt_6Pq0bUr5oO1P",
  "taskId": "tsk_5Op9aTq4nN0O",
  "author": {
    "userId": "usr_3Mp7yRo2kL8M",
    "name": "佐藤花子"
  },
  "body": "トップページのモックアップが完成しました。レビューお願いします。",
  "attachments": [
    {
      "url": "https://example.com/files/mockup-top.png",
      "filename": "mockup-top.png",
      "size": 1024000,
      "mimeType": "image/png"
    }
  ],
  "createdAt": "2026-01-28T15:00:00Z",
  "updatedAt": "2026-01-28T15:00:00Z"
}
```

### ユースケース3: 検索とフィルタリング

#### 複数条件での検索

```http
GET /api/v1/tasks?projectId=prj_4No8zSp3mM9N&status=in_progress&assigneeId=usr_3Mp7yRo2kL8M&priority=high&sort=dueDate:asc&page=1&limit=20
```

**クエリパラメータ:**

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| projectId | string | ❌ | プロジェクトIDでフィルタ |
| status | string | ❌ | ステータスでフィルタ（`todo`, `in_progress`, `done`, `archived`） |
| assigneeId | string | ❌ | 担当者IDでフィルタ |
| priority | string | ❌ | 優先度でフィルタ（`low`, `medium`, `high`, `urgent`） |
| dueDate | string | ❌ | 期限日でフィルタ（ISO 8601形式） |
| tags | string | ❌ | タグでフィルタ（カンマ区切りで複数指定可能） |
| search | string | ❌ | タイトル・説明文での全文検索 |
| sort | string | ❌ | ソート順（`field:order`形式、例: `dueDate:asc`） |
| page | number | ❌ | ページ番号（デフォルト: 1） |
| limit | number | ❌ | 1ページあたりの件数（デフォルト: 20、最大: 100） |

**レスポンス（200 OK）:**

```json
{
  "data": [
    {
      "id": "tsk_5Op9aTq4nN0O",
      "projectId": "prj_4No8zSp3mM9N",
      "title": "デザインモックアップ作成",
      "status": "in_progress",
      "priority": "high",
      "assignee": {
        "userId": "usr_3Mp7yRo2kL8M",
        "name": "佐藤花子"
      },
      "dueDate": "2026-02-15",
      "progress": 30,
      "createdAt": "2026-01-28T12:10:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 1,
    "totalPages": 1,
    "hasNext": false,
    "hasPrev": false
  },
  "filters": {
    "projectId": "prj_4No8zSp3mM9N",
    "status": "in_progress",
    "assigneeId": "usr_3Mp7yRo2kL8M",
    "priority": "high"
  },
  "sort": {
    "field": "dueDate",
    "order": "asc"
  }
}
```

---

## よくある間違いと改善方法

### 間違い1: 説明だけで例がない

#### ❌ 悪い例

```markdown
### POST /api/users

ユーザーを作成します。nameとemailが必須です。
```

#### ✅ 良い例

```markdown
### POST /api/users

ユーザーを作成します。

#### リクエスト例

```json
{
  "name": "田中太郎",
  "email": "tanaka@example.com"
}
```

#### レスポンス例（201 Created）

```json
{
  "id": "usr_2Nq8xQp1jK9L",
  "name": "田中太郎",
  "email": "tanaka@example.com",
  "createdAt": "2026-01-28T12:00:00Z"
}
```
```

### 間違い2: エラーケースがない

#### ❌ 悪い例

成功時のレスポンスしか記載していない。

#### ✅ 良い例

```markdown
#### レスポンス

**成功時（201 Created）:**
```json
{
  "id": "usr_2Nq8xQp1jK9L",
  "name": "田中太郎"
}
```

**エラー時（400 Bad Request）:**
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "emailは必須です"
  }
}
```
```

### 間違い3: データ型が不明

#### ❌ 悪い例

```markdown
| フィールド | 説明 |
|-----------|------|
| createdAt | 作成日時 |
```

#### ✅ 良い例

```markdown
| フィールド | 型 | 説明 |
|-----------|-----|------|
| createdAt | string | 作成日時（ISO 8601形式、UTC） 例: `2026-01-28T12:00:00Z` |
```

### 間違い4: バリデーションルールが不明

#### ❌ 悪い例

```markdown
| フィールド | 型 | 必須 |
|-----------|-----|------|
| name | string | ✅ |
```

#### ✅ 良い例

```markdown
| フィールド | 型 | 必須 | バリデーション |
|-----------|-----|------|---------------|
| name | string | ✅ | 1-50文字、空白のみは不可 |
```

### 間違い5: 認証情報の記載がない

#### ❌ 悪い例

```bash
curl -X GET https://api.example.com/v1/users
```

#### ✅ 良い例

```bash
curl -X GET https://api.example.com/v1/users \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### 間違い6: 実際のAPIキーを公開

#### ❌ 悪い例

```bash
curl -X GET https://api.example.com/v1/users \
  -H "Authorization: Bearer sk_live_1234567890abcdef"
```

#### ✅ 良い例

```bash
curl -X GET https://api.example.com/v1/users \
  -H "Authorization: Bearer YOUR_API_KEY"
```

> **注意:** 実際のAPIキー、トークン、パスワードは絶対に公開しないでください。

---

## ツールを活用した例の自動生成

### Postmanでの例の管理

Postmanを使うと、リクエスト・レスポンス例を簡単に管理・共有できます。

#### Postmanのコレクション例

```json
{
  "info": {
    "name": "User API",
    "description": "ユーザー管理API",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "item": [
    {
      "name": "Create User",
      "request": {
        "method": "POST",
        "header": [
          {
            "key": "Content-Type",
            "value": "application/json"
          },
          {
            "key": "Authorization",
            "value": "Bearer {{apiKey}}"
          }
        ],
        "body": {
          "mode": "raw",
          "raw": "{\n  \"name\": \"田中太郎\",\n  \"email\": \"tanaka@example.com\"\n}"
        },
        "url": {
          "raw": "{{baseUrl}}/api/v1/users",
          "host": ["{{baseUrl}}"],
          "path": ["api", "v1", "users"]
        }
      },
      "response": [
        {
          "name": "Success",
          "originalRequest": {
            "method": "POST",
            "header": [],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"name\": \"田中太郎\",\n  \"email\": \"tanaka@example.com\"\n}"
            },
            "url": {
              "raw": "{{baseUrl}}/api/v1/users",
              "host": ["{{baseUrl}}"],
              "path": ["api", "v1", "users"]
            }
          },
          "status": "Created",
          "code": 201,
          "_postman_previewlanguage": "json",
          "header": [
            {
              "key": "Content-Type",
              "value": "application/json"
            }
          ],
          "body": "{\n  \"id\": \"usr_2Nq8xQp1jK9L\",\n  \"name\": \"田中太郎\",\n  \"email\": \"tanaka@example.com\",\n  \"createdAt\": \"2026-01-28T12:00:00Z\"\n}"
        }
      ]
    }
  ]
}
```

### HTTPieでの例

HTTPieは、curlよりも読みやすいCLI HTTPクライアントです。

```bash
# GET リクエスト
http GET https://api.example.com/v1/users \
  Authorization:"Bearer YOUR_API_KEY"

# POST リクエスト
http POST https://api.example.com/v1/users \
  Authorization:"Bearer YOUR_API_KEY" \
  name="田中太郎" \
  email="tanaka@example.com"

# JSON ファイルから読み込み
http POST https://api.example.com/v1/users \
  Authorization:"Bearer YOUR_API_KEY" \
  < user.json
```

### TypeScriptの型定義から例を生成

TypeScriptの型定義があれば、それをドキュメントに活用できます。

```typescript
// types/user.ts
export interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'member' | 'viewer';
  status: 'active' | 'suspended' | 'deleted';
  createdAt: string;
  updatedAt: string;
}

export interface CreateUserRequest {
  name: string;
  email: string;
  role?: 'admin' | 'member' | 'viewer';
}

export interface CreateUserResponse {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'member' | 'viewer';
  status: 'active';
  createdAt: string;
  updatedAt: string;
}
```

この型定義をドキュメントに記載することで、開発者は型安全にAPIを使用できます。

---

## チェックリスト

この章で学んだことを確認しましょう。

### リクエスト例

- [ ] curlコマンドがコピー&ペーストで動作する
- [ ] 必要なヘッダー（認証、Content-Typeなど）が含まれている
- [ ] JSONが整形されていて読みやすい
- [ ] 実際のAPIキーやトークンではなく、プレースホルダーを使用している
- [ ] 複数の言語での例を提供している（任意）

### レスポンス例

- [ ] HTTPステータスコードが明記されている
- [ ] 成功時のレスポンスがある
- [ ] エラー時のレスポンスがある
- [ ] JSONが整形されていて読みやすい
- [ ] 各フィールドの説明がある
- [ ] データ型が明記されている

### パラメータ説明

- [ ] すべてのパラメータが表形式で説明されている
- [ ] 型、必須/任意が明記されている
- [ ] バリデーションルールが明記されている
- [ ] デフォルト値がある場合は記載されている
- [ ] 値の例が提供されている

### エラーレスポンス

- [ ] よくあるエラーケースがすべて文書化されている
- [ ] エラーコードとメッセージが明確
- [ ] エラーの原因と対処方法が説明されている
- [ ] エラーレスポンスのフォーマットが統一されている

### バリデーションルール

- [ ] すべてのバリデーションルールが明記されている
- [ ] 文字列の長さ制限がある場合は記載されている
- [ ] 正規表現を使う場合は説明と例がある
- [ ] 相互依存するフィールドのルールが説明されている
- [ ] 条件付きバリデーションが説明されている

### データ型とフォーマット

- [ ] すべてのフィールドのデータ型が明記されている
- [ ] 日時フォーマット（ISO 8601など）が説明されている
- [ ] 通貨フォーマットが説明されている
- [ ] ID形式が説明されている
- [ ] Nullableフィールドが明記されている

---

## 次のステップ

この章では、リクエスト・レスポンス例の書き方を学びました。

次の章では、**OpenAPI/Swagger**を使った、より構造化されたAPI仕様書の作成方法を学びます。

### 関連リソース

- [HTTP ステータスコード - MDN](https://developer.mozilla.org/ja/docs/Web/HTTP/Status)
- [ISO 8601（日時フォーマット）- Wikipedia](https://ja.wikipedia.org/wiki/ISO_8601)
- [RFC 5322（メールアドレス形式）](https://datatracker.ietf.org/doc/html/rfc5322)
- [E.164（国際電話番号形式）](https://en.wikipedia.org/wiki/E.164)
- [ISO 4217（通貨コード）](https://en.wikipedia.org/wiki/ISO_4217)
- [Postman公式ドキュメント](https://learning.postman.com/docs/)
- [HTTPie公式ドキュメント](https://httpie.io/docs)

### 実践課題

1. 自分のプロジェクトのAPIエンドポイントを1つ選び、この章で学んだ形式でリクエスト・レスポンス例を作成してください
2. 少なくとも3つのエラーケースを文書化してください
3. すべてのバリデーションルールを明記してください
4. curlコマンドを実際に実行して、動作することを確認してください

---

**記事の文字数: 約27,000字**
