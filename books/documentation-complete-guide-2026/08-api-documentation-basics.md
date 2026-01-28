# 第8章: API仕様書の基本

## この章で学ぶこと

API仕様書は、開発者がAPIを理解し、正しく利用するために不可欠なドキュメントです。この章では、効果的なAPI仕様書を作成するための基本原則と構成要素を学びます。

**この章で学ぶ内容:**
- API仕様書の目的と読者
- API仕様書の基本構成
- API概要の書き方
- 認証・認可の文書化
- エンドポイント一覧の作成
- バージョニング戦略
- API仕様書のフォーマットと配置

API仕様書は単なる技術資料ではありません。フロントエンド開発者、モバイル開発者、サードパーティ開発者など、様々な背景を持つ読者がAPIを効率的に利用できるようにするための重要なコミュニケーションツールです。

この章では、初めてAPI仕様書を書く人でも、読者に優しく、保守可能な仕様書を作成できるように、実践的なガイドラインと豊富な例を提供します。

## API仕様書の目的と重要性

### API仕様書とは何か

API仕様書は、APIの機能、使用方法、制約事項を包括的に記述したドキュメントです。コードを読まなくても、開発者がAPIを理解し、統合できるようにすることが主な目的です。

**API仕様書が果たす役割:**

1. **理解の促進**: APIの全体像と個々のエンドポイントの機能を明確にする
2. **実装の支援**: リクエスト・レスポンスの形式、パラメータ、エラーハンドリングを示す
3. **契約の明確化**: APIプロバイダーとコンシューマー間の仕様を定義する
4. **変更の追跡**: バージョン管理と変更履歴を記録する
5. **開発の効率化**: フロントエンドとバックエンドの並行開発を可能にする

### READMEとAPI仕様書の違い

READMEとAPI仕様書は補完的な関係にありますが、目的が異なります:

**README:**
- プロジェクト全体の概要
- セットアップ手順
- 基本的な使い方
- クイックスタートガイド

**API仕様書:**
- APIの詳細な技術仕様
- すべてのエンドポイントの完全なリファレンス
- 認証フロー
- データモデル

一般的に、READMEはプロジェクトの入口として機能し、API仕様書へのリンクを提供します。

### API仕様書の読者

API仕様書の読者を理解することは、効果的なドキュメントを作成するための第一歩です。

**主な読者:**

1. **フロントエンド開発者**
   - WebアプリケーションからAPIを呼び出す
   - レスポンスデータの構造を理解する必要がある
   - エラーハンドリングの実装

2. **モバイル開発者**
   - iOS/AndroidアプリからAPIを利用
   - ネットワーク効率とキャッシュ戦略
   - オフライン対応

3. **サードパーティ開発者**
   - 外部サービスとの統合
   - 認証フローの実装
   - レート制限への対応

4. **QA/テストエンジニア**
   - APIテストケースの作成
   - 期待される動作の確認
   - エラーケースの検証

5. **技術営業・プロダクトマネージャー**
   - APIの能力と制限の理解
   - 顧客への説明
   - 新機能の企画

読者によって必要な情報が異なるため、API仕様書は複数の視点を考慮して構成する必要があります。

## API仕様書の基本構成

効果的なAPI仕様書には、いくつかの標準的なセクションがあります。すべてのセクションが必須というわけではありませんが、読者がAPIを理解し利用するために必要な情報を網羅することが重要です。

### 推奨される基本構成

```markdown
# API Documentation

## 1. 概要
- APIの目的
- 主要な機能
- 対象読者

## 2. はじめに
- 前提知識
- アカウント作成
- APIキーの取得

## 3. 認証
- 認証方式
- 認証フロー
- トークンの管理

## 4. 基本情報
- ベースURL
- リクエスト形式
- レスポンス形式
- 日付・時刻の形式

## 5. エンドポイント一覧
- リソース別のエンドポイント
- HTTPメソッド
- 簡単な説明

## 6. エンドポイント詳細
- パラメータ
- リクエスト例
- レスポンス例
- エラーレスポンス

## 7. データモデル
- リソースの構造
- フィールドの説明
- データ型

## 8. エラーハンドリング
- エラーコード一覧
- エラーレスポンスの形式
- トラブルシューティング

## 9. レート制限
- 制限の内容
- ヘッダー情報
- 制限超過時の対応

## 10. ベストプラクティス
- 効率的な利用方法
- キャッシュ戦略
- セキュリティ推奨事項

## 11. バージョニング
- 現在のバージョン
- 変更履歴
- 非推奨機能

## 12. サポート
- 問い合わせ先
- コミュニティリソース
- FAQ
```

### セクションの優先度

すべてのAPIが上記のすべてのセクションを必要とするわけではありません。プロジェクトの規模と複雑さに応じて、以下のように優先度を考慮します:

**必須（Tier 1）:**
- 概要
- 認証
- 基本情報
- エンドポイント一覧
- エンドポイント詳細

**推奨（Tier 2）:**
- はじめに
- データモデル
- エラーハンドリング
- バージョニング

**任意（Tier 3）:**
- レート制限（公開APIでは必須）
- ベストプラクティス
- サポート

### 構成の実例

小規模な内部APIと大規模な公開APIでは、構成が大きく異なります。

**❌ 避けるべきパターン:**

```markdown
# API Documentation

## GET /api/users
Returns users.

## POST /api/users
Creates a user.

## GET /api/posts
Returns posts.
```

この例は情報が不足しており、読者がAPIを理解するには不十分です。認証方法、リクエスト形式、レスポンス例、エラーハンドリングなどが欠けています。

**✅ 推奨されるパターン:**

```markdown
# User Management API Documentation

## 概要

User Management APIは、ユーザーアカウントの作成、更新、削除を行うためのREST APIです。

**主要機能:**
- ユーザーアカウント管理（CRUD操作）
- ユーザー検索とフィルタリング
- ユーザー認証とトークン管理
- ロールベースのアクセス制御

**対象読者:**
このAPIは、当社のWebアプリケーションとモバイルアプリケーションで使用されます。

## 基本情報

**ベースURL:**
- 本番環境: `https://api.example.com/v1`
- ステージング環境: `https://api-staging.example.com/v1`

**リクエスト形式:**
- Content-Type: `application/json`
- 文字エンコーディング: UTF-8

**レスポンス形式:**
- Content-Type: `application/json`

**日付時刻形式:**
- ISO 8601形式（例: `2026-01-28T12:00:00Z`）
- タイムゾーン: UTC

## 認証

このAPIはJWT（JSON Web Token）ベースの認証を使用します。

### 認証フロー

1. `/auth/login`エンドポイントで認証
2. アクセストークンとリフレッシュトークンを取得
3. 各リクエストのAuthorizationヘッダーにアクセストークンを含める

### トークンの使用

```http
GET /api/users HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### トークンの有効期限

- アクセストークン: 15分
- リフレッシュトークン: 7日

## エンドポイント一覧

### ユーザー管理

| メソッド | エンドポイント | 説明 | 認証 |
|---------|---------------|------|-----|
| GET | `/users` | ユーザー一覧取得 | 必要 |
| GET | `/users/:id` | ユーザー詳細取得 | 必要 |
| POST | `/users` | ユーザー作成 | 必要 |
| PUT | `/users/:id` | ユーザー更新 | 必要 |
| DELETE | `/users/:id` | ユーザー削除 | 必要 |

### 認証

| メソッド | エンドポイント | 説明 | 認証 |
|---------|---------------|------|-----|
| POST | `/auth/login` | ログイン | 不要 |
| POST | `/auth/refresh` | トークン更新 | リフレッシュトークン |
| POST | `/auth/logout` | ログアウト | 必要 |
```

