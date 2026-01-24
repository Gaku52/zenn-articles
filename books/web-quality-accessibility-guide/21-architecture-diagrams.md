# アーキテクチャ図 - システムの可視化

## アーキテクチャ図の重要性

アーキテクチャ図は、システムの構造とデータフローを視覚的に表現します。

## Mermaid

Mermaidは、テキストベースで図を作成できるツールです。GitHubやZennで直接レンダリングされます。

### システム構成図

\`\`\`mermaid
graph TB
    Client[Client<br/>React]
    API[API Server<br/>Next.js]
    DB[(Database<br/>PostgreSQL)]
    Cache[(Cache<br/>Redis)]

    Client -->|HTTPS| API
    API -->|Query| DB
    API -->|Cache| Cache
\`\`\`

### シーケンス図

\`\`\`mermaid
sequenceDiagram
    actor User
    participant Client
    participant API
    participant DB

    User->>Client: ログイン
    Client->>API: POST /api/auth/login
    API->>DB: ユーザー検証
    DB-->>API: ユーザー情報
    API-->>Client: JWT トークン
    Client-->>User: ログイン成功
\`\`\`

### ER図

\`\`\`mermaid
erDiagram
    User ||--o{ Post : writes
    User {
        string id PK
        string name
        string email
        datetime createdAt
    }
    Post {
        string id PK
        string userId FK
        string title
        text content
        datetime createdAt
    }
\`\`\`

## C4モデル

システムを複数のレベルで表現します：

1. **コンテキスト**: システム全体と外部システム
2. **コンテナ**: アプリケーション、データベース等
3. **コンポーネント**: 各コンテナ内の主要コンポーネント
4. **コード**: クラス図等

## まとめ

アーキテクチャ図は、Mermaid等のテキストベースツールで管理することを推奨します。

### チェックリスト

- システム構成図がある
- データフローが明確
- 主要コンポーネントが表現されている
- 最新に保たれている

## 次のステップ

最終章では、アクセシビリティ自動化について解説します。
