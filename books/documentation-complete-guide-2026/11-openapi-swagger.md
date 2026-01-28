---
title: "OpenAPI/Swagger活用"
---

# 第11章: OpenAPI/Swagger活用

この章では、OpenAPI Specification（以下OpenAPI）を活用したAPI仕様書の自動化と、Swagger UIによる可視化の実践的な方法を学びます。OpenAPIは、RESTful APIを記述するための標準仕様であり、機械可読な形式でAPIを定義することで、ドキュメント生成、クライアントコード生成、モックサーバー構築など、さまざまな自動化を可能にします。

## この章で学ぶこと

- OpenAPI Specificationの基本構造と記述方法
- YAML/JSON形式での仕様書作成
- Swagger UIによる対話的なドキュメント生成
- コードからの仕様書自動生成とその逆
- 実践的なOpenAPI活用パターン

## なぜOpenAPIが重要か

従来のAPI仕様書は、Word文書やMarkdownファイルとして手作業で作成・更新されることが多く、以下の課題がありました：

1. **ドキュメントとコードの乖離**: コードを更新してもドキュメントの更新を忘れる
2. **可読性の問題**: テキストベースのドキュメントは検索や理解が困難
3. **手動テストの負担**: APIを試すためにcurlコマンドを手打ちする必要がある
4. **クライアント実装の手間**: APIクライアントコードを手作業で実装

OpenAPIは、これらの課題を解決します。機械可読な形式でAPIを定義することで、ドキュメント、モックサーバー、クライアントコードなどを自動生成でき、API開発のライフサイクル全体を効率化できます。

## 前提知識

この章を理解するために、以下の知識があると望ましいです：

- RESTful APIの基本原則（第9章）
- HTTPメソッド、ステータスコード、リクエスト/レスポンスの概念（第10章）
- YAML形式の基本的な構文
- JSONの基本的な構文

---

## OpenAPI Specificationの基礎

### OpenAPIとは

OpenAPI Specification（OAS）は、RESTful APIを記述するための標準仕様です。OpenAPI Initiative（Linux Foundationのプロジェクト）によって管理されており、多くの企業やツールがサポートしています。

**公式サイト**: https://www.openapis.org/

OpenAPIの前身は「Swagger Specification」でしたが、2016年にOpenAPI Specificationとして標準化されました。現在の最新バージョンは3.1.0（2021年リリース）ですが、広く使われているのはOpenAPI 3.0系です。

### OpenAPIのバージョン

| バージョン | リリース年 | 主な特徴 |
|-----------|-----------|---------|
| Swagger 2.0 | 2014 | 初期の標準仕様 |
| OpenAPI 3.0.0 | 2017 | Swaggerから改名、大幅な改善 |
| OpenAPI 3.0.3 | 2020 | バグ修正と明確化 |
| OpenAPI 3.1.0 | 2021 | JSON Schema準拠 |

本章では、最も広く使われているOpenAPI 3.0系を中心に解説します。

### OpenAPIの基本構造

OpenAPI仕様書は、以下の主要セクションで構成されます：

```yaml
openapi: 3.0.3                    # OpenAPIのバージョン（必須）
info:                             # API全体のメタ情報（必須）
  title: My API
  version: 1.0.0
servers:                          # APIサーバーのURL
  - url: https://api.example.com
paths:                            # APIエンドポイントの定義（必須）
  /users:
    get:
      summary: List users
components:                       # 再利用可能なコンポーネント
  schemas:
    User:
      type: object
security:                         # セキュリティ設定
  - bearerAuth: []
```

---

## OpenAPI仕様書の作成

### 最小限のOpenAPI仕様書

まずは、最もシンプルなOpenAPI仕様書から始めましょう。以下は、単一のエンドポイントを持つAPI仕様書の例です：

```yaml
openapi: 3.0.3
info:
  title: Simple User API
  description: ユーザー管理のためのシンプルなAPI
  version: 1.0.0
  contact:
    name: API Support
    email: support@example.com
    url: https://support.example.com

servers:
  - url: https://api.example.com/v1
    description: 本番環境
  - url: https://staging-api.example.com/v1
    description: ステージング環境

paths:
  /users:
    get:
      summary: ユーザー一覧を取得
      description: 登録されているすべてのユーザーを取得します
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                type: array
                items:
                  type: object
                  properties:
                    id:
                      type: string
                      example: "user_123"
                    name:
                      type: string
                      example: "山田太郎"
                    email:
                      type: string
                      format: email
                      example: "yamada@example.com"
```

この仕様書は、以下の情報を定義しています：

- API名、説明、バージョン
- 本番環境とステージング環境のURL
- `/users`エンドポイントのGETメソッド
- レスポンスの構造と例

### infoセクション: APIのメタ情報

`info`セクションには、API全体のメタ情報を記述します：

```yaml
info:
  title: User Management API                # API名（必須）
  description: |                             # API説明（Markdown可）
    ユーザー管理のためのRESTful API。

    ## 主な機能
    - ユーザーの作成、取得、更新、削除
    - ユーザー認証とトークン管理
    - ユーザープロフィール管理
  version: 1.0.0                             # APIバージョン（必須）
  termsOfService: https://example.com/terms  # 利用規約URL
  contact:                                   # 連絡先情報
    name: API Support Team
    email: api-support@example.com
    url: https://support.example.com
  license:                                   # ライセンス情報
    name: MIT
    url: https://opensource.org/licenses/MIT
```

**ポイント**:
- `description`フィールドはMarkdown形式をサポート
- `version`はAPIのバージョンであり、OpenAPIのバージョン（`openapi`フィールド）とは別
- 連絡先情報は、APIユーザーがサポートを受けるために重要

### serversセクション: APIサーバーの定義

`servers`セクションには、APIが利用可能なサーバーのURLを記述します：

```yaml
servers:
  - url: https://api.example.com/v1
    description: 本番環境
  - url: https://staging-api.example.com/v1
    description: ステージング環境
  - url: http://localhost:3000/v1
    description: ローカル開発環境
```

変数を使った動的なURL定義も可能です：

```yaml
servers:
  - url: https://{environment}.example.com/v1
    description: 環境ごとのAPI
    variables:
      environment:
        default: api
        description: API環境
        enum:
          - api          # 本番環境
          - staging-api  # ステージング環境
          - dev-api      # 開発環境
```

### pathsセクション: エンドポイントの定義

`paths`セクションは、OpenAPI仕様書の中核部分で、すべてのAPIエンドポイントを定義します。

#### 基本的なエンドポイント定義

```yaml
paths:
  /users:
    get:
      summary: ユーザー一覧を取得
      description: 登録されているすべてのユーザーを取得します
      operationId: listUsers           # 操作の一意なID（コード生成で使用）
      tags:
        - users                        # エンドポイントのグループ化
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/User'
```

#### パスパラメータの定義

```yaml
paths:
  /users/{userId}:
    get:
      summary: ユーザー詳細を取得
      description: 指定されたIDのユーザー詳細情報を取得します
      parameters:
        - name: userId
          in: path                     # パラメータの場所
          required: true               # 必須かどうか
          description: ユーザーID
          schema:
            type: string
            example: "user_123"
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          description: ユーザーが見つかりません
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
```

#### クエリパラメータの定義

```yaml
paths:
  /users:
    get:
      summary: ユーザー一覧を取得
      description: ページネーション、フィルタリング、ソートをサポート
      parameters:
        - name: page
          in: query                    # クエリパラメータ
          description: ページ番号（1から始まる）
          required: false
          schema:
            type: integer
            minimum: 1
            default: 1
            example: 1
        - name: limit
          in: query
          description: 1ページあたりの件数
          required: false
          schema:
            type: integer
            minimum: 1
            maximum: 100
            default: 20
            example: 20
        - name: sort
          in: query
          description: ソート順
          required: false
          schema:
            type: string
            enum:
              - name_asc
              - name_desc
              - created_at_asc
              - created_at_desc
            default: created_at_desc
            example: name_asc
        - name: role
          in: query
          description: ロールでフィルタリング
          required: false
          schema:
            type: string
            enum:
              - admin
              - user
              - guest
            example: user
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/User'
                  pagination:
                    $ref: '#/components/schemas/Pagination'
```