この構成により、読者は段階的にAPIを理解できます。

## API概要の書き方

API概要は、API仕様書の最初のセクションであり、読者の第一印象を決定します。このセクションでは、APIの目的、主要機能、使用対象を明確に伝えます。

### 効果的な概要の要素

**1. APIの目的（What）**

APIが何を提供するのかを1〜2文で簡潔に説明します。

```markdown
## 概要

Payment Processing APIは、クレジットカード決済、銀行振込、電子マネー決済を安全に処理するためのREST APIです。
```

**2. 主要機能（Features）**

APIの主要な機能を箇条書きで列挙します。すべての機能を列挙するのではなく、最も重要な3〜7個に絞ります。

```markdown
**主要機能:**
- クレジットカード決済（Visa、Mastercard、JCB、American Express）
- 3Dセキュア認証対応
- 定期課金（サブスクリプション）
- 返金処理
- 取引履歴の照会
- Webhook通知
```

**3. 使用例（Use Cases）**

どのような状況でこのAPIを使用するのかを示します。

```markdown
**使用例:**
- ECサイトでの商品購入
- SaaSサービスの月額課金
- モバイルアプリでのアプリ内課金
- クラウドファンディングプラットフォーム
```

**4. 前提条件（Prerequisites）**

APIを使用する前に必要な準備や知識を明記します。

```markdown
**前提条件:**
- 加盟店アカウントの作成（[サインアップページ](https://example.com/signup)）
- APIキーの取得（管理画面から発行）
- RESTful APIの基本的な理解
- HTTPS通信の知識
```

### 概要セクションの例

**❌ 不十分な概要:**

```markdown
## 概要

このAPIはユーザー管理用です。
```

この概要は情報が不足しており、読者にAPIの価値や使用方法を伝えられません。

**✅ 効果的な概要:**

```markdown
## 概要

User Management APIは、マルチテナントSaaSアプリケーション向けのユーザーアカウント管理APIです。組織・チーム・ユーザーの階層構造をサポートし、きめ細かなアクセス制御を提供します。

### 主要機能

**ユーザー管理:**
- ユーザーアカウントのCRUD操作
- メールアドレス・電話番号の検証
- プロフィール画像のアップロード
- ユーザー検索とフィルタリング（ページネーション対応）

**組織・チーム管理:**
- 組織の作成と管理
- チームの作成とメンバー管理
- 組織階層の管理

**認証・認可:**
- JWT（JSON Web Token）ベースの認証
- OAuth 2.0対応
- ロールベースアクセス制御（RBAC）
- 多要素認証（MFA）サポート

**監査・ログ:**
- ユーザーアクティビティの追跡
- 監査ログのエクスポート
- Webhook通知

### 使用対象

このAPIは以下のクライアントでの使用を想定しています:
- Webアプリケーション（React、Vue.js、Angularなど）
- モバイルアプリケーション（iOS、Android）
- 管理ツール・内部システム

### バージョン情報

- 現在のバージョン: v1.2.0
- APIバージョン: v1
- 最終更新日: 2026-01-28

### サポート

- ドキュメント: https://docs.example.com
- API Status: https://status.example.com
- サポート: support@example.com
- コミュニティ: https://community.example.com
```

この概要は、APIの全体像を明確に伝え、読者が次に何を読むべきかを示しています。

## 基本情報の文書化

基本情報セクションでは、すべてのAPIリクエストに共通する情報を提供します。このセクションを最初に読むことで、読者はAPIの基本的な使い方を理解できます。

### ベースURL

ベースURLは、すべてのAPIエンドポイントの起点となるURLです。環境別に異なるURLを提供することが一般的です。

```markdown
## 基本情報

### ベースURL

本APIは以下のベースURLを使用します:

**本番環境:**
```
https://api.example.com/v1
```

**ステージング環境:**
```
https://api-staging.example.com/v1
```

**開発環境:**
```
https://api-dev.example.com/v1
```

すべてのエンドポイントはベースURLからの相対パスで指定します。例えば、ユーザー一覧を取得する場合:

```
GET https://api.example.com/v1/users
```
```

**環境変数の例も提供:**

```markdown
### 環境別の設定

開発時は環境変数でベースURLを管理することを推奨します:

```bash
# .env.production
VITE_API_BASE_URL=https://api.example.com/v1

# .env.staging
VITE_API_BASE_URL=https://api-staging.example.com/v1

# .env.development
VITE_API_BASE_URL=https://api-dev.example.com/v1
```
```

### リクエスト形式

リクエストの形式、ヘッダー、エンコーディングを明記します。

```markdown
### リクエスト形式

**Content-Type:**

すべてのPOST、PUT、PATCHリクエストは`Content-Type: application/json`ヘッダーを含める必要があります。

```http
POST /users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer YOUR_ACCESS_TOKEN

{
  "name": "John Doe",
  "email": "john@example.com"
}
```

**文字エンコーディング:**

- UTF-8を使用してください
- その他のエンコーディングはサポートされていません

**Accept ヘッダー:**

レスポンス形式を指定する場合、`Accept`ヘッダーを使用できます:

```http
GET /users HTTP/1.1
Host: api.example.com
Accept: application/json
```

現在は`application/json`のみサポートしています。
```

### レスポンス形式

レスポンスの構造とステータスコードの意味を説明します。

```markdown
### レスポンス形式

**成功レスポンス:**

すべての成功レスポンスは以下の構造を持ちます:

```json
{
  "success": true,
  "data": {
    // リソース固有のデータ
  },
  "meta": {
    "requestId": "req_abc123",
    "timestamp": "2026-01-28T12:00:00Z"
  }
}
```

**エラーレスポンス:**

エラーが発生した場合、以下の構造でレスポンスが返されます:

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力データが不正です",
    "details": [
      {
        "field": "email",
        "message": "有効なメールアドレスを入力してください"
      }
    ]
  },
  "meta": {
    "requestId": "req_abc123",
    "timestamp": "2026-01-28T12:00:00Z"
  }
}
```

**HTTPステータスコード:**

| コード | 説明 | 使用例 |
|-------|------|--------|
| 200 | OK | リソースの取得成功 |
| 201 | Created | リソースの作成成功 |
| 204 | No Content | リソースの削除成功 |
| 400 | Bad Request | 不正なリクエスト |
| 401 | Unauthorized | 認証が必要 |
| 403 | Forbidden | アクセス権限なし |
| 404 | Not Found | リソースが存在しない |
| 409 | Conflict | リソースの競合 |
| 422 | Unprocessable Entity | バリデーションエラー |
| 429 | Too Many Requests | レート制限超過 |
| 500 | Internal Server Error | サーバーエラー |
| 503 | Service Unavailable | サービス利用不可 |
```

### 日付・時刻の形式

APIで使用する日付・時刻の形式を統一し、明記します。

```markdown
### 日付・時刻の形式

**形式:**

すべての日付・時刻はISO 8601形式で表現されます:

```
YYYY-MM-DDTHH:mm:ssZ
```

例:
```
2026-01-28T12:00:00Z
```

**タイムゾーン:**

- すべての日付・時刻はUTC（協定世界時）で表現されます
- タイムゾーンは常に`Z`サフィックスで示されます
- ローカル時刻への変換はクライアント側で行ってください

**日付のみの場合:**

日付のみを扱う場合（例: 生年月日）は以下の形式を使用します:

```
YYYY-MM-DD
```

例:
```
1990-01-15
```

**時刻のみの場合:**

時刻のみを扱う場合（例: 営業時間）は以下の形式を使用します:

```
HH:mm:ss
```

例:
```
09:30:00
```
```

### ページネーション

リスト取得APIのページネーション方式を説明します。

```markdown
### ページネーション

