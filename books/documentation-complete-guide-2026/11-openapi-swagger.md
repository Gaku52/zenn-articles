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

---

## OpenAPI Specificationの基礎

### OpenAPIとは

OpenAPI Specification（OAS）は、RESTful APIを記述するための標準仕様です。OpenAPI Initiative（Linux Foundationのプロジェクト）によって管理されており、多くの企業やツールがサポートしています。

**公式サイト**: https://www.openapis.org/

OpenAPIの前身は「Swagger Specification」でしたが、2016年にOpenAPI Specificationとして標準化されました。本章では、最も広く使われているOpenAPI 3.0系を中心に解説します。

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

まずは、最もシンプルなOpenAPI仕様書から始めましょう：

```yaml
openapi: 3.0.3
info:
  title: Simple User API
  description: ユーザー管理のためのシンプルなAPI
  version: 1.0.0
  contact:
    name: API Support
    email: support@example.com

servers:
  - url: https://api.example.com/v1
    description: 本番環境

paths:
  /users:
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

### infoセクション: APIのメタ情報

`info`セクションには、API全体のメタ情報を記述します：

```yaml
info:
  title: User Management API
  description: |
    ユーザー管理のためのRESTful API。

    ## 主な機能
    - ユーザーの作成、取得、更新、削除
    - ユーザー認証とトークン管理
  version: 1.0.0
  contact:
    name: API Support Team
    email: api-support@example.com
  license:
    name: MIT
```

### pathsセクション: エンドポイントの定義

`paths`セクションは、OpenAPI仕様書の中核部分で、すべてのAPIエンドポイントを定義します。

#### パスパラメータの定義

```yaml
paths:
  /users/{userId}:
    get:
      summary: ユーザー詳細を取得
      parameters:
        - name: userId
          in: path
          required: true
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
```

#### クエリパラメータの定義

```yaml
paths:
  /users:
    get:
      summary: ユーザー一覧を取得
      parameters:
        - name: page
          in: query
          description: ページ番号（1から始まる）
          required: false
          schema:
            type: integer
            minimum: 1
            default: 1
        - name: limit
          in: query
          description: 1ページあたりの件数
          required: false
          schema:
            type: integer
            minimum: 1
            maximum: 100
            default: 20
```

#### リクエストボディの定義

```yaml
paths:
  /users:
    post:
      summary: ユーザーを作成
      requestBody:
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
      responses:
        '201':
          description: 作成成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '400':
          description: バリデーションエラー
```

### componentsセクション: 再利用可能なコンポーネント

`components`セクションには、仕様書全体で再利用可能なコンポーネントを定義します：

```yaml
components:
  schemas:
    User:
      type: object
      required:
        - id
        - name
        - email
      properties:
        id:
          type: string
          description: ユーザーID
          example: "user_123"
        name:
          type: string
          minLength: 1
          maxLength: 50
          example: "山田太郎"
        email:
          type: string
          format: email
          example: "yamada@example.com"
        role:
          type: string
          enum: [admin, user, guest]
          default: user

    UserCreateInput:
      type: object
      required:
        - name
        - email
        - password
      properties:
        name:
          type: string
          minLength: 1
          maxLength: 50
        email:
          type: string
          format: email
        password:
          type: string
          format: password
          minLength: 8

    Error:
      type: object
      required:
        - code
        - message
      properties:
        code:
          type: string
          example: "USER_NOT_FOUND"
        message:
          type: string
          example: "指定されたユーザーが見つかりません"
```

---

## セキュリティの定義

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

security:
  - apiKey: []
```

---

## Swagger UIによる可視化

### Swagger UIとは

Swagger UIは、OpenAPI仕様書から自動的にWebベースのドキュメントを生成するツールです。以下の機能を提供します：

- エンドポイント一覧の表示
- リクエスト/レスポンスの詳細表示
- ブラウザから直接APIを試せる"Try it out"機能
- 認証のサポート

### Swagger UIのセットアップ

#### Node.jsで使用

```bash
npm install swagger-ui-express
```

```javascript
const express = require('express');
const swaggerUi = require('swagger-ui-express');
const YAML = require('yamljs');

const app = express();
const swaggerDocument = YAML.load('./openapi.yaml');

app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerDocument, {
  customSiteTitle: "User API Documentation",
  swaggerOptions: {
    persistAuthorization: true,
  }
}));

app.listen(3000, () => {
  console.log('API Docs: http://localhost:3000/api-docs');
});
```

---

## コードからの仕様書自動生成

### NestJS の例

NestJSには、OpenAPI仕様書を生成する機能が組み込まれています。

```bash
npm install @nestjs/swagger
```

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('User Management API')
    .setDescription('ユーザー管理のためのRESTful API')
    .setVersion('1.0.0')
    .addBearerAuth()
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api-docs', app, document);

  await app.listen(3000);
}
bootstrap();
```

```typescript
// src/users/users.controller.ts
import { Controller, Get, Post, Body, Param } from '@nestjs/common';
import { ApiTags, ApiOperation, ApiResponse } from '@nestjs/swagger';

@ApiTags('users')
@Controller('users')
export class UsersController {
  @Get()
  @ApiOperation({ summary: 'ユーザー一覧を取得' })
  @ApiResponse({ status: 200, description: '成功' })
  async findAll() {
    return [];
  }