#### リクエストボディの定義

```yaml
paths:
  /users:
    post:
      summary: ユーザーを作成
      description: 新しいユーザーを作成します
      requestBody:
        description: ユーザー情報
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UserCreateInput'
            examples:
              basic:
                summary: 基本的な例
                value:
                  name: "山田太郎"
                  email: "yamada@example.com"
                  password: "SecurePass123!"
              admin:
                summary: 管理者ユーザーの例
                value:
                  name: "管理者"
                  email: "admin@example.com"
                  password: "AdminPass123!"
                  role: "admin"
      responses:
        '201':
          description: 作成成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '400':
          description: バリデーションエラー
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ValidationError'
        '409':
          description: メールアドレスが既に使用されています
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
```

### componentsセクション: 再利用可能なコンポーネント

`components`セクションには、仕様書全体で再利用可能なコンポーネントを定義します。これにより、重複を避け、保守性を高めることができます。

#### スキーマの定義

```yaml
components:
  schemas:
    User:
      type: object
      description: ユーザー情報
      required:
        - id
        - name
        - email
        - createdAt
      properties:
        id:
          type: string
          description: ユーザーID
          example: "user_123"
        name:
          type: string
          description: ユーザー名
          minLength: 1
          maxLength: 50
          example: "山田太郎"
        email:
          type: string
          format: email
          description: メールアドレス
          example: "yamada@example.com"
        role:
          type: string
          description: ユーザーロール
          enum:
            - admin
            - user
            - guest
          default: user
          example: "user"
        profile:
          $ref: '#/components/schemas/UserProfile'
        createdAt:
          type: string
          format: date-time
          description: 作成日時
          example: "2026-01-28T12:00:00Z"
        updatedAt:
          type: string
          format: date-time
          description: 更新日時
          example: "2026-01-28T12:00:00Z"

    UserProfile:
      type: object
      description: ユーザープロフィール
      properties:
        bio:
          type: string
          description: 自己紹介
          maxLength: 500
          example: "ソフトウェアエンジニアです"
        avatarUrl:
          type: string
          format: uri
          description: アバター画像URL
          example: "https://example.com/avatars/user_123.jpg"
        website:
          type: string
          format: uri
          description: WebサイトURL
          example: "https://example.com"

    UserCreateInput:
      type: object
      description: ユーザー作成時の入力データ
      required:
        - name
        - email
        - password
      properties:
        name:
          type: string
          minLength: 1
          maxLength: 50
          example: "山田太郎"
        email:
          type: string
          format: email
          example: "yamada@example.com"
        password:
          type: string
          format: password
          minLength: 8
          maxLength: 100
          description: パスワード（8文字以上、大小英数字と記号を含む）
          example: "SecurePass123!"
        role:
          type: string
          enum:
            - admin
            - user
            - guest
          default: user
          example: "user"

    Pagination:
      type: object
      description: ページネーション情報
      required:
        - page
        - limit
        - total
        - totalPages
      properties:
        page:
          type: integer
          description: 現在のページ番号
          minimum: 1
          example: 1
        limit:
          type: integer
          description: 1ページあたりの件数
          minimum: 1
          maximum: 100
          example: 20
        total:
          type: integer
          description: 総件数
          minimum: 0
          example: 100
        totalPages:
          type: integer
          description: 総ページ数
          minimum: 0
          example: 5

    Error:
      type: object
      description: エラーレスポンス
      required:
        - code
        - message
      properties:
        code:
          type: string
          description: エラーコード
          example: "USER_NOT_FOUND"
        message:
          type: string
          description: エラーメッセージ
          example: "指定されたユーザーが見つかりません"
        details:
          type: object
          description: エラーの詳細情報
          additionalProperties: true

    ValidationError:
      type: object
      description: バリデーションエラーレスポンス
      required:
        - code
        - message
        - errors
      properties:
        code:
          type: string
          description: エラーコード
          example: "VALIDATION_ERROR"
        message:
          type: string
          description: エラーメッセージ
          example: "入力データが不正です"
        errors:
          type: array
          description: フィールドごとのエラー
          items:
            type: object
            required:
              - field
              - message
            properties:
              field:
                type: string
                description: エラーが発生したフィールド名
                example: "email"
              message:
                type: string
                description: フィールド固有のエラーメッセージ
                example: "メールアドレスの形式が不正です"
```

**ポイント**:
- `required`配列で必須フィールドを指定
- `example`で具体的な値の例を提供
- `$ref`で他のスキーマを参照し、重複を避ける
- バリデーションルール（`minLength`, `maxLength`, `minimum`, `maximum`など）を明示

#### レスポンスの定義

共通のレスポンスも`components`で定義できます：

```yaml
components:
  responses:
    NotFound:
      description: リソースが見つかりません
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: "NOT_FOUND"
            message: "指定されたリソースが見つかりません"

    Unauthorized:
      description: 認証が必要です
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: "UNAUTHORIZED"
            message: "認証情報が提供されていないか、無効です"

    Forbidden:
      description: アクセス権限がありません
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: "FORBIDDEN"
            message: "このリソースにアクセスする権限がありません"

    ValidationError:
      description: バリデーションエラー
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ValidationError'
```

これらを使用すると、エンドポイント定義がシンプルになります：

```yaml
paths:
  /users/{userId}:
    get:
      summary: ユーザー詳細を取得
      parameters:
        - name: userId
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '404':
          $ref: '#/components/responses/NotFound'
```

#### パラメータの定義

共通のパラメータも定義できます：

```yaml
components:
  parameters:
    PageParam:
      name: page
      in: query
      description: ページ番号（1から始まる）
      required: false
      schema:
        type: integer
        minimum: 1
        default: 1
        example: 1

    LimitParam:
      name: limit
      in: query
      description: 1ページあたりの件数
      required: false
      schema:
        type: integer
        minimum: 1
        maximum: 100
        default: 20
        example: 20

    UserIdParam:
      name: userId
      in: path
      description: ユーザーID
      required: true
      schema:
        type: string
        example: "user_123"
```

使用例：

```yaml
paths:
  /users:
    get:
      summary: ユーザー一覧を取得
      parameters:
        - $ref: '#/components/parameters/PageParam'
        - $ref: '#/components/parameters/LimitParam'
      responses:
        '200':
          description: 成功
```

---

## セキュリティの定義

APIの認証・認可方式をOpenAPIで定義できます。

### Bearer Token認証

```yaml
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: |
        JWT Bearer トークンによる認証。
        ログインAPIで取得したトークンをAuthorizationヘッダーに含めてください。

        例: `Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...`

# API全体にセキュリティを適用
security:
  - bearerAuth: []

paths:
  /users:
    get:
      summary: ユーザー一覧を取得
      # このエンドポイントはbearerAuth認証が必要
      responses:
        '200':
          description: 成功
        '401':
          $ref: '#/components/responses/Unauthorized'

  /public/info:
    get:
      summary: 公開情報を取得
      security: []  # このエンドポイントは認証不要
      responses:
        '200':
          description: 成功
```

### API Key認証

```yaml
components:
  securitySchemes:
    apiKey:
      type: apiKey
      in: header
      name: X-API-Key
      description: |
        API Keyによる認証。
        管理画面で発行されたAPI Keyをヘッダーに含めてください。

        例: `X-API-Key: your-api-key-here`

security:
  - apiKey: []
```

### OAuth 2.0認証

```yaml
components:
  securitySchemes:
    oauth2:
      type: oauth2
      description: OAuth 2.0認証
      flows:
        authorizationCode:
          authorizationUrl: https://auth.example.com/oauth/authorize
          tokenUrl: https://auth.example.com/oauth/token
          refreshUrl: https://auth.example.com/oauth/refresh
          scopes:
            read:users: ユーザー情報の読み取り
            write:users: ユーザー情報の書き込み
            admin: 管理者権限

security:
  - oauth2:
      - read:users

paths:
  /users:
    get:
      summary: ユーザー一覧を取得
      security:
        - oauth2:
            - read:users
      responses:
        '200':
          description: 成功

    post:
      summary: ユーザーを作成
      security:
        - oauth2:
            - write:users
      responses:
        '201':
          description: 作成成功
```