リソースのリスト取得エンドポイントは、ページネーションをサポートします。

**カーソルベースのページネーション:**

```http
GET /users?limit=20&cursor=eyJpZCI6MTAwfQ HTTP/1.1
```

**パラメータ:**

| パラメータ | 型 | デフォルト | 説明 |
|----------|---|-----------|------|
| limit | integer | 20 | 1ページあたりの件数（最大100） |
| cursor | string | - | 次のページのカーソル |

**レスポンス例:**

```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "name": "John Doe"
    },
    {
      "id": 2,
      "name": "Jane Smith"
    }
  ],
  "pagination": {
    "limit": 20,
    "hasMore": true,
    "nextCursor": "eyJpZCI6MjB9"
  },
  "meta": {
    "requestId": "req_abc123",
    "timestamp": "2026-01-28T12:00:00Z"
  }
}
```

**次のページを取得する例:**

```http
GET /users?limit=20&cursor=eyJpZCI6MjB9 HTTP/1.1
```

`hasMore`が`false`の場合、これ以上のページは存在しません。
```

### フィルタリングとソート

リスト取得APIでのフィルタリングとソートの方法を説明します。

```markdown
### フィルタリング

リソースのフィルタリングは、クエリパラメータで行います。

**基本的なフィルタリング:**

```http
GET /users?status=active&role=admin HTTP/1.1
```

**複数値のフィルタリング:**

同じパラメータを複数回指定することで、OR条件でフィルタリングできます:

```http
GET /users?role=admin&role=editor HTTP/1.1
```

**範囲指定:**

日付や数値の範囲指定には`_gte`（以上）、`_lte`（以下）サフィックスを使用します:

```http
GET /users?created_at_gte=2026-01-01&created_at_lte=2026-01-31 HTTP/1.1
```

**部分一致検索:**

`_contains`サフィックスで部分一致検索ができます:

```http
GET /users?name_contains=John HTTP/1.1
```

### ソート

ソートは`sort`パラメータで指定します。

**昇順:**

```http
GET /users?sort=name HTTP/1.1
```

**降順:**

フィールド名の前に`-`を付けます:

```http
GET /users?sort=-created_at HTTP/1.1
```

**複数フィールドのソート:**

カンマ区切りで複数のフィールドを指定できます:

```http
GET /users?sort=-created_at,name HTTP/1.1
```
```

## 認証・認可の文書化

認証は、APIセキュリティの基盤です。認証方式、認証フロー、トークン管理を明確に文書化することで、開発者が安全にAPIを利用できるようにします。

### 認証方式の選択と文書化

一般的な認証方式と、それぞれの文書化方法を見ていきます。

#### 1. APIキー認証

最もシンプルな認証方式です。内部APIや信頼できるクライアント向けに使用されます。

```markdown
## 認証

### APIキー認証

このAPIは、APIキーベースの認証を使用します。APIキーは管理画面から発行できます。

#### APIキーの取得

1. [管理画面](https://dashboard.example.com)にログイン
2. 「設定」→「APIキー」に移動
3. 「新しいAPIキーを作成」をクリック
4. APIキーをコピーして安全に保管

**重要:** APIキーは秘密情報です。公開リポジトリやクライアントサイドコードに含めないでください。

#### APIキーの使用

APIキーは`X-API-Key`ヘッダーに含めます:

```http
GET /users HTTP/1.1
Host: api.example.com
X-API-Key: sk_live_abc123def456
```

#### APIキーの種類

| 種類 | プレフィックス | 用途 |
|-----|--------------|------|
| 本番用 | `sk_live_` | 本番環境 |
| テスト用 | `sk_test_` | 開発・テスト環境 |

**例:**

```bash
# 本番環境
curl -H "X-API-Key: sk_live_abc123def456" \
  https://api.example.com/v1/users

# テスト環境
curl -H "X-API-Key: sk_test_xyz789ghi012" \
  https://api-staging.example.com/v1/users
```

#### セキュリティのベストプラクティス

- APIキーは環境変数で管理する
- 定期的にAPIキーをローテーションする
- 不要になったAPIキーは削除する
- APIキーごとにアクセス権限を設定する
```
```

#### 2. JWT（JSON Web Token）認証

モダンなWebアプリケーションやモバイルアプリで広く使用される認証方式です。

```markdown
## 認証

### JWT認証

このAPIは、JWT（JSON Web Token）ベースの認証を使用します。

#### 認証フロー

```mermaid
sequenceDiagram
    participant Client
    participant API

    Client->>API: POST /auth/login (email, password)
    API->>Client: アクセストークン + リフレッシュトークン
    Client->>API: GET /users (Authorization: Bearer <token>)
    API->>Client: ユーザーデータ
    Note over Client,API: トークン有効期限切れ
    Client->>API: POST /auth/refresh (リフレッシュトークン)
    API->>Client: 新しいアクセストークン
```

#### ステップ1: ログイン

ユーザーのメールアドレスとパスワードで認証します:

```http
POST /auth/login HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "your_password"
}
```

**成功レスポンス (200 OK):**

```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expiresIn": 900,
    "tokenType": "Bearer"
  }
}
```

**エラーレスポンス (401 Unauthorized):**

```json
{
  "success": false,
  "error": {
    "code": "INVALID_CREDENTIALS",
    "message": "メールアドレスまたはパスワードが正しくありません"
  }
}
```

#### ステップ2: アクセストークンの使用

取得したアクセストークンを`Authorization`ヘッダーに含めます:

```http
GET /users HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

#### ステップ3: トークンのリフレッシュ

アクセストークンの有効期限が切れた場合、リフレッシュトークンを使用して新しいアクセストークンを取得します:

```http
POST /auth/refresh HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**成功レスポンス (200 OK):**

```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expiresIn": 900,
    "tokenType": "Bearer"
  }
}
```

#### トークンの有効期限

| トークン | 有効期限 | 用途 |
|---------|---------|------|
| アクセストークン | 15分 | API呼び出し |
| リフレッシュトークン | 7日 | アクセストークンの更新 |

#### トークンの保存

**Web アプリケーション:**
- アクセストークン: メモリ内（変数）
- リフレッシュトークン: HttpOnly Cookie（推奨）または localStorage

**モバイルアプリケーション:**
- アクセストークン: メモリ内
- リフレッシュトークン: Keychain（iOS）/ Keystore（Android）

**重要:** XSS攻撃を防ぐため、トークンをlocalStorageに保存する場合は十分な注意が必要です。

#### ログアウト

```http
POST /auth/logout HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

ログアウトにより、リフレッシュトークンが無効化されます。
```
```

#### 3. OAuth 2.0認証

サードパーティ開発者向けのAPIで使用される認証方式です。

```markdown
## 認証

### OAuth 2.0

このAPIは、OAuth 2.0の認可コードフローをサポートしています。

#### OAuth 2.0フロー

```mermaid
sequenceDiagram
    participant User
    participant Client
    participant API

    User->>Client: アプリ連携を開始
    Client->>API: 認可リクエスト
    API->>User: ログイン画面表示
    User->>API: ログイン + 権限承認
    API->>Client: 認可コード
    Client->>API: トークンリクエスト（認可コード）
    API->>Client: アクセストークン + リフレッシュトークン
    Client->>API: APIリクエスト（アクセストークン）
    API->>Client: レスポンス
```

#### 1. アプリケーションの登録

まず、[開発者ポータル](https://developers.example.com)でアプリケーションを登録します:

1. 「新しいアプリケーション」を作成
2. アプリケーション名を入力
3. リダイレクトURIを設定（例: `https://your-app.com/auth/callback`）
4. Client IDとClient Secretを取得

