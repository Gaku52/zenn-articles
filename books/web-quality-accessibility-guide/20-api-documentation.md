# API仕様書 - 明確なインターフェース定義

## API仕様書の重要性

API仕様書は、フロントエンドとバックエンドのインターフェースを明確にします。

## OpenAPI (Swagger)

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

```typescript
// types/api.ts
export interface User {
  id: string
  name: string
  email: string
  createdAt: string
}

export interface CreateUserRequest {
  name: string
  email: string
}

export interface ApiError {
  message: string
  code: string
}
```

## まとめ

API仕様書は、OpenAPIまたはTypeScript型定義で明確に定義します。

### チェックリスト

- 全エンドポイントが記載されている
- リクエスト/レスポンス形式が明確
- エラーレスポンスが定義されている
- 例が含まれている

## 次のステップ

次章では、アーキテクチャ図の描き方について解説します。