### 複数の認証方式のサポート

```yaml
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
    apiKey:
      type: apiKey
      in: header
      name: X-API-Key

# どちらかの認証方式を使用可能
security:
  - bearerAuth: []
  - apiKey: []
```

---

## 実践的なOpenAPI仕様書の例

ここでは、実際のプロジェクトで使用できる完全なOpenAPI仕様書の例を示します。

### ユーザー管理APIの完全な仕様書

```yaml
openapi: 3.0.3
info:
  title: User Management API
  description: |
    ユーザー管理のためのRESTful API

    ## 認証
    このAPIは、JWT Bearer トークンによる認証を使用します。
    ログインエンドポイント (`POST /auth/login`) でトークンを取得してください。

    ## レート制限
    - 認証済みユーザー: 1000リクエスト/時間
    - 未認証ユーザー: 100リクエスト/時間

    ## エラーハンドリング
    すべてのエラーレスポンスは、統一されたフォーマットで返されます。
    詳細は `Error` スキーマを参照してください。
  version: 1.0.0
  contact:
    name: API Support Team
    email: api-support@example.com
    url: https://support.example.com
  license:
    name: MIT
    url: https://opensource.org/licenses/MIT

servers:
  - url: https://api.example.com/v1
    description: 本番環境
  - url: https://staging-api.example.com/v1
    description: ステージング環境
  - url: http://localhost:3000/v1
    description: ローカル開発環境

tags:
  - name: auth
    description: 認証関連のエンドポイント
  - name: users
    description: ユーザー管理
  - name: admin
    description: 管理者専用機能

paths:
  /auth/login:
    post:
      summary: ログイン
      description: メールアドレスとパスワードでログインし、JWTトークンを取得します
      operationId: login
      tags:
        - auth
      security: []  # 認証不要
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - email
                - password
              properties:
                email:
                  type: string
                  format: email
                  example: "yamada@example.com"
                password:
                  type: string
                  format: password
                  example: "SecurePass123!"
      responses:
        '200':
          description: ログイン成功
          content:
            application/json:
              schema:
                type: object
                required:
                  - token
                  - expiresIn
                  - user
                properties:
                  token:
                    type: string
                    description: JWTトークン
                    example: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
                  expiresIn:
                    type: integer
                    description: トークンの有効期限（秒）
                    example: 3600
                  user:
                    $ref: '#/components/schemas/User'
        '401':
          description: 認証失敗
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
              example:
                code: "INVALID_CREDENTIALS"
                message: "メールアドレスまたはパスワードが正しくありません"

  /auth/refresh:
    post:
      summary: トークンリフレッシュ
      description: 現在のトークンを使用して新しいトークンを取得します
      operationId: refreshToken
      tags:
        - auth
      security:
        - bearerAuth: []
      responses:
        '200':
          description: リフレッシュ成功
          content:
            application/json:
              schema:
                type: object
                required:
                  - token
                  - expiresIn
                properties:
                  token:
                    type: string
                    description: 新しいJWTトークン
                  expiresIn:
                    type: integer
                    description: トークンの有効期限（秒）
        '401':
          $ref: '#/components/responses/Unauthorized'

  /users:
    get:
      summary: ユーザー一覧を取得
      description: |
        登録されているユーザーの一覧を取得します。
        ページネーション、フィルタリング、ソートをサポートしています。
      operationId: listUsers
      tags:
        - users
      security:
        - bearerAuth: []
      parameters:
        - $ref: '#/components/parameters/PageParam'
        - $ref: '#/components/parameters/LimitParam'
        - name: sort
          in: query
          description: ソート順
          required: false
          schema:
            type: string
            enum:
              - name_asc
              - name_desc
              - created_at_asc
              - created_at_desc
            default: created_at_desc
        - name: role
          in: query
          description: ロールでフィルタリング
          required: false
          schema:
            type: string
            enum:
              - admin
              - user
              - guest
        - name: search
          in: query
          description: 名前またはメールアドレスで検索
          required: false
          schema:
            type: string
            minLength: 1
            maxLength: 100
            example: "yamada"
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                type: object
                required:
                  - data
                  - pagination
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/User'
                  pagination:
                    $ref: '#/components/schemas/Pagination'
        '401':
          $ref: '#/components/responses/Unauthorized'

    post:
      summary: ユーザーを作成
      description: 新しいユーザーを作成します
      operationId: createUser
      tags:
        - users
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UserCreateInput'
            examples:
              basic:
                summary: 基本的なユーザー
                value:
                  name: "山田太郎"
                  email: "yamada@example.com"
                  password: "SecurePass123!"
              admin:
                summary: 管理者ユーザー
                value:
                  name: "管理者"
                  email: "admin@example.com"
                  password: "AdminPass123!"
                  role: "admin"
      responses:
        '201':
          description: 作成成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '400':
          $ref: '#/components/responses/ValidationError'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '409':
          description: メールアドレスが既に使用されています
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
              example:
                code: "EMAIL_ALREADY_EXISTS"
                message: "このメールアドレスは既に使用されています"

  /users/{userId}:
    get:
      summary: ユーザー詳細を取得
      description: 指定されたIDのユーザー詳細情報を取得します
      operationId: getUser
      tags:
        - users
      security:
        - bearerAuth: []
      parameters:
        - $ref: '#/components/parameters/UserIdParam'
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '404':
          $ref: '#/components/responses/NotFound'

    put:
      summary: ユーザーを更新
      description: 指定されたIDのユーザー情報を更新します
      operationId: updateUser
      tags:
        - users
      security:
        - bearerAuth: []
      parameters:
        - $ref: '#/components/parameters/UserIdParam'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UserUpdateInput'
      responses:
        '200':
          description: 更新成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '400':
          $ref: '#/components/responses/ValidationError'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '404':
          $ref: '#/components/responses/NotFound'

    delete:
      summary: ユーザーを削除
      description: 指定されたIDのユーザーを削除します
      operationId: deleteUser
      tags:
        - users
      security:
        - bearerAuth: []
      parameters:
        - $ref: '#/components/parameters/UserIdParam'
      responses:
        '204':
          description: 削除成功
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '404':
          $ref: '#/components/responses/NotFound'

  /users/{userId}/profile:
    put:
      summary: ユーザープロフィールを更新
      description: 指定されたIDのユーザープロフィール情報を更新します
      operationId: updateUserProfile
      tags:
        - users
      security:
        - bearerAuth: []
      parameters:
        - $ref: '#/components/parameters/UserIdParam'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UserProfile'
      responses:
        '200':
          description: 更新成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '400':
          $ref: '#/components/responses/ValidationError'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '404':
          $ref: '#/components/responses/NotFound'

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: |
        JWT Bearer トークンによる認証。
        ログインAPIで取得したトークンをAuthorizationヘッダーに含めてください。

        例: `Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...`

  parameters:
    PageParam:
      name: page
      in: query
      description: ページ番号（1から始まる）
      required: false
      schema:
        type: integer
        minimum: 1
        default: 1
        example: 1

    LimitParam:
      name: limit
      in: query
      description: 1ページあたりの件数
      required: false
      schema:
        type: integer
        minimum: 1
        maximum: 100
        default: 20
        example: 20

    UserIdParam:
      name: userId
      in: path
      description: ユーザーID
      required: true
      schema:
        type: string
        pattern: '^user_[a-zA-Z0-9]+$'
        example: "user_123"

  schemas:
    User:
      type: object
      description: ユーザー情報
      required:
        - id
        - name
        - email
        - role
        - createdAt
        - updatedAt
      properties:
        id:
          type: string
          description: ユーザーID
          example: "user_123"
        name:
          type: string
          description: ユーザー名
          minLength: 1
          maxLength: 50
          example: "山田太郎"
        email:
          type: string
          format: email
          description: メールアドレス
          example: "yamada@example.com"
        role:
          type: string
          description: ユーザーロール
          enum:
            - admin
            - user
            - guest
          example: "user"
        profile:
          $ref: '#/components/schemas/UserProfile'
        createdAt:
          type: string
          format: date-time
          description: 作成日時
          example: "2026-01-28T12:00:00Z"
        updatedAt:
          type: string
          format: date-time
          description: 更新日時
          example: "2026-01-28T12:00:00Z"

    UserProfile:
      type: object
      description: ユーザープロフィール
      properties:
        bio:
          type: string
          description: 自己紹介
          maxLength: 500
          example: "ソフトウェアエンジニアです"
        avatarUrl:
          type: string
          format: uri
          description: アバター画像URL
          example: "https://example.com/avatars/user_123.jpg"
        website:
          type: string
          format: uri
          description: WebサイトURL
          example: "https://example.com"
        location:
          type: string
          description: 所在地
          maxLength: 100
          example: "東京都"
        twitter:
          type: string
          description: Twitterハンドル
          pattern: '^@[a-zA-Z0-9_]{1,15}$'
          example: "@yamada"

    UserCreateInput:
      type: object
      description: ユーザー作成時の入力データ
      required:
        - name
        - email
        - password
      properties:
        name:
          type: string
          minLength: 1
          maxLength: 50
          example: "山田太郎"
        email:
          type: string
          format: email
          example: "yamada@example.com"
        password:
          type: string
          format: password
          minLength: 8
          maxLength: 100
          description: |
            パスワード要件:
            - 8文字以上100文字以下
            - 大文字と小文字を含む
            - 数字を含む
            - 記号を含む
          example: "SecurePass123!"
        role:
          type: string
          enum:
            - admin
            - user
            - guest
          default: user
          example: "user"

    UserUpdateInput:
      type: object
      description: ユーザー更新時の入力データ
      properties:
        name:
          type: string
          minLength: 1
          maxLength: 50
          example: "山田太郎"
        email:
          type: string
          format: email
          example: "yamada@example.com"
        password:
          type: string
          format: password
          minLength: 8
          maxLength: 100
          description: 新しいパスワード（変更する場合のみ）
          example: "NewSecurePass123!"
        role:
          type: string
          enum:
            - admin
            - user
            - guest
          example: "user"

    Pagination:
      type: object
      description: ページネーション情報
      required:
        - page
        - limit
        - total
        - totalPages
      properties:
        page:
          type: integer
          description: 現在のページ番号
          minimum: 1
          example: 1
        limit:
          type: integer
          description: 1ページあたりの件数
          minimum: 1
          maximum: 100
          example: 20
        total:
          type: integer
          description: 総件数
          minimum: 0
          example: 100
        totalPages:
          type: integer
          description: 総ページ数
          minimum: 0
          example: 5

    Error:
      type: object
      description: エラーレスポンス
      required:
        - code
        - message
      properties:
        code:
          type: string
          description: エラーコード
          example: "USER_NOT_FOUND"
        message:
          type: string
          description: エラーメッセージ
          example: "指定されたユーザーが見つかりません"
        details:
          type: object
          description: エラーの詳細情報
          additionalProperties: true

    ValidationError:
      type: object
      description: バリデーションエラーレスポンス
      required:
        - code
        - message
        - errors
      properties:
        code:
          type: string
          description: エラーコード
          example: "VALIDATION_ERROR"
        message:
          type: string
          description: エラーメッセージ
          example: "入力データが不正です"
        errors:
          type: array
          description: フィールドごとのエラー
          items:
            type: object
            required:
              - field
              - message
            properties:
              field:
                type: string
                description: エラーが発生したフィールド名
                example: "email"
              message:
                type: string
                description: フィールド固有のエラーメッセージ
                example: "メールアドレスの形式が不正です"
              value:
                description: 入力された値
                example: "invalid-email"

  responses:
    NotFound:
      description: リソースが見つかりません
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: "NOT_FOUND"
            message: "指定されたリソースが見つかりません"

    Unauthorized:
      description: 認証が必要です
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: "UNAUTHORIZED"
            message: "認証情報が提供されていないか、無効です"

    Forbidden:
      description: アクセス権限がありません
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: "FORBIDDEN"
            message: "このリソースにアクセスする権限がありません"

    ValidationError:
      description: バリデーションエラー
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ValidationError'
          example:
            code: "VALIDATION_ERROR"
            message: "入力データが不正です"
            errors:
              - field: "email"
                message: "メールアドレスの形式が不正です"
                value: "invalid-email"
              - field: "password"
                message: "パスワードは8文字以上である必要があります"
                value: "short"

security:
  - bearerAuth: []
```