#### 2. 認可リクエスト

ユーザーを以下のURLにリダイレクトします:

```
GET https://api.example.com/oauth/authorize
  ?client_id=YOUR_CLIENT_ID
  &redirect_uri=https://your-app.com/auth/callback
  &response_type=code
  &scope=users:read users:write
  &state=RANDOM_STATE_STRING
```

**パラメータ:**

| パラメータ | 必須 | 説明 |
|----------|-----|------|
| client_id | Yes | アプリケーションのClient ID |
| redirect_uri | Yes | 認可後のリダイレクト先 |
| response_type | Yes | `code`固定 |
| scope | Yes | 要求するスコープ（スペース区切り） |
| state | Yes | CSRF対策用のランダム文字列 |

#### 3. 認可コードの取得

ユーザーが権限を承認すると、以下のようにリダイレクトされます:

```
https://your-app.com/auth/callback
  ?code=AUTHORIZATION_CODE
  &state=RANDOM_STATE_STRING
```

`state`パラメータが一致することを確認してください。

#### 4. アクセストークンの取得

認可コードを使用してアクセストークンを取得します:

```http
POST /oauth/token HTTP/1.1
Host: api.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=AUTHORIZATION_CODE
&redirect_uri=https://your-app.com/auth/callback
&client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
```

**成功レスポンス (200 OK):**

```json
{
  "access_token": "ya29.a0AfH6SMB...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "1//0gHK9F8mQ...",
  "scope": "users:read users:write"
}
```

#### 5. APIリクエスト

取得したアクセストークンを使用してAPIを呼び出します:

```http
GET /users HTTP/1.1
Host: api.example.com
Authorization: Bearer ya29.a0AfH6SMB...
```

#### スコープ一覧

| スコープ | 説明 |
|---------|------|
| users:read | ユーザー情報の読み取り |
| users:write | ユーザー情報の作成・更新 |
| users:delete | ユーザーの削除 |
| teams:read | チーム情報の読み取り |
| teams:write | チーム情報の作成・更新 |

#### トークンのリフレッシュ

アクセストークンの有効期限が切れた場合、リフレッシュトークンを使用します:

```http
POST /oauth/token HTTP/1.1
Host: api.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=1//0gHK9F8mQ...
&client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
```
```

### 認可（Authorization）

認証（Authentication）と認可（Authorization）は異なる概念です。認証は「誰であるか」を確認し、認可は「何ができるか」を決定します。

```markdown
## 認可

### ロールベースアクセス制御（RBAC）

このAPIは、ロールベースのアクセス制御を実装しています。

#### ロール一覧

| ロール | 説明 | 権限 |
|-------|------|------|
| admin | 管理者 | すべての操作が可能 |
| editor | 編集者 | リソースの作成・更新・削除 |
| viewer | 閲覧者 | リソースの読み取りのみ |
| guest | ゲスト | 公開リソースの読み取りのみ |

#### ロールの確認

現在のユーザーのロールは、`/auth/me`エンドポイントで確認できます:

```http
GET /auth/me HTTP/1.1
Host: api.example.com
Authorization: Bearer YOUR_ACCESS_TOKEN
```

**レスポンス:**

```json
{
  "success": true,
  "data": {
    "id": "user_123",
    "email": "user@example.com",
    "role": "editor",
    "permissions": [
      "users:read",
      "users:write",
      "teams:read"
    ]
  }
}
```

#### エンドポイント別の必要なロール

| エンドポイント | メソッド | 必要なロール |
|-------------|---------|------------|
| /users | GET | viewer以上 |
| /users | POST | editor以上 |
| /users/:id | PUT | editor以上 |
| /users/:id | DELETE | admin |
| /teams | GET | viewer以上 |
| /teams | POST | admin |

#### 権限不足時のレスポンス

必要な権限がない場合、`403 Forbidden`が返されます:

```json
{
  "success": false,
  "error": {
    "code": "INSUFFICIENT_PERMISSIONS",
    "message": "この操作を実行する権限がありません",
    "details": {
      "required": "admin",
      "current": "editor"
    }
  }
}
```
```

## エンドポイント一覧の作成

エンドポイント一覧は、APIの全体像を把握するための重要なセクションです。リソース別に整理し、各エンドポイントの概要を示します。

### エンドポイント一覧の構成

**基本的な構成要素:**
1. HTTPメソッド
2. エンドポイントパス
3. 簡単な説明
4. 認証の要否
5. 必要な権限

### エンドポイント一覧の例

```markdown
## エンドポイント一覧

### ユーザー管理

#### ユーザー一覧取得

```http
GET /users
```

**概要:** ユーザーの一覧を取得します。ページネーション、フィルタリング、ソートをサポートします。

**認証:** 必要（viewer以上）

**主なクエリパラメータ:**
- `limit`: 取得件数（デフォルト: 20、最大: 100）
- `cursor`: ページネーションカーソル
- `status`: ステータスフィルタ（active/inactive）
- `sort`: ソート順（created_at/-created_at/name/-name）

---

#### ユーザー詳細取得

```http
GET /users/:id
```

**概要:** 指定されたIDのユーザー情報を取得します。

**認証:** 必要（viewer以上）

**パスパラメータ:**
- `id`: ユーザーID

---

#### ユーザー作成

```http
POST /users
```

**概要:** 新しいユーザーを作成します。

**認証:** 必要（editor以上）

**必須フィールド:**
- `email`: メールアドレス
- `name`: ユーザー名
- `password`: パスワード（8文字以上）

---

#### ユーザー更新

```http
PUT /users/:id
```

**概要:** 指定されたIDのユーザー情報を更新します。

**認証:** 必要（editor以上）

**パスパラメータ:**
- `id`: ユーザーID

**更新可能なフィールド:**
- `name`: ユーザー名
- `email`: メールアドレス
- `status`: ステータス（active/inactive）
- `role`: ロール（admin/editor/viewer）

---

#### ユーザー削除

```http
DELETE /users/:id
```

**概要:** 指定されたIDのユーザーを削除します。論理削除（ソフトデリート）が実行されます。

**認証:** 必要（admin）

**パスパラメータ:**
- `id`: ユーザーID

---

### チーム管理

#### チーム一覧取得

```http
GET /teams
```

**概要:** チームの一覧を取得します。

**認証:** 必要（viewer以上）

---

#### チーム作成

```http
POST /teams
```

**概要:** 新しいチームを作成します。

**認証:** 必要（admin）

**必須フィールド:**
- `name`: チーム名
- `organizationId`: 所属する組織ID

---

#### チームメンバー追加

```http
POST /teams/:id/members
```

**概要:** チームにメンバーを追加します。

**認証:** 必要（editor以上）

**パスパラメータ:**
- `id`: チームID

**必須フィールド:**
- `userId`: 追加するユーザーID
- `role`: チーム内ロール（owner/member）

---

### 認証

#### ログイン

```http
POST /auth/login
```

**概要:** メールアドレスとパスワードで認証し、アクセストークンを取得します。

**認証:** 不要

**必須フィールド:**
- `email`: メールアドレス
- `password`: パスワード

---

#### トークンリフレッシュ

```http
POST /auth/refresh
```

**概要:** リフレッシュトークンを使用して新しいアクセストークンを取得します。

**認証:** リフレッシュトークンが必要

**必須フィールド:**
- `refreshToken`: リフレッシュトークン

---

#### ログアウト

```http
POST /auth/logout
```

**概要:** ログアウトし、リフレッシュトークンを無効化します。

**認証:** 必要

---

### プロフィール

#### 現在のユーザー情報取得

```http
GET /auth/me
```

**概要:** 現在認証されているユーザーの情報を取得します。

**認証:** 必要

