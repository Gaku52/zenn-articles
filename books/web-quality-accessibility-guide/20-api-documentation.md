# API仕様書 - 明確なインターフェース定義

## API仕様書の重要性

API仕様書は、フロントエンドとバックエンドのインターフェースを明確にします。適切なAPI仕様書により、以下のメリットが期待されます:

- **開発効率の向上**: フロントエンドとバックエンドの並行開発が可能
- **コミュニケーションコストの削減**: 仕様が明確になり、認識齟齬が減少
- **自動テストの生成**: 仕様からテストコードを自動生成可能
- **型定義の自動生成**: TypeScript型定義を自動生成可能
- **ドキュメントの鮮度維持**: コードと仕様の同期が容易

## OpenAPI (Swagger)

OpenAPI Specification（OAS）は、RESTful APIを記述するための業界標準フォーマットです[^openapi-spec]。

[^openapi-spec]: [OpenAPI Specification - OpenAPI Initiative](https://spec.openapis.org/oas/latest.html)

### 基本構造

```yaml
openapi: 3.0.0
info:
  title: User API
  version: 1.0.0
  description: ユーザー管理API

servers:
  - url: https://api.example.com/v1
    description: Production
  - url: http://localhost:3000/v1
    description: Development

paths:
  /users:
    get:
      summary: ユーザー一覧取得
      tags:
        - Users
      parameters:
        - name: limit
          in: query
          schema:
            type: integer
            default: 10
        - name: offset
          in: query
          schema:
            type: integer
            default: 0
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/User'

    post:
      summary: ユーザー作成
      tags:
        - Users
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
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
                $ref: '#/components/schemas/Error'

  /users/{id}:
    get:
      summary: ユーザー詳細取得
      tags:
        - Users
      parameters:
        - name: id
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
        '404':
          description: ユーザーが見つかりません

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
          example: "123"
        name:
          type: string
          example: "山田太郎"
        email:
          type: string
          format: email
          example: "yamada@example.com"
        createdAt:
          type: string
          format: date-time

    CreateUserRequest:
      type: object
      required:
        - name
        - email
      properties:
        name:
          type: string
          minLength: 1
        email:
          type: string
          format: email

    Error:
      type: object
      properties:
        message:
          type: string
        code:
          type: string
```

## TypeScript型定義

OpenAPI仕様からTypeScript型定義を自動生成することができます。

### openapi-typescript を使用

```bash
# インストール
pnpm add -D openapi-typescript

# 型定義を生成
npx openapi-typescript openapi.yaml -o types/api.ts
```

生成される型定義:

```typescript
// types/api.ts（自動生成）
export interface paths {
  '/users': {
    get: operations['getUsers']
    post: operations['createUser']
  }
  '/users/{id}': {
    get: operations['getUser']
  }
}

export interface components {
  schemas: {
    User: {
      id: string
      name: string
      email: string
      createdAt: string
    }
    CreateUserRequest: {
      name: string
      email: string
    }
    Error: {
      message: string
      code: string
    }
  }
}

export interface operations {
  getUsers: {
    parameters: {
      query?: {
        limit?: number
        offset?: number
      }
    }
    responses: {
      200: {
        content: {
          'application/json': components['schemas']['User'][]
        }
      }
    }
  }
  createUser: {
    requestBody: {
      content: {
        'application/json': components['schemas']['CreateUserRequest']
      }
    }
    responses: {
      201: {
        content: {
          'application/json': components['schemas']['User']
        }
      }
      400: {
        content: {
          'application/json': components['schemas']['Error']
        }
      }
    }
  }
}
```

### 型定義の使用例

```typescript
import type { components, operations } from './types/api'

type User = components['schemas']['User']
type CreateUserRequest = components['schemas']['CreateUserRequest']

async function createUser(data: CreateUserRequest): Promise<User> {
  const response = await fetch('/api/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  })

  if (!response.ok) {
    throw new Error('Failed to create user')
  }

  return response.json()
}
```

## ベストプラクティス

### 1. バージョニング

APIのバージョン管理は、互換性を保ちながら進化させるために重要です。

```yaml
# ✅ 良い例: URLにバージョンを含める
servers:
  - url: https://api.example.com/v1
    description: Production
  - url: https://api.example.com/v2
    description: Production (v2)
```

**バージョニング戦略**:
- **URLパス**: `/v1/users`、`/v2/users`（推奨）
- **ヘッダー**: `Accept: application/vnd.api.v1+json`
- **クエリパラメータ**: `/users?version=1`（非推奨）

### 2. エラーレスポンスの標準化

一貫したエラーレスポンス形式を定義します。

```yaml
components:
  schemas:
    Error:
      type: object
      required:
        - message
        - code
      properties:
        message:
          type: string
          description: エラーメッセージ
          example: "Validation failed"
        code:
          type: string
          description: エラーコード
          example: "VALIDATION_ERROR"
        errors:
          type: array
          description: 詳細エラー
          items:
            type: object
            properties:
              field:
                type: string
                example: "email"
              message:
                type: string
                example: "Invalid email format"
```

### 3. ページネーション

大量のデータを返すエンドポイントには、ページネーションを実装します。

```yaml
paths:
  /users:
    get:
      parameters:
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
            minimum: 1
            maximum: 100
        - name: offset
          in: query
          schema:
            type: integer
            default: 0
            minimum: 0
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
                    type: object
                    properties:
                      total:
                        type: integer
                        example: 100
                      limit:
                        type: integer
                        example: 20
                      offset:
                        type: integer
                        example: 0
```

### 4. 認証・認可

認証情報の扱いを明確に定義します。

```yaml
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - bearerAuth: []

paths:
  /users/me:
    get:
      summary: 現在のユーザー情報を取得
      security:
        - bearerAuth: []
      responses:
        '200':
          description: 成功
        '401':
          description: 認証エラー
```

## 自動ドキュメント生成

OpenAPI仕様から、インタラクティブなAPIドキュメントを自動生成できます。

### Swagger UI

```bash
# Swagger UIのインストール
pnpm add swagger-ui-react

# Next.jsでの使用例
# app/api-docs/page.tsx
'use client'

import SwaggerUI from 'swagger-ui-react'
import 'swagger-ui-react/swagger-ui.css'

export default function ApiDocsPage() {
  return <SwaggerUI url="/openapi.yaml" />
}
```

### Redoc

Redocは、より洗練されたドキュメントUIを提供します。

```bash
# Redocのインストール
pnpm add redoc

# 静的HTMLの生成
npx redoc-cli bundle openapi.yaml -o docs/api.html
```

## CI/CDでの活用

OpenAPI仕様を活用した自動化の例:

```yaml
# .github/workflows/api-validation.yml
name: API Validation

on: [push, pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # OpenAPI仕様の検証
      - name: Validate OpenAPI Spec
        uses: char0n/swagger-editor-validate@v1
        with:
          definition-file: openapi.yaml

      # TypeScript型定義の生成
      - name: Generate TypeScript Types
        run: |
          npx openapi-typescript openapi.yaml -o types/api.ts
          git diff --exit-code types/api.ts || \
            (echo "Generated types differ from committed types" && exit 1)
```

## まとめ

API仕様書は、OpenAPIで明確に定義し、TypeScript型定義を自動生成することで、開発効率と品質を向上させることができます。

### チェックリスト

- 全エンドポイントが記載されている
- リクエスト/レスポンス形式が明確
- エラーレスポンスが定義されている
- 例が含まれている

## 次のステップ

次章では、アーキテクチャ図の描き方について解説します。