この仕様書は、以下の要素を含んでいます：

- 認証エンドポイント（ログイン、トークンリフレッシュ）
- CRUD操作（作成、読み取り、更新、削除）
- ページネーション、フィルタリング、ソート
- 詳細なエラーハンドリング
- 再利用可能なコンポーネント
- JWT認証
- 実用的な例とドキュメント

---

## Swagger UIによる可視化

OpenAPI仕様書を作成したら、Swagger UIを使って対話的なドキュメントを生成できます。

### Swagger UIとは

Swagger UIは、OpenAPI仕様書から自動的にWebベースのドキュメントを生成するツールです。以下の機能を提供します：

- エンドポイント一覧の表示
- リクエスト/レスポンスの詳細表示
- ブラウザから直接APIを試せる"Try it out"機能
- 認証のサポート
- スキーマの可視化

**公式サイト**: https://swagger.io/tools/swagger-ui/

### Swagger UIのセットアップ

#### 方法1: HTMLファイルで使用

最もシンプルな方法は、HTMLファイルにSwagger UIを埋め込むことです：

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>API Documentation</title>
  <link rel="stylesheet" href="https://unpkg.com/swagger-ui-dist@5/swagger-ui.css" />
</head>
<body>
  <div id="swagger-ui"></div>

  <script src="https://unpkg.com/swagger-ui-dist@5/swagger-ui-bundle.js"></script>
  <script src="https://unpkg.com/swagger-ui-dist@5/swagger-ui-standalone-preset.js"></script>
  <script>
    window.onload = function() {
      SwaggerUIBundle({
        url: "./openapi.yaml",  // OpenAPI仕様書のパス
        dom_id: '#swagger-ui',
        deepLinking: true,
        presets: [
          SwaggerUIBundle.presets.apis,
          SwaggerUIStandalonePreset
        ],
        plugins: [
          SwaggerUIBundle.plugins.DownloadUrl
        ],
        layout: "StandaloneLayout"
      });
    };
  </script>
</body>
</html>
```

このHTMLファイルを、OpenAPI仕様書（`openapi.yaml`）と同じディレクトリに配置し、Webサーバーで公開します。

#### 方法2: Node.jsで使用

Express.jsアプリケーションにSwagger UIを組み込む例：

```bash
npm install swagger-ui-express
```

```javascript
// server.js
const express = require('express');
const swaggerUi = require('swagger-ui-express');
const YAML = require('yamljs');

const app = express();

// OpenAPI仕様書を読み込み
const swaggerDocument = YAML.load('./openapi.yaml');

// Swagger UIをセットアップ
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerDocument, {
  customCss: '.swagger-ui .topbar { display: none }',  // トップバーを非表示
  customSiteTitle: "User API Documentation",
  swaggerOptions: {
    persistAuthorization: true,  // 認証情報を保持
  }
}));

// APIルート
app.get('/api/v1/users', (req, res) => {
  // ユーザー一覧を返す
  res.json({ data: [], pagination: {} });
});