---

#### プロフィール更新

```http
PUT /auth/me
```

**概要:** 現在のユーザーのプロフィールを更新します。

**認証:** 必要

**更新可能なフィールド:**
- `name`: ユーザー名
- `bio`: 自己紹介
- `avatarUrl`: プロフィール画像URL
```

### テーブル形式のエンドポイント一覧

より簡潔にエンドポイント一覧を提示する場合、テーブル形式も効果的です。

```markdown
## エンドポイント一覧（概要）

### ユーザー管理

| メソッド | エンドポイント | 説明 | 認証 | 権限 |
|---------|---------------|------|-----|------|
| GET | `/users` | ユーザー一覧取得 | 必要 | viewer以上 |
| GET | `/users/:id` | ユーザー詳細取得 | 必要 | viewer以上 |
| POST | `/users` | ユーザー作成 | 必要 | editor以上 |
| PUT | `/users/:id` | ユーザー更新 | 必要 | editor以上 |
| DELETE | `/users/:id` | ユーザー削除 | 必要 | admin |
| GET | `/users/:id/teams` | ユーザーの所属チーム取得 | 必要 | viewer以上 |

### チーム管理

| メソッド | エンドポイント | 説明 | 認証 | 権限 |
|---------|---------------|------|-----|------|
| GET | `/teams` | チーム一覧取得 | 必要 | viewer以上 |
| GET | `/teams/:id` | チーム詳細取得 | 必要 | viewer以上 |
| POST | `/teams` | チーム作成 | 必要 | admin |
| PUT | `/teams/:id` | チーム更新 | 必要 | editor以上 |
| DELETE | `/teams/:id` | チーム削除 | 必要 | admin |
| GET | `/teams/:id/members` | チームメンバー一覧 | 必要 | viewer以上 |
| POST | `/teams/:id/members` | メンバー追加 | 必要 | editor以上 |
| DELETE | `/teams/:id/members/:userId` | メンバー削除 | 必要 | editor以上 |

### 認証

| メソッド | エンドポイント | 説明 | 認証 | 権限 |
|---------|---------------|------|-----|------|
| POST | `/auth/login` | ログイン | 不要 | - |
| POST | `/auth/refresh` | トークンリフレッシュ | リフレッシュトークン | - |
| POST | `/auth/logout` | ログアウト | 必要 | - |
| GET | `/auth/me` | 現在のユーザー情報 | 必要 | - |
| PUT | `/auth/me` | プロフィール更新 | 必要 | - |
| PUT | `/auth/password` | パスワード変更 | 必要 | - |

詳細は各エンドポイントのリファレンスを参照してください。
```

## バージョニング戦略

APIは進化し続けます。既存のクライアントに影響を与えずに新機能を追加したり、仕様を変更したりするために、バージョニング戦略が必要です。

### バージョニングの方法

#### 1. URLパスでのバージョニング（推奨）

最も明示的で、広く採用されている方法です。

```markdown
## バージョニング

### バージョン指定

このAPIは、URLパスでバージョンを指定します。

**現在のバージョン:** v1

```
https://api.example.com/v1/users
```

### バージョンポリシー

- 現在サポートしているバージョン: **v1**
- 各バージョンのサポート期間: 最低12ヶ月
- 非推奨の告知期間: 最低6ヶ月前

### バージョンの変更が必要な場合

以下の変更を行う場合、新しいバージョンをリリースします:

**破壊的変更（Breaking Changes）:**
- エンドポイントの削除
- 必須パラメータの追加
- レスポンス構造の変更
- データ型の変更
- エラーコードの変更

**非破壊的変更（Non-Breaking Changes）:**

以下の変更は既存のバージョン内で実施されます:
- 新しいエンドポイントの追加
- オプションパラメータの追加
- レスポンスへの新しいフィールドの追加
- 新しいHTTPヘッダーの追加

### 移行ガイド

新しいバージョンへの移行方法は、[移行ガイド](/migration-guide)を参照してください。

### バージョン別のドキュメント

- [v1 ドキュメント](/docs/v1) (現在)
- [v2 ドキュメント (ベータ版)](/docs/v2-beta)
```

#### 2. ヘッダーでのバージョニング

GitHubなどで採用されている方法です。

```markdown
## バージョニング

### バージョン指定

このAPIは、`Accept`ヘッダーでバージョンを指定します。

```http
GET /users HTTP/1.1
Host: api.example.com
Accept: application/vnd.example.v1+json
```

**バージョン指定なしの場合:**

`Accept`ヘッダーでバージョンを指定しない場合、最新の安定版が使用されます:

```http
GET /users HTTP/1.1
Host: api.example.com
Accept: application/json
```

現在のデフォルトバージョン: **v1**

### バージョン一覧

| バージョン | ステータス | Accept ヘッダー |
|----------|----------|----------------|
| v1 | 安定版 | `application/vnd.example.v1+json` |
| v2 | ベータ版 | `application/vnd.example.v2+json` |
```

### 非推奨（Deprecation）の通知

APIの一部機能を非推奨にする場合、明確に通知する必要があります。

```markdown
## 非推奨機能

### 非推奨となったエンドポイント

以下のエンドポイントは非推奨となり、将来のバージョンで削除されます。

#### GET /users/list （非推奨）

**非推奨日:** 2026-01-01
**削除予定日:** 2026-07-01
**代替:** `GET /users`を使用してください

**レスポンスヘッダーでの通知:**

非推奨のエンドポイントを呼び出すと、以下のヘッダーが返されます:

```http
Deprecation: true
Sunset: Tue, 01 Jul 2026 00:00:00 GMT
Link: <https://docs.example.com/migration-guide>; rel="deprecation"
```

### 移行手順

```typescript
// ❌ 非推奨
const response = await fetch('https://api.example.com/v1/users/list');

// ✅ 推奨
const response = await fetch('https://api.example.com/v1/users');
```
```

### 変更履歴（Changelog）

APIの変更履歴を記録し、開発者が変更を追跡できるようにします。

```markdown
## 変更履歴

### v1.2.0 (2026-01-28)

#### 追加
- `/users/:id/avatar`エンドポイントを追加（プロフィール画像アップロード）
- `/teams`エンドポイントに`sort`パラメータを追加

#### 変更
- `/users`エンドポイントのデフォルトソート順を`created_at`から`name`に変更
- ページネーションのデフォルト`limit`を10から20に変更

#### 修正
- `/auth/refresh`でリフレッシュトークンが無効な場合のエラーメッセージを改善
- `/users/:id`で存在しないユーザーIDを指定した際の404レスポンスを修正

#### 非推奨
- なし

---

### v1.1.0 (2026-01-15)

#### 追加
- `/auth/me`エンドポイントを追加
- OAuth 2.0サポート

#### 変更
- JWTトークンの有効期限を30分から15分に短縮

#### 修正
- メールアドレスのバリデーションを改善

---

### v1.0.0 (2025-12-01)

初回リリース
```

## API仕様書のフォーマットと配置

API仕様書のフォーマットと配置場所は、保守性と利便性に大きく影響します。

### フォーマットの選択

#### 1. Markdown形式

**メリット:**
- 人間が読みやすい
- Git diff が見やすい
- バージョン管理が容易
- GitHub/GitLabで自動レンダリング

**デメリット:**
- 構造化データとして扱いにくい
- ツールでの自動検証が困難

**使用例:**

```markdown
プロジェクト構造:
docs/
├── api/
│   ├── README.md              # API概要
│   ├── authentication.md      # 認証
│   ├── endpoints/
│   │   ├── users.md          # ユーザーエンドポイント
│   │   ├── teams.md          # チームエンドポイント
│   │   └── auth.md           # 認証エンドポイント
│   └── changelog.md          # 変更履歴
```

#### 2. OpenAPI（Swagger）形式

**メリット:**
- 業界標準
- ツールエコシステムが豊富
- コードからの自動生成が可能
- Swagger UIで対話的に試せる
- クライアントコードの自動生成

**デメリット:**
- YAMLの記述が複雑
- 人間が直接読むには不向き

**使用例:**

```yaml
openapi: 3.0.0
info:
  title: User Management API
  version: 1.0.0
  description: ユーザーアカウント管理API