  @Post()
  @ApiOperation({ summary: 'ユーザーを作成' })
  @ApiResponse({ status: 201, description: '作成成功' })
  async create(@Body() createUserDto: any) {
    return {};
  }
}
```

### FastAPI (Python) の例

FastAPIは、型ヒントから自動的にOpenAPI仕様書を生成します。

```python
from fastapi import FastAPI, Query
from pydantic import BaseModel, EmailStr

app = FastAPI(
    title="User Management API",
    description="ユーザー管理のためのRESTful API",
    version="1.0.0",
)

class User(BaseModel):
    id: str
    name: str
    email: EmailStr
    role: str = "user"

@app.get("/users", response_model=list[User])
async def list_users(
    page: int = Query(1, ge=1, description="ページ番号"),
    limit: int = Query(20, ge=1, le=100, description="1ページあたりの件数"),
):
    return []

@app.post("/users", response_model=User, status_code=201)
async def create_user(user: User):
    return user
```

FastAPIでは、アプリケーションを起動すると以下のURLで自動生成されたドキュメントにアクセスできます：

- Swagger UI: `http://localhost:8000/docs`
- ReDoc: `http://localhost:8000/redoc`
- OpenAPI JSON: `http://localhost:8000/openapi.json`

---

## 仕様書からのコード生成

### OpenAPI Generatorの使用

OpenAPI Generatorは、OpenAPI仕様書から様々な言語のコードを生成するツールです。

```bash
# TypeScriptクライアントの生成
openapi-generator-cli generate \
  -i openapi.yaml \
  -g typescript-axios \
  -o ./generated/client
```

生成されたクライアントの使用例：

```typescript
import { Configuration, UsersApi } from './generated/client';

const config = new Configuration({
  basePath: 'https://api.example.com/v1',
  accessToken: 'your-jwt-token',
});

const usersApi = new UsersApi(config);
const response = await usersApi.listUsers(1, 20);
```

---

## OpenAPIのベストプラクティス

### 1. スキーマの再利用

`$ref`を活用して、スキーマを再利用しましょう。

```yaml
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
```

### 2. 例（examples）の充実

具体的な例を豊富に提供しましょう。

```yaml
components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: string
          example: "user_123"
        name:
          type: string
          example: "山田太郎"
```

### 3. エラーレスポンスの統一

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
        message:
          type: string

  responses:
    NotFound:
      description: リソースが見つかりません
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
```

### 4. バリデーションルールの明示

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
          minLength: 1
          maxLength: 50
        email:
          type: string
          format: email
        password:
          type: string
          minLength: 8
          maxLength: 100
          pattern: '^(?=.*[a-z])(?=.*[A-Z])(?=.*\d).*$'
```

### 5. operationIdの使用

各エンドポイントに一意な`operationId`を付けましょう。

```yaml
paths:
  /users:
    get:
      operationId: listUsers
      summary: ユーザー一覧を取得
    post:
      operationId: createUser
      summary: ユーザーを作成
```

### 6. タグによるグループ化

```yaml
tags:
  - name: auth
    description: 認証関連
  - name: users
    description: ユーザー管理

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
```

---

## OpenAPI仕様書のバリデーション

### CLIでのバリデーション

```bash
npm install -g @apidevtools/swagger-cli

# バリデーション
swagger-cli validate openapi.yaml
```

### CIでのバリデーション

GitHub Actionsで自動的にバリデーションする例：

```yaml
name: OpenAPI Validation

on:
  pull_request:
    paths:
      - 'openapi.yaml'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Validate OpenAPI spec
        uses: char0n/swagger-editor-validate@v1
        with:
          definition-file: openapi.yaml
```

---

## よくある課題と解決策

### 課題1: 仕様書とコードの乖離

**解決策**:
- Code Firstアプローチを採用（コードから仕様書を自動生成）
- CIでのバリデーション
- 自動生成スクリプトをビルドプロセスに組み込む

### 課題2: 大規模な仕様書の管理

**解決策**: ファイルを分割し、`$ref`で参照する

```yaml
# openapi.yaml
paths:
  /users:
    $ref: './paths/users.yaml'

components:
  schemas:
    User:
      $ref: './schemas/User.yaml'
```

---

## まとめ

この章では、OpenAPI SpecificationとSwagger UIを活用したAPI仕様書の自動化について学びました。

**重要なポイント**:

1. **OpenAPIの基本構造**: `info`, `servers`, `paths`, `components`, `security`の役割
2. **仕様書の作成方法**: 手動記述とコードからの自動生成
3. **Swagger UIによる可視化**: 対話的なドキュメント
4. **コード生成**: クライアントコードやサーバースタブの自動生成
5. **ベストプラクティス**: スキーマの再利用、例の提供、エラーの統一

OpenAPIを活用することで、APIドキュメントの作成・更新・公開のプロセスを大幅に効率化できます。

---

## チェックリスト

- [ ] OpenAPI Specificationの基本構造を理解している
- [ ] YAML形式でOpenAPI仕様書を作成できる
- [ ] Swagger UIでドキュメントを可視化できる
- [ ] コードからOpenAPI仕様書を自動生成できる
- [ ] 再利用可能なコンポーネントを活用している
- [ ] 認証・認可の定義方法を理解している
- [ ] エラーレスポンスが統一されている
- [ ] バリデーションルールが明示されている

---

## 次のステップ

第12章「アーキテクチャ図の作成」では、システム全体の構造を視覚化する方法を学びます。OpenAPI仕様書がAPIの詳細を記述するのに対し、アーキテクチャ図はシステム全体の構造と関係性を示します。