app.listen(3000, () => {
  console.log('Server is running on http://localhost:3000');
  console.log('API Docs: http://localhost:3000/api-docs');
});
```

#### 方法3: Dockerで使用

Dockerコンテナとして実行する方法：

```bash
docker run -p 80:8080 -e SWAGGER_JSON=/foo/openapi.yaml -v /path/to/your/openapi.yaml:/foo/openapi.yaml swaggerapi/swagger-ui
```

または、`docker-compose.yml`を使用：

```yaml
version: '3.8'
services:
  swagger-ui:
    image: swaggerapi/swagger-ui
    ports:
      - "8080:8080"
    environment:
      SWAGGER_JSON: /openapi/openapi.yaml
    volumes:
      - ./openapi.yaml:/openapi/openapi.yaml
```

### Swagger UIの使い方

#### エンドポイントの確認

Swagger UIを開くと、すべてのエンドポイントがタグごとにグループ化されて表示されます。各エンドポイントをクリックすると、詳細情報が展開されます。

#### APIの実行（Try it out）

1. エンドポイントを展開
2. "Try it out"ボタンをクリック
3. パラメータやリクエストボディを入力
4. "Execute"ボタンをクリック
5. レスポンスが表示される

#### 認証の設定

Bearer Token認証を使用する場合：

1. 右上の"Authorize"ボタンをクリック
2. トークンを入力（`Bearer `プレフィックスは不要）
3. "Authorize"をクリック
4. 以降のリクエストに自動的にトークンが含まれる

### Swagger UIのカスタマイズ

#### カスタムCSSの適用

```javascript
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerDocument, {
  customCss: `
    .swagger-ui .topbar {
      display: none;
    }
    .swagger-ui .info .title {
      color: #3b82f6;
      font-size: 2rem;
    }
  `,
  customSiteTitle: "User API Documentation",
}));
```

#### カスタムロゴの追加

```javascript
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerDocument, {
  customCssUrl: '/custom-swagger-ui.css',
  customfavIcon: '/favicon.ico',
  customSiteTitle: "User API Documentation",
}));
```

---

## コードからの仕様書自動生成

手動でOpenAPI仕様書を書くのは手間がかかります。多くのフレームワークやライブラリは、コードから自動的にOpenAPI仕様書を生成する機能を提供しています。

### TypeScript + Express の例

`tsoa`ライブラリを使用すると、TypeScriptのデコレータからOpenAPI仕様書を生成できます。

#### インストール

```bash
npm install tsoa express @types/express
npm install -D typescript @types/node
```

#### コントローラーの定義

```typescript
// src/controllers/userController.ts
import { Body, Controller, Get, Path, Post, Query, Route, Tags, Response, Security } from 'tsoa';

interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user' | 'guest';
  createdAt: string;
  updatedAt: string;
}

interface UserCreateInput {
  name: string;
  email: string;
  password: string;
  role?: 'admin' | 'user' | 'guest';
}

interface PaginatedResponse<T> {
  data: T[];
  pagination: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
  };
}

interface ErrorResponse {
  code: string;
  message: string;
}

@Route('users')
@Tags('Users')
export class UserController extends Controller {
  /**
   * ユーザー一覧を取得
   * @summary ユーザー一覧を取得
   * @param page ページ番号（1から始まる）
   * @param limit 1ページあたりの件数
   * @param sort ソート順
   * @param role ロールでフィルタリング
   */
  @Get()
  @Security('bearerAuth')
  @Response<ErrorResponse>(401, 'Unauthorized')
  public async listUsers(
    @Query() page: number = 1,
    @Query() limit: number = 20,
    @Query() sort?: 'name_asc' | 'name_desc' | 'created_at_asc' | 'created_at_desc',
    @Query() role?: 'admin' | 'user' | 'guest'
  ): Promise<PaginatedResponse<User>> {
    // 実装
    return {
      data: [],
      pagination: {
        page,
        limit,
        total: 0,
        totalPages: 0,
      },
    };
  }

  /**
   * ユーザーを作成
   * @summary ユーザーを作成
   */
  @Post()
  @Security('bearerAuth')
  @Response<ErrorResponse>(400, 'Validation Error')
  @Response<ErrorResponse>(401, 'Unauthorized')
  @Response<ErrorResponse>(409, 'Email Already Exists')
  public async createUser(
    @Body() body: UserCreateInput
  ): Promise<User> {
    // 実装
    return {
      id: 'user_123',
      name: body.name,
      email: body.email,
      role: body.role || 'user',
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
    };
  }

  /**
   * ユーザー詳細を取得
   * @summary ユーザー詳細を取得
   * @param userId ユーザーID
   */
  @Get('{userId}')
  @Security('bearerAuth')
  @Response<ErrorResponse>(401, 'Unauthorized')
  @Response<ErrorResponse>(404, 'Not Found')
  public async getUser(
    @Path() userId: string
  ): Promise<User> {
    // 実装
    return {
      id: userId,
      name: '山田太郎',
      email: 'yamada@example.com',
      role: 'user',
      createdAt: '2026-01-28T12:00:00Z',
      updatedAt: '2026-01-28T12:00:00Z',
    };
  }
}
```

#### tsoaの設定

```json
// tsoa.json
{
  "entryFile": "src/app.ts",
  "noImplicitAdditionalProperties": "throw-on-extras",
  "spec": {
    "outputDirectory": "public",
    "specVersion": 3,
    "name": "User Management API",
    "description": "ユーザー管理のためのRESTful API",
    "version": "1.0.0",
    "securityDefinitions": {
      "bearerAuth": {
        "type": "http",
        "scheme": "bearer",
        "bearerFormat": "JWT"
      }
    }
  },
  "routes": {
    "routesDir": "src"
  }
}
```

#### 仕様書の生成

```bash
npx tsoa spec
```

これにより、`public/swagger.json`にOpenAPI仕様書が生成されます。

### NestJS の例

NestJSには、OpenAPI仕様書を生成する機能が組み込まれています。

#### インストール

```bash
npm install @nestjs/swagger
```

#### Swagger設定

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Swagger設定
  const config = new DocumentBuilder()
    .setTitle('User Management API')
    .setDescription('ユーザー管理のためのRESTful API')
    .setVersion('1.0.0')
    .addBearerAuth()
    .addTag('users', 'ユーザー管理')
    .addTag('auth', '認証')
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api-docs', app, document);

  await app.listen(3000);
  console.log('API Docs: http://localhost:3000/api-docs');
}
bootstrap();
```

#### コントローラーの定義

```typescript
// src/users/users.controller.ts
import { Controller, Get, Post, Put, Delete, Body, Param, Query } from '@nestjs/common';
import { ApiTags, ApiOperation, ApiResponse, ApiBearerAuth, ApiQuery } from '@nestjs/swagger';
import { UserDto } from './dto/user.dto';
import { CreateUserDto } from './dto/create-user.dto';

@ApiTags('users')
@ApiBearerAuth()
@Controller('users')
export class UsersController {
  @Get()
  @ApiOperation({ summary: 'ユーザー一覧を取得' })
  @ApiResponse({ status: 200, description: '成功', type: [UserDto] })
  @ApiResponse({ status: 401, description: '認証が必要です' })
  @ApiQuery({ name: 'page', required: false, type: Number, description: 'ページ番号' })
  @ApiQuery({ name: 'limit', required: false, type: Number, description: '1ページあたりの件数' })
  async findAll(
    @Query('page') page: number = 1,
    @Query('limit') limit: number = 20,
  ) {
    // 実装
    return { data: [], pagination: {} };
  }

  @Post()
  @ApiOperation({ summary: 'ユーザーを作成' })
  @ApiResponse({ status: 201, description: '作成成功', type: UserDto })
  @ApiResponse({ status: 400, description: 'バリデーションエラー' })
  @ApiResponse({ status: 401, description: '認証が必要です' })
  async create(@Body() createUserDto: CreateUserDto) {
    // 実装
    return {};
  }

  @Get(':id')
  @ApiOperation({ summary: 'ユーザー詳細を取得' })
  @ApiResponse({ status: 200, description: '成功', type: UserDto })
  @ApiResponse({ status: 401, description: '認証が必要です' })
  @ApiResponse({ status: 404, description: 'ユーザーが見つかりません' })
  async findOne(@Param('id') id: string) {
    // 実装
    return {};
  }
}
```