servers:
  - url: https://api.example.com/v1
    description: 本番環境
  - url: https://api-staging.example.com/v1
    description: ステージング環境

paths:
  /users:
    get:
      summary: ユーザー一覧取得
      description: ユーザーの一覧をページネーション形式で取得します
      security:
        - bearerAuth: []
      parameters:
        - name: limit
          in: query
          description: 取得件数
          schema:
            type: integer
            default: 20
            maximum: 100
        - name: cursor
          in: query
          description: ページネーションカーソル
          schema:
            type: string
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                type: object
                properties:
                  success:
                    type: boolean
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/User'
                  pagination:
                    $ref: '#/components/schemas/Pagination'

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    User:
      type: object
      properties:
        id:
          type: string
          example: "user_123"
        email:
          type: string
          format: email
          example: "user@example.com"
        name:
          type: string
          example: "John Doe"
        role:
          type: string
          enum: [admin, editor, viewer]
          example: "editor"
        createdAt:
          type: string
          format: date-time
          example: "2026-01-28T12:00:00Z"

    Pagination:
      type: object
      properties:
        limit:
          type: integer
        hasMore:
          type: boolean
        nextCursor:
          type: string
```

#### 3. ハイブリッドアプローチ（推奨）

MarkdownとOpenAPIを組み合わせる方法が最も実用的です。

```markdown
プロジェクト構造:
docs/
├── api/
│   ├── README.md              # API概要（人間向け）
│   ├── getting-started.md     # はじめに
│   ├── authentication.md      # 認証の詳細説明
│   ├── best-practices.md      # ベストプラクティス
│   ├── changelog.md           # 変更履歴
│   └── openapi.yaml           # OpenAPI仕様（機械可読）
```

このアプローチでは:
- **Markdown**: 概念的な説明、認証フロー、ベストプラクティス
- **OpenAPI**: エンドポイントの詳細仕様、データモデル、バリデーションルール

### 配置場所の選択

#### パターン1: ドキュメント専用リポジトリ

大規模なプロジェクトや公開APIの場合、ドキュメント専用のリポジトリを作成することがあります。

```
組織/
├── api-server/           # APIサーバーのコード
├── web-app/             # Webアプリケーション
└── api-docs/            # API ドキュメント専用リポジトリ
    ├── docs/
    │   ├── getting-started.md
    │   ├── authentication.md
    │   └── ...
    ├── openapi.yaml
    └── package.json     # ドキュメントサイト生成用
```

**メリット:**
- ドキュメントの独立したバージョン管理
- 複数の言語版を管理しやすい
- 外部コントリビューターが貢献しやすい

**デメリット:**
- コードとドキュメントの同期が困難
- 変更が複数のリポジトリに分散

#### パターン2: モノレポ内のdocsディレクトリ（推奨）

コードと同じリポジトリにドキュメントを配置します。

```
project/
├── src/                 # ソースコード
├── tests/              # テスト
├── docs/               # ドキュメント
│   ├── api/
│   │   ├── README.md
│   │   ├── authentication.md
│   │   └── openapi.yaml
│   ├── architecture/
│   └── guides/
├── README.md
└── package.json
```

**メリット:**
- コードとドキュメントが同期しやすい
- PRでコードとドキュメントを同時にレビュー
- ドキュメント更新の漏れを防げる

**デメリット:**
- リポジトリサイズが大きくなる
- 外部向けと内部向けの分離が困難

#### パターン3: コードに埋め込み（自動生成）

コードのコメントや型定義からドキュメントを自動生成します。

```typescript
/**
 * ユーザー一覧を取得します
 *
 * @route GET /users
 * @group Users - ユーザー管理
 * @param {integer} limit.query - 取得件数（デフォルト: 20、最大: 100）
 * @param {string} cursor.query - ページネーションカーソル
 * @returns {User[]} 200 - ユーザーの配列
 * @returns {Error} 401 - 認証エラー
 * @security JWT
 */
router.get('/users', async (req, res) => {
  // ...
});
```

**メリット:**
- コードとドキュメントの完全な同期
- 実装とドキュメントが乖離しない

**デメリット:**
- 詳細な説明を記述しにくい
- ツールへの依存

### ドキュメントサイトの公開

#### 静的サイトジェネレーター

**VitePress（推奨）:**

```bash
# インストール
npm install -D vitepress

# ドキュメントサイトの初期化
npx vitepress init
```

```javascript
// docs/.vitepress/config.js
export default {
  title: 'API Documentation',
  description: 'User Management API',
  themeConfig: {
    nav: [
      { text: 'ホーム', link: '/' },
      { text: 'API リファレンス', link: '/api/' },
      { text: 'ガイド', link: '/guides/' }
    ],
    sidebar: {
      '/api/': [
        {
          text: '基本情報',
          items: [
            { text: '概要', link: '/api/' },
            { text: 'はじめに', link: '/api/getting-started' },
            { text: '認証', link: '/api/authentication' }
          ]
        },
        {
          text: 'エンドポイント',
          items: [
            { text: 'ユーザー', link: '/api/endpoints/users' },
            { text: 'チーム', link: '/api/endpoints/teams' }
          ]
        }
      ]
    }
  }
}
```

**Docusaurus:**

Reactベースの強力なドキュメントサイトジェネレーター。

**MkDocs:**

Pythonベースのシンプルなツール。

#### Swagger UI

OpenAPI仕様から対話的なAPIドキュメントを生成します。

```yaml
# docker-compose.yml
version: '3'
services:
  swagger-ui:
    image: swaggerapi/swagger-ui
    ports:
      - "8080:8080"
    environment:
      SWAGGER_JSON: /api/openapi.yaml
    volumes:
      - ./docs/api/openapi.yaml:/api/openapi.yaml
```

#### GitHub Pages

静的サイトを無料でホスティングできます。

```yaml
# .github/workflows/deploy-docs.yml
name: Deploy Documentation

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm ci

      - name: Build documentation
        run: npm run docs:build

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./docs/.vitepress/dist
```

## データモデルの文書化

APIのデータモデルを明確に文書化することで、開発者はリクエストとレスポンスの構造を理解できます。

### データモデルの記述方法

```markdown
## データモデル

### User（ユーザー）

ユーザーアカウントを表すリソースです。

#### フィールド

| フィールド | 型 | 必須 | 説明 |
|----------|---|-----|------|
| id | string | Yes | ユーザーID（例: `user_123`） |
| email | string | Yes | メールアドレス（一意） |
| name | string | Yes | ユーザー名（1-50文字） |
| bio | string | No | 自己紹介（最大500文字） |
| avatarUrl | string | No | プロフィール画像URL |
| role | string | Yes | ロール（`admin`, `editor`, `viewer`） |
| status | string | Yes | ステータス（`active`, `inactive`, `suspended`） |
| emailVerified | boolean | Yes | メールアドレスが確認済みか |
| lastLoginAt | string | No | 最終ログイン日時（ISO 8601形式） |
| createdAt | string | Yes | 作成日時（ISO 8601形式） |
| updatedAt | string | Yes | 更新日時（ISO 8601形式） |

#### 例