#### DTOの定義

```typescript
// src/users/dto/user.dto.ts
import { ApiProperty } from '@nestjs/swagger';

export class UserDto {
  @ApiProperty({
    description: 'ユーザーID',
    example: 'user_123',
  })
  id: string;

  @ApiProperty({
    description: 'ユーザー名',
    example: '山田太郎',
    minLength: 1,
    maxLength: 50,
  })
  name: string;

  @ApiProperty({
    description: 'メールアドレス',
    example: 'yamada@example.com',
    format: 'email',
  })
  email: string;

  @ApiProperty({
    description: 'ユーザーロール',
    example: 'user',
    enum: ['admin', 'user', 'guest'],
  })
  role: string;

  @ApiProperty({
    description: '作成日時',
    example: '2026-01-28T12:00:00Z',
  })
  createdAt: string;

  @ApiProperty({
    description: '更新日時',
    example: '2026-01-28T12:00:00Z',
  })
  updatedAt: string;
}
```

```typescript
// src/users/dto/create-user.dto.ts
import { ApiProperty } from '@nestjs/swagger';
import { IsEmail, IsString, MinLength, MaxLength, IsOptional, IsEnum } from 'class-validator';

export class CreateUserDto {
  @ApiProperty({
    description: 'ユーザー名',
    example: '山田太郎',
    minLength: 1,
    maxLength: 50,
  })
  @IsString()
  @MinLength(1)
  @MaxLength(50)
  name: string;

  @ApiProperty({
    description: 'メールアドレス',
    example: 'yamada@example.com',
  })
  @IsEmail()
  email: string;

  @ApiProperty({
    description: 'パスワード',
    example: 'SecurePass123!',
    minLength: 8,
    maxLength: 100,
  })
  @IsString()
  @MinLength(8)
  @MaxLength(100)
  password: string;

  @ApiProperty({
    description: 'ユーザーロール',
    example: 'user',
    enum: ['admin', 'user', 'guest'],
    required: false,
    default: 'user',
  })
  @IsOptional()
  @IsEnum(['admin', 'user', 'guest'])
  role?: string;
}
```

NestJSでは、アプリケーションを起動すると自動的にSwagger UIが`/api-docs`で利用可能になります。

### FastAPI (Python) の例

FastAPIは、型ヒントから自動的にOpenAPI仕様書を生成します。

```python
# main.py
from fastapi import FastAPI, HTTPException, Query, Path, Body
from pydantic import BaseModel, EmailStr, Field
from typing import Optional, List
from datetime import datetime

app = FastAPI(
    title="User Management API",
    description="ユーザー管理のためのRESTful API",
    version="1.0.0",
)

# モデル定義
class UserProfile(BaseModel):
    bio: Optional[str] = Field(None, max_length=500, description="自己紹介")
    avatar_url: Optional[str] = Field(None, description="アバター画像URL")
    website: Optional[str] = Field(None, description="WebサイトURL")

class User(BaseModel):
    id: str = Field(..., description="ユーザーID", example="user_123")
    name: str = Field(..., min_length=1, max_length=50, description="ユーザー名", example="山田太郎")
    email: EmailStr = Field(..., description="メールアドレス", example="yamada@example.com")
    role: str = Field(..., description="ユーザーロール", example="user")
    profile: Optional[UserProfile] = None
    created_at: datetime = Field(..., description="作成日時")
    updated_at: datetime = Field(..., description="更新日時")

class UserCreateInput(BaseModel):
    name: str = Field(..., min_length=1, max_length=50, example="山田太郎")
    email: EmailStr = Field(..., example="yamada@example.com")
    password: str = Field(..., min_length=8, max_length=100, example="SecurePass123!")
    role: Optional[str] = Field("user", description="ユーザーロール")

class Pagination(BaseModel):
    page: int = Field(..., ge=1, description="現在のページ番号")
    limit: int = Field(..., ge=1, le=100, description="1ページあたりの件数")
    total: int = Field(..., ge=0, description="総件数")
    total_pages: int = Field(..., ge=0, description="総ページ数")

class PaginatedUsers(BaseModel):
    data: List[User]
    pagination: Pagination

# エンドポイント定義
@app.get(
    "/users",
    response_model=PaginatedUsers,
    tags=["users"],
    summary="ユーザー一覧を取得",
    description="登録されているユーザーの一覧を取得します"
)
async def list_users(
    page: int = Query(1, ge=1, description="ページ番号（1から始まる）"),
    limit: int = Query(20, ge=1, le=100, description="1ページあたりの件数"),
    sort: Optional[str] = Query(None, description="ソート順"),
    role: Optional[str] = Query(None, description="ロールでフィルタリング"),
):
    # 実装
    return {
        "data": [],
        "pagination": {
            "page": page,
            "limit": limit,
            "total": 0,
            "total_pages": 0
        }
    }

@app.post(
    "/users",
    response_model=User,
    status_code=201,
    tags=["users"],
    summary="ユーザーを作成",
    description="新しいユーザーを作成します"
)
async def create_user(user: UserCreateInput = Body(...)):
    # 実装
    return {
        "id": "user_123",
        "name": user.name,
        "email": user.email,
        "role": user.role or "user",
        "created_at": datetime.now(),
        "updated_at": datetime.now(),
    }

@app.get(
    "/users/{user_id}",
    response_model=User,
    tags=["users"],
    summary="ユーザー詳細を取得",
    description="指定されたIDのユーザー詳細情報を取得します"
)
async def get_user(
    user_id: str = Path(..., description="ユーザーID", example="user_123")
):
    # 実装
    return {
        "id": user_id,
        "name": "山田太郎",
        "email": "yamada@example.com",
        "role": "user",
        "created_at": datetime.now(),
        "updated_at": datetime.now(),
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

FastAPIでは、アプリケーションを起動すると以下のURLで自動生成されたドキュメントにアクセスできます：

- Swagger UI: `http://localhost:8000/docs`
- ReDoc: `http://localhost:8000/redoc`
- OpenAPI JSON: `http://localhost:8000/openapi.json`

---

## 仕様書からのコード生成

OpenAPI仕様書から、クライアントコードやサーバースタブを自動生成できます。

### OpenAPI Generatorの使用

OpenAPI Generatorは、OpenAPI仕様書から様々な言語のコードを生成するツールです。

**公式サイト**: https://openapi-generator.tech/

#### インストール

```bash
# npm経由
npm install @openapitools/openapi-generator-cli -g

# Homebrew経由（macOS）
brew install openapi-generator

# Docker経由
docker pull openapitools/openapi-generator-cli
```

#### TypeScriptクライアントの生成

```bash
openapi-generator-cli generate \
  -i openapi.yaml \
  -g typescript-axios \
  -o ./generated/client
```

生成されたクライアントの使用例：

```typescript
import { Configuration, UsersApi } from './generated/client';

// APIクライアントの設定
const config = new Configuration({
  basePath: 'https://api.example.com/v1',
  accessToken: 'your-jwt-token',
});

const usersApi = new UsersApi(config);

// ユーザー一覧を取得
const response = await usersApi.listUsers(1, 20);
console.log(response.data);

// ユーザーを作成
const newUser = await usersApi.createUser({
  name: '山田太郎',
  email: 'yamada@example.com',
  password: 'SecurePass123!',
});
console.log(newUser.data);
```

#### サーバースタブの生成

Node.js (Express) のサーバースタブを生成：

```bash
openapi-generator-cli generate \
  -i openapi.yaml \
  -g nodejs-express-server \
  -o ./generated/server
```

#### サポートされる言語・フレームワーク

OpenAPI Generatorは、50以上の言語・フレームワークをサポートしています：

**クライアント**:
- TypeScript (axios, fetch, angular)
- JavaScript
- Python
- Java
- Kotlin
- Swift
- Go
- Ruby
- PHP
- C#

**サーバー**:
- Node.js (Express, NestJS)
- Python (FastAPI, Flask, Django)
- Java (Spring)
- Go
- Ruby (Rails, Sinatra)
- PHP (Laravel, Symfony)
- C# (ASP.NET Core)