```json
{
  "id": "user_123",
  "email": "john@example.com",
  "name": "John Doe",
  "bio": "Software Engineer",
  "avatarUrl": "https://example.com/avatars/john.jpg",
  "role": "editor",
  "status": "active",
  "emailVerified": true,
  "lastLoginAt": "2026-01-28T10:30:00Z",
  "createdAt": "2025-12-01T00:00:00Z",
  "updatedAt": "2026-01-28T10:30:00Z"
}
```

#### バリデーションルール

**email:**
- 形式: 有効なメールアドレス
- 一意性: システム内で一意
- 例: `user@example.com`

**name:**
- 長さ: 1-50文字
- 文字種: 英数字、日本語、スペース、ハイフン
- 例: `John Doe`, `山田 太郎`

**bio:**
- 長さ: 0-500文字
- 任意項目

**role:**
- 許可される値: `admin`, `editor`, `viewer`
- デフォルト: `viewer`

**status:**
- 許可される値: `active`, `inactive`, `suspended`
- デフォルト: `active`

---

### Team（チーム）

複数のユーザーをグループ化するリソースです。

#### フィールド

| フィールド | 型 | 必須 | 説明 |
|----------|---|-----|------|
| id | string | Yes | チームID（例: `team_456`） |
| name | string | Yes | チーム名（1-100文字） |
| description | string | No | チームの説明（最大1000文字） |
| organizationId | string | Yes | 所属する組織のID |
| memberCount | integer | Yes | メンバー数 |
| createdAt | string | Yes | 作成日時（ISO 8601形式） |
| updatedAt | string | Yes | 更新日時（ISO 8601形式） |

#### 例

```json
{
  "id": "team_456",
  "name": "Engineering Team",
  "description": "Product development team",
  "organizationId": "org_789",
  "memberCount": 12,
  "createdAt": "2025-12-01T00:00:00Z",
  "updatedAt": "2026-01-15T08:20:00Z"
}
```

---

### Pagination（ページネーション）

リスト取得APIのページネーション情報です。

#### フィールド

| フィールド | 型 | 必須 | 説明 |
|----------|---|-----|------|
| limit | integer | Yes | 1ページあたりの件数 |
| hasMore | boolean | Yes | 次のページが存在するか |
| nextCursor | string | No | 次のページのカーソル（hasMore=trueの場合のみ） |

#### 例

```json
{
  "limit": 20,
  "hasMore": true,
  "nextCursor": "eyJpZCI6MjB9"
}
```
```

### エンティティ間のリレーション

複数のエンティティ間の関係を図で示すと理解しやすくなります。

```markdown
## エンティティ関係図

```mermaid
erDiagram
    Organization ||--o{ Team : contains
    Team ||--o{ TeamMember : has
    User ||--o{ TeamMember : belongs

    Organization {
        string id PK
        string name
        datetime createdAt
    }

    Team {
        string id PK
        string name
        string organizationId FK
        datetime createdAt
    }

    User {
        string id PK
        string email
        string name
        string role
        datetime createdAt
    }

    TeamMember {
        string teamId FK
        string userId FK
        string role
        datetime joinedAt
    }
```

**リレーション:**

- 1つの組織（Organization）は複数のチーム（Team）を持つ
- 1つのチーム（Team）は複数のメンバー（TeamMember）を持つ
- 1人のユーザー（User）は複数のチーム（TeamMember）に所属できる
```

## エラーハンドリングの文書化

適切なエラーハンドリングの文書化は、開発者がエラーに対処するために不可欠です。

### エラーレスポンスの標準形式

```markdown
## エラーハンドリング

### エラーレスポンスの形式

すべてのエラーレスポンスは以下の構造を持ちます:

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "人間が読めるエラーメッセージ",
    "details": {} // エラーの詳細情報（オプション）
  },
  "meta": {
    "requestId": "req_abc123",
    "timestamp": "2026-01-28T12:00:00Z"
  }
}
```

### エラーコード一覧

#### 認証・認可エラー (401, 403)

| エラーコード | HTTPステータス | 説明 | 対処方法 |
|------------|--------------|------|---------|
| `AUTHENTICATION_REQUIRED` | 401 | 認証が必要です | 認証トークンを含めてリクエストしてください |
| `INVALID_TOKEN` | 401 | トークンが無効です | 新しいトークンを取得してください |
| `TOKEN_EXPIRED` | 401 | トークンの有効期限が切れています | リフレッシュトークンで更新してください |
| `INSUFFICIENT_PERMISSIONS` | 403 | 権限が不足しています | 必要な権限を持つユーザーでアクセスしてください |

**例:**

```json
{
  "success": false,
  "error": {
    "code": "TOKEN_EXPIRED",
    "message": "アクセストークンの有効期限が切れています",
    "details": {
      "expiredAt": "2026-01-28T12:00:00Z"
    }
  },
  "meta": {
    "requestId": "req_abc123",
    "timestamp": "2026-01-28T12:15:00Z"
  }
}
```

#### バリデーションエラー (400, 422)

| エラーコード | HTTPステータス | 説明 | 対処方法 |
|------------|--------------|------|---------|
| `VALIDATION_ERROR` | 422 | 入力データが不正です | エラーの詳細を確認して修正してください |
| `INVALID_FORMAT` | 400 | データ形式が不正です | 正しい形式でリクエストしてください |
| `MISSING_REQUIRED_FIELD` | 400 | 必須フィールドが不足しています | 必要なフィールドを含めてください |

**例:**

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力データが不正です",
    "details": [
      {
        "field": "email",
        "message": "有効なメールアドレスを入力してください",
        "value": "invalid-email"
      },
      {
        "field": "password",
        "message": "パスワードは8文字以上である必要があります",
        "value": "***"
      }
    ]
  },
  "meta": {
    "requestId": "req_abc123",
    "timestamp": "2026-01-28T12:00:00Z"
  }
}
```

#### リソースエラー (404, 409)

| エラーコード | HTTPステータス | 説明 | 対処方法 |
|------------|--------------|------|---------|
| `RESOURCE_NOT_FOUND` | 404 | リソースが見つかりません | URLやIDを確認してください |
| `RESOURCE_ALREADY_EXISTS` | 409 | リソースが既に存在します | 別の識別子を使用してください |
| `RESOURCE_CONFLICT` | 409 | リソースの状態が競合しています | リソースの状態を確認してください |

**例:**

```json
{
  "success": false,
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "指定されたユーザーが見つかりません",
    "details": {
      "resource": "User",
      "id": "user_999"
    }
  },
  "meta": {
    "requestId": "req_abc123",
    "timestamp": "2026-01-28T12:00:00Z"
  }
}
```

#### レート制限エラー (429)

| エラーコード | HTTPステータス | 説明 | 対処方法 |
|------------|--------------|------|---------|
| `RATE_LIMIT_EXCEEDED` | 429 | レート制限を超過しています | 時間をおいて再試行してください |

**例:**

```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "リクエスト数の上限を超えました",
    "details": {
      "limit": 100,
      "remaining": 0,
      "resetAt": "2026-01-28T13:00:00Z"
    }
  },
  "meta": {
    "requestId": "req_abc123",
    "timestamp": "2026-01-28T12:30:00Z"
  }
}
```