利用可能なジェネレータの一覧：

```bash
openapi-generator-cli list
```

### 実践的なワークフロー

#### パターン1: API Firstアプローチ

1. OpenAPI仕様書を先に作成
2. レビュー・承認
3. サーバースタブとクライアントコードを生成
4. 実装

```bash
# 1. 仕様書を作成
vim openapi.yaml

# 2. 仕様書のバリデーション
npx @apidevtools/swagger-cli validate openapi.yaml

# 3. サーバースタブを生成
openapi-generator-cli generate -i openapi.yaml -g nodejs-express-server -o ./server

# 4. クライアントコードを生成
openapi-generator-cli generate -i openapi.yaml -g typescript-axios -o ./client

# 5. 実装
cd server && npm install && npm start
```

#### パターン2: Code Firstアプローチ

1. コードを先に実装
2. コードから仕様書を自動生成
3. レビュー・公開

```bash
# 1. コードを実装（NestJS, tsoa, FastAPIなど）

# 2. 仕様書を生成（例: NestJS）
npm run build
# 仕様書はアプリ起動時に自動生成される

# 3. 生成された仕様書を保存
curl http://localhost:3000/api-docs-json > openapi.json

# 4. YAMLに変換（オプション）
npx swagger2openapi openapi.json -o openapi.yaml
```

---

## OpenAPIのベストプラクティス

### 1. バージョニング戦略

APIバージョンを明確に管理しましょう。

```yaml
info:
  version: 1.0.0  # APIのバージョン

servers:
  - url: https://api.example.com/v1  # URLにバージョンを含める
```

**推奨される方法**:
- URLパスにバージョンを含める（`/v1`, `/v2`）
- メジャーバージョンのみURLに含める
- マイナーバージョン・パッチバージョンは`info.version`で管理

### 2. スキーマの再利用

`$ref`を活用して、スキーマを再利用しましょう。

```yaml
# ❌ 悪い例: スキーマが重複
paths:
  /users:
    get:
      responses:
        '200':
          content:
            application/json:
              schema:
                type: object
                properties:
                  id:
                    type: string
                  name:
                    type: string

  /users/{id}:
    get:
      responses:
        '200':
          content:
            application/json:
              schema:
                type: object
                properties:
                  id:
                    type: string
                  name:
                    type: string

# ✅ 良い例: スキーマを再利用
components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: string
        name:
          type: string

paths:
  /users:
    get:
      responses:
        '200':
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/User'

  /users/{id}:
    get:
      responses:
        '200':
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
```

### 3. 例（examples）の充実

具体的な例を豊富に提供しましょう。

```yaml
components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: string
          example: "user_123"  # 各フィールドに例を追加
        name:
          type: string
          example: "山田太郎"
        email:
          type: string
          format: email
          example: "yamada@example.com"

paths:
  /users:
    post:
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UserCreateInput'
            examples:
              basic:
                summary: 基本的なユーザー
                value:
                  name: "山田太郎"
                  email: "yamada@example.com"
                  password: "SecurePass123!"
              admin:
                summary: 管理者ユーザー
                value:
                  name: "管理者"
                  email: "admin@example.com"
                  password: "AdminPass123!"
                  role: "admin"
```

### 4. エラーレスポンスの統一

エラーレスポンスの形式を統一しましょう。

```yaml
components:
  schemas:
    Error:
      type: object
      required:
        - code
        - message
      properties:
        code:
          type: string
          description: エラーコード
        message:
          type: string
          description: エラーメッセージ
        details:
          type: object
          description: エラーの詳細情報

  responses:
    BadRequest:
      description: リクエストが不正です
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'

    Unauthorized:
      description: 認証が必要です
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'

    NotFound:
      description: リソースが見つかりません
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
```

すべてのエンドポイントで同じエラーレスポンスを使用：

```yaml
paths:
  /users/{userId}:
    get:
      responses:
        '200':
          description: 成功
        '401':
          $ref: '#/components/responses/Unauthorized'
        '404':
          $ref: '#/components/responses/NotFound'
```

### 5. バリデーションルールの明示

入力データのバリデーションルールを明確に記述しましょう。

```yaml
components:
  schemas:
    UserCreateInput:
      type: object
      required:
        - name
        - email
        - password
      properties:
        name:
          type: string
          minLength: 1        # 最小文字数
          maxLength: 50       # 最大文字数
          example: "山田太郎"
        email:
          type: string
          format: email       # メールアドレス形式
          example: "yamada@example.com"
        password:
          type: string
          format: password
          minLength: 8
          maxLength: 100
          pattern: '^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}$'
          description: |
            パスワード要件:
            - 8文字以上100文字以下
            - 大文字と小文字を含む
            - 数字を含む
            - 記号を含む
          example: "SecurePass123!"
```

### 6. operationIdの使用

各エンドポイントに一意な`operationId`を付けましょう。コード生成時に関数名として使用されます。

```yaml
paths:
  /users:
    get:
      operationId: listUsers  # 一意なID
      summary: ユーザー一覧を取得
    post:
      operationId: createUser
      summary: ユーザーを作成

  /users/{userId}:
    get:
      operationId: getUser
      summary: ユーザー詳細を取得
    put:
      operationId: updateUser
      summary: ユーザーを更新
    delete:
      operationId: deleteUser
      summary: ユーザーを削除
```

### 7. タグによるグループ化

関連するエンドポイントをタグでグループ化しましょう。

```yaml
tags:
  - name: auth
    description: 認証関連のエンドポイント
  - name: users
    description: ユーザー管理
  - name: posts
    description: 投稿管理
  - name: admin
    description: 管理者専用機能

paths:
  /auth/login:
    post:
      tags:
        - auth
      summary: ログイン

  /users:
    get:
      tags:
        - users
      summary: ユーザー一覧を取得

  /posts:
    get:
      tags:
        - posts
      summary: 投稿一覧を取得
```

### 8. ページネーションの標準化

ページネーションの形式を統一しましょう。

```yaml
components:
  parameters:
    PageParam:
      name: page
      in: query
      description: ページ番号（1から始まる）
      required: false
      schema:
        type: integer
        minimum: 1
        default: 1

    LimitParam:
      name: limit
      in: query
      description: 1ページあたりの件数
      required: false
      schema:
        type: integer
        minimum: 1
        maximum: 100
        default: 20

  schemas:
    Pagination:
      type: object
      required:
        - page
        - limit
        - total
        - totalPages
      properties:
        page:
          type: integer
          description: 現在のページ番号
        limit:
          type: integer
          description: 1ページあたりの件数
        total:
          type: integer
          description: 総件数
        totalPages:
          type: integer
          description: 総ページ数

paths:
  /users:
    get:
      parameters:
        - $ref: '#/components/parameters/PageParam'
        - $ref: '#/components/parameters/LimitParam'
      responses:
        '200':
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/User'
                  pagination:
                    $ref: '#/components/schemas/Pagination'
```

---

## OpenAPI仕様書のバリデーション

OpenAPI仕様書が正しく記述されているかをバリデーションするツールを使用しましょう。

### Swagger Editorでのバリデーション

Swagger Editorは、リアルタイムでOpenAPI仕様書をバリデーションします。

**オンライン版**: https://editor.swagger.io/

**ローカルで実行**:

```bash
docker run -p 80:8080 swaggerapi/swagger-editor
```

### CLIでのバリデーション

#### swagger-cli

```bash
npm install -g @apidevtools/swagger-cli

# バリデーション
swagger-cli validate openapi.yaml

# 成功時
openapi.yaml is valid

# エラー時
openapi.yaml is invalid:
  - Schema error at paths./users.get.responses.200
    should have required property 'description'
```

#### openapi-generator-cli

```bash
openapi-generator-cli validate -i openapi.yaml
```

### CIでのバリデーション

GitHub Actionsで自動的にバリデーションする例：