**レスポンスヘッダー:**

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1706446800
Retry-After: 1800
```

#### サーバーエラー (500, 503)

| エラーコード | HTTPステータス | 説明 | 対処方法 |
|------------|--------------|------|---------|
| `INTERNAL_SERVER_ERROR` | 500 | サーバー内部エラー | しばらく時間をおいて再試行してください |
| `SERVICE_UNAVAILABLE` | 503 | サービスが一時的に利用できません | メンテナンス情報を確認してください |

**例:**

```json
{
  "success": false,
  "error": {
    "code": "INTERNAL_SERVER_ERROR",
    "message": "サーバーでエラーが発生しました。しばらく時間をおいて再試行してください",
    "details": {
      "reference": "ERR-2026-01-28-12345"
    }
  },
  "meta": {
    "requestId": "req_abc123",
    "timestamp": "2026-01-28T12:00:00Z"
  }
}
```

### トラブルシューティング

#### よくあるエラーと解決方法

**1. 401 Unauthorized: INVALID_TOKEN**

**原因:**
- トークンの形式が不正
- トークンが改ざんされている
- トークンの署名が無効

**解決方法:**
```typescript
// ✅ 正しいトークンの使用
const response = await fetch('https://api.example.com/v1/users', {
  headers: {
    'Authorization': `Bearer ${accessToken}`
  }
});

// ❌ 間違った使用方法
// - "Bearer "プレフィックスの欠如
// - トークンの一部のみを送信
```

**2. 422 Validation Error: VALIDATION_ERROR**

**原因:**
- 必須フィールドが不足
- データ型が不正
- バリデーションルールに違反

**解決方法:**
```typescript
// ❌ 不正なリクエスト
await fetch('https://api.example.com/v1/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${accessToken}`
  },
  body: JSON.stringify({
    email: 'invalid-email', // 不正な形式
    name: '', // 空文字列
    password: '123' // 8文字未満
  })
});

// ✅ 正しいリクエスト
await fetch('https://api.example.com/v1/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${accessToken}`
  },
  body: JSON.stringify({
    email: 'user@example.com',
    name: 'John Doe',
    password: 'SecurePassword123'
  })
});
```

**3. 429 Too Many Requests: RATE_LIMIT_EXCEEDED**

**原因:**
- レート制限を超過

**解決方法:**
```typescript
// エクスポネンシャルバックオフでリトライ
async function fetchWithRetry(url: string, options: RequestInit, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    const response = await fetch(url, options);

    if (response.status !== 429) {
      return response;
    }

    // Retry-Afterヘッダーを確認
    const retryAfter = response.headers.get('Retry-After');
    const waitTime = retryAfter ? parseInt(retryAfter) * 1000 : Math.pow(2, i) * 1000;

    await new Promise(resolve => setTimeout(resolve, waitTime));
  }

  throw new Error('Max retries exceeded');
}
```
```

## レート制限の文書化

公開APIでは、レート制限を明確に文書化する必要があります。

```markdown
## レート制限

### 制限内容

このAPIは、APIの安定性を保つためにレート制限を実施しています。

**制限:**
- 1時間あたり100リクエスト（認証済みユーザー）
- 1時間あたり20リクエスト（未認証ユーザー）

**カウント方法:**
- IPアドレスまたはAPIキー単位でカウント
- スライディングウィンドウ方式

### レート制限ヘッダー

すべてのレスポンスには、以下のヘッダーが含まれます:

```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1706446800
```

**ヘッダーの説明:**

| ヘッダー | 説明 |
|---------|------|
| `X-RateLimit-Limit` | 1時間あたりの上限リクエスト数 |
| `X-RateLimit-Remaining` | 残りのリクエスト数 |
| `X-RateLimit-Reset` | レート制限がリセットされる時刻（Unixタイムスタンプ） |

### レート制限超過時のレスポンス

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1706446800
Retry-After: 1800
Content-Type: application/json

{
  "success": false,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "リクエスト数の上限を超えました",
    "details": {
      "limit": 100,
      "remaining": 0,
      "resetAt": "2026-01-28T13:00:00Z"
    }
  }
}
```

### レート制限の確認方法

```typescript
const response = await fetch('https://api.example.com/v1/users', {
  headers: {
    'Authorization': `Bearer ${accessToken}`
  }
});

const limit = response.headers.get('X-RateLimit-Limit');
const remaining = response.headers.get('X-RateLimit-Remaining');
const reset = response.headers.get('X-RateLimit-Reset');

console.log(`制限: ${limit}, 残り: ${remaining}, リセット時刻: ${new Date(parseInt(reset) * 1000)}`);
```

### ベストプラクティス

1. **リクエストのバッチ処理**
   - 複数のリソースを1回のリクエストで取得
   - ページネーションの活用

2. **キャッシュの活用**
   - `Cache-Control`ヘッダーの確認
   - 不要なリクエストを削減

3. **レート制限の監視**
   - `X-RateLimit-Remaining`を確認
   - 残りが少ない場合は処理を調整

4. **エクスポネンシャルバックオフ**
   - 429エラー時は指数的に待機時間を増やす
   - `Retry-After`ヘッダーを尊重
```

## この章のまとめ

### 学んだこと

この章では、API仕様書の基本について学びました:

1. **API仕様書の目的**
   - APIの機能と使用方法を伝えるドキュメント
   - 開発者がコードを読まずにAPIを理解できるようにする
   - フロントエンド・モバイル・サードパーティ開発者向け

2. **基本構成**
   - 概要、認証、基本情報、エンドポイント一覧、エンドポイント詳細
   - データモデル、エラーハンドリング、レート制限
   - バージョニング、変更履歴

3. **基本情報の文書化**
   - ベースURL（環境別）
   - リクエスト・レスポンス形式
   - 日付時刻の形式
   - ページネーション、フィルタリング、ソート

4. **認証・認可**
   - APIキー認証
   - JWT認証
   - OAuth 2.0
   - ロールベースアクセス制御（RBAC）

5. **エンドポイント一覧**
   - リソース別の整理
   - HTTPメソッド、パス、説明、認証、権限の明記
   - テーブル形式での表現

6. **バージョニング戦略**
   - URLパスでのバージョニング（推奨）
   - ヘッダーでのバージョニング
   - 非推奨機能の通知
   - 変更履歴の記録

7. **フォーマットと配置**
   - Markdown形式（人間向け）
   - OpenAPI形式（機械可読）
   - ハイブリッドアプローチ（推奨）
   - ドキュメントサイトの公開

8. **データモデルの文書化**
   - フィールドの型、必須/任意、説明
   - バリデーションルール
   - リレーションの図示

9. **エラーハンドリング**
   - 標準的なエラーレスポンス形式
   - エラーコード一覧
   - トラブルシューティング

10. **レート制限**
    - 制限内容の明記
    - レート制限ヘッダー
    - ベストプラクティス

### チェックリスト

- [ ] API仕様書の目的と読者を理解している
- [ ] 基本的な構成要素を含めることができる
- [ ] ベースURL、リクエスト形式、レスポンス形式を明記できる
- [ ] 認証方式を明確に文書化できる
- [ ] エンドポイント一覧をわかりやすく整理できる
- [ ] バージョニング戦略を定義できる
- [ ] 適切なフォーマットと配置場所を選択できる
- [ ] データモデルを明確に記述できる
- [ ] エラーレスポンスを標準化して文書化できる
- [ ] レート制限を明確に説明できる

## 次のステップ

第8章では、API仕様書の基本構成と各セクションの書き方を学びました。次の第9章「エンドポイント設計と文書化」では、さらに詳細なエンドポイントの設計原則と文書化方法を学びます:

- RESTful APIの設計原則
- エンドポイント命名規則
- HTTPメソッドの適切な選択
- ステータスコードの使い分け
- リクエスト・レスポンスの詳細な文書化
- パラメータのバリデーション

**関連リソース:**

- [OpenAPI Specification](https://swagger.io/specification/)
- [REST API Tutorial](https://restfulapi.net/)
- [HTTP Status Codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
- [JSON Web Tokens (JWT)](https://jwt.io/)
- [OAuth 2.0](https://oauth.net/2/)