```yaml
# .github/workflows/openapi-validation.yml
name: OpenAPI Validation

on:
  pull_request:
    paths:
      - 'openapi.yaml'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Validate OpenAPI spec
        uses: char0n/swagger-editor-validate@v1
        with:
          definition-file: openapi.yaml

      - name: Generate documentation
        if: success()
        run: |
          npm install -g redoc-cli
          redoc-cli bundle openapi.yaml -o docs/api.html

      - name: Deploy documentation
        if: success() && github.ref == 'refs/heads/main'
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./docs
```

---

## OpenAPIツールエコシステム

OpenAPIを活用するための有用なツールを紹介します。

### ドキュメント生成

| ツール | 説明 | URL |
|--------|------|-----|
| Swagger UI | 対話的なAPIドキュメント | https://swagger.io/tools/swagger-ui/ |
| ReDoc | レスポンシブなAPIドキュメント | https://redocly.com/redoc/ |
| Stoplight Elements | モダンなUIコンポーネント | https://stoplight.io/open-source/elements |

### コード生成

| ツール | 説明 | URL |
|--------|------|-----|
| OpenAPI Generator | 多言語対応のコード生成 | https://openapi-generator.tech/ |
| Swagger Codegen | 公式コード生成ツール | https://swagger.io/tools/swagger-codegen/ |
| oapi-codegen | Go向けコード生成 | https://github.com/deepmap/oapi-codegen |

### エディタ

| ツール | 説明 | URL |
|--------|------|-----|
| Swagger Editor | Web/ローカルエディタ | https://editor.swagger.io/ |
| Stoplight Studio | ビジュアルエディタ | https://stoplight.io/studio/ |
| VS Code拡張 | OpenAPI (Swagger) Editor | マーケットプレイス |

### モックサーバー

| ツール | 説明 | URL |
|--------|------|-----|
| Prism | OpenAPIからモック生成 | https://stoplight.io/open-source/prism |
| Mockoon | GUIモックサーバー | https://mockoon.com/ |
| json-server | 簡易JSONモック | https://github.com/typicode/json-server |

### バリデーション

| ツール | 説明 | URL |
|--------|------|-----|
| swagger-cli | CLI バリデーションツール | https://apitools.dev/swagger-cli/ |
| Spectral | リンター・バリデーター | https://stoplight.io/open-source/spectral |

### ReDocの使用例

ReDocは、美しいAPIドキュメントを生成するツールです。

#### インストール

```bash
npm install -g redoc-cli
```

#### HTMLの生成

```bash
redoc-cli bundle openapi.yaml -o docs/api.html
```

#### Node.jsでの使用

```bash
npm install redoc-express
```

```javascript
const express = require('express');
const { setup } = require('redoc-express');

const app = express();

app.use('/api-docs', setup({
  specUrl: '/openapi.yaml',
  title: 'API Documentation',
}));

app.listen(3000);
```

---

## よくある課題と解決策

### 課題1: 仕様書とコードの乖離

**問題**: コードを更新してもOpenAPI仕様書の更新を忘れる

**解決策**:

1. **Code Firstアプローチを採用**: コードから仕様書を自動生成する
2. **CIでのバリデーション**: PRごとに仕様書の整合性をチェック
3. **自動生成スクリプト**: ビルド時に仕様書を自動生成

```json
// package.json
{
  "scripts": {
    "build": "tsc",
    "generate:spec": "tsoa spec",
    "prebuild": "npm run generate:spec"
  }
}
```

### 課題2: 大規模な仕様書の管理

**問題**: 単一のYAMLファイルが巨大になり、管理が困難

**解決策**: ファイルを分割し、`$ref`で参照する

```yaml
# openapi.yaml（メインファイル）
openapi: 3.0.3
info:
  title: User Management API
  version: 1.0.0

paths:
  /users:
    $ref: './paths/users.yaml'
  /users/{userId}:
    $ref: './paths/users-id.yaml'

components:
  schemas:
    User:
      $ref: './schemas/User.yaml'
    Error:
      $ref: './schemas/Error.yaml'
```

```yaml
# paths/users.yaml
get:
  summary: ユーザー一覧を取得
  responses:
    '200':
      description: 成功
      content:
        application/json:
          schema:
            type: array
            items:
              $ref: '../schemas/User.yaml'
```

```yaml
# schemas/User.yaml
type: object
required:
  - id
  - name
  - email
properties:
  id:
    type: string
    example: "user_123"
  name:
    type: string
    example: "山田太郎"
  email:
    type: string
    format: email
    example: "yamada@example.com"
```

### 課題3: 複数のAPIバージョンの管理

**問題**: v1とv2のAPIを同時に運用する必要がある

**解決策**: バージョンごとに仕様書を分ける

```
docs/
├── openapi-v1.yaml
├── openapi-v2.yaml
└── README.md
```

```javascript
// server.js
const swaggerV1 = YAML.load('./docs/openapi-v1.yaml');
const swaggerV2 = YAML.load('./docs/openapi-v2.yaml');

app.use('/api-docs/v1', swaggerUi.serve, swaggerUi.setup(swaggerV1));
app.use('/api-docs/v2', swaggerUi.serve, swaggerUi.setup(swaggerV2));
```

### 課題4: 認証情報のテスト

**問題**: Swagger UIでAPIをテストする際、毎回トークンを入力するのが面倒

**解決策**: 開発環境用のデフォルトトークンを設定

```javascript
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerDocument, {
  swaggerOptions: {
    persistAuthorization: true,  // 認証情報を保持
    // 開発環境のみ: デフォルトトークンを設定
    ...(process.env.NODE_ENV === 'development' && {
      authAction: {
        bearerAuth: {
          name: 'bearerAuth',
          schema: { type: 'http', scheme: 'bearer' },
          value: 'dev-token-for-testing'
        }
      }
    })
  }
}));
```

---

## まとめ

この章では、OpenAPI SpecificationとSwagger UIを活用したAPI仕様書の自動化について学びました。

**重要なポイント**:

1. **OpenAPIの基本構造**: `info`, `servers`, `paths`, `components`, `security`の各セクションの役割を理解する

2. **仕様書の作成方法**:
   - 手動でYAML/JSON形式で記述
   - コードから自動生成（tsoa, NestJS, FastAPIなど）
   - どちらのアプローチもメリット・デメリットがある

3. **Swagger UIによる可視化**: 対話的なドキュメントで、APIをブラウザから直接テストできる

4. **コード生成**: OpenAPI仕様書から、クライアントコードやサーバースタブを自動生成できる

5. **ベストプラクティス**:
   - スキーマの再利用（`$ref`）
   - 具体的な例の提供
   - エラーレスポンスの統一
   - バリデーションルールの明示
   - タグによるグループ化

6. **ツールエコシステム**: Swagger UI, ReDoc, OpenAPI Generator, Prismなど、豊富なツールが利用可能

OpenAPIを活用することで、APIドキュメントの作成・更新・公開のプロセスを大幅に効率化でき、開発者体験を向上させることができます。

---

## チェックリスト

- [ ] OpenAPI Specificationの基本構造を理解している
- [ ] YAML/JSON形式でOpenAPI仕様書を作成できる
- [ ] Swagger UIでドキュメントを可視化できる
- [ ] コードからOpenAPI仕様書を自動生成できる
- [ ] OpenAPI仕様書からコードを生成できる
- [ ] 再利用可能なコンポーネントを活用している
- [ ] 認証・認可の定義方法を理解している
- [ ] エラーレスポンスが統一されている
- [ ] バリデーションルールが明示されている
- [ ] OpenAPI仕様書のバリデーションを実施している

---

## 次のステップ

第12章「アーキテクチャ図の作成」では、システム全体の構造を視覚化する方法を学びます。OpenAPI仕様書がAPIの詳細を記述するのに対し、アーキテクチャ図はシステム全体の構造と関係性を示します。両者を組み合わせることで、技術ドキュメントの完成度が大きく向上します。

次章では、C4モデルやMermaid.jsを使った図の作成方法、適切な抽象度の選択など、効果的なアーキテクチャ図の作成テクニックを詳しく解説します。
