---
title: "アーキテクチャ図 - システムの可視化"
---

# アーキテクチャ図 - システムの可視化

## アーキテクチャ図の重要性

アーキテクチャ図は、システムの構造、コンポーネント間の関係、データフロー、技術スタックを視覚的に表現する重要なドキュメントです。適切に作成されたアーキテクチャ図は、以下のメリットをもたらします：

- **チーム内のコミュニケーション円滑化**: 新規メンバーのオンボーディング時間を短縮し、共通理解を促進
- **設計判断の記録**: アーキテクチャの意思決定プロセスと理由を文書化
- **問題の早期発見**: システムのボトルネックや潜在的な問題を視覚的に特定
- **保守性の向上**: システムの全体像を把握し、変更影響範囲を正確に予測

## Mermaid記法によるアーキテクチャ図

Mermaidは、テキストベースで図を作成できるツールです。GitHubやZennで直接レンダリングされ、バージョン管理との親和性が高いことが特徴です。

### システム構成図（フローチャート）

システム全体のコンポーネントと通信経路を表現します。

```mermaid
graph TB
    subgraph "クライアント層"
        WebApp[Web Application<br/>React + TypeScript]
        MobileApp[Mobile App<br/>React Native]
    end

    subgraph "API層"
        Gateway[API Gateway<br/>Next.js API Routes]
        GraphQL[GraphQL Server<br/>Apollo Server]
    end

    subgraph "バックエンドサービス"
        Auth[認証サービス<br/>Auth0]
        Storage[ストレージ<br/>S3]
    end

    subgraph "データ層"
        DB[(PostgreSQL<br/>RDS)]
        Cache[(Redis<br/>ElastiCache)]
        Queue[(メッセージキュー<br/>SQS)]
    end

    WebApp -->|HTTPS| Gateway
    MobileApp -->|HTTPS| Gateway
    Gateway -->|GraphQL| GraphQL
    GraphQL -->|認証| Auth
    GraphQL -->|クエリ| DB
    GraphQL -->|キャッシュ| Cache
    GraphQL -->|非同期処理| Queue
    GraphQL -->|ファイル保存| Storage
```

### シーケンス図

ユーザー操作からシステム応答までの時系列処理フローを表現します。

```mermaid
sequenceDiagram
    actor User as ユーザー
    participant Client as クライアント
    participant Gateway as APIゲートウェイ
    participant Auth as 認証サービス
    participant API as GraphQL API
    participant Cache as Redisキャッシュ
    participant DB as データベース

    User->>Client: ログイン要求
    Client->>Gateway: POST /api/auth/login
    Gateway->>Auth: 認証情報検証
    Auth-->>Gateway: アクセストークン発行
    Gateway-->>Client: JWT + リフレッシュトークン

    Note over Client: ユーザー情報取得

    Client->>Gateway: GET /api/user<br/>Authorization: Bearer {token}
    Gateway->>API: GraphQL Query
    API->>Cache: キャッシュ確認

    alt キャッシュヒット
        Cache-->>API: ユーザー情報
    else キャッシュミス
        API->>DB: SELECT * FROM users
        DB-->>API: ユーザー情報
        API->>Cache: キャッシュ保存（TTL: 300s）
    end

    API-->>Gateway: ユーザー情報
    Gateway-->>Client: JSON Response
    Client-->>User: ダッシュボード表示
```

### ER図（エンティティ関係図）

データベーススキーマとテーブル間のリレーションシップを表現します。

```mermaid
erDiagram
    User ||--o{ Post : "執筆"
    User ||--o{ Comment : "投稿"
    Post ||--o{ Comment : "受付"
    Post }o--|| Category : "分類"
    Post }o--o{ Tag : "付与"

    User {
        uuid id PK "主キー"
        string email UK "メールアドレス"
        string name "表示名"
        string password_hash "ハッシュ化パスワード"
        datetime created_at "作成日時"
        datetime updated_at "更新日時"
    }

    Post {
        uuid id PK "主キー"
        uuid user_id FK "著者ID"
        uuid category_id FK "カテゴリID"
        string title "タイトル"
        text content "本文"
        string status "ステータス"
        int view_count "閲覧数"
        datetime published_at "公開日時"
        datetime created_at "作成日時"
        datetime updated_at "更新日時"
    }

    Category {
        uuid id PK "主キー"
        string name UK "カテゴリ名"
        string slug UK "URLスラッグ"
        text description "説明"
    }

    Tag {
        uuid id PK "主キー"
        string name UK "タグ名"
        string slug UK "URLスラッグ"
    }

    Comment {
        uuid id PK "主キー"
        uuid post_id FK "投稿ID"
        uuid user_id FK "コメント者ID"
        text content "コメント本文"
        datetime created_at "作成日時"
    }
```

### 状態遷移図

ワークフローやステータス管理を表現します。

```mermaid
stateDiagram-v2
    [*] --> Draft: 下書き作成

    Draft --> InReview: レビュー依頼
    Draft --> Archived: 破棄

    InReview --> Draft: 差し戻し
    InReview --> Approved: 承認
    InReview --> Rejected: 却下

    Approved --> Published: 公開
    Approved --> Scheduled: 公開予約

    Scheduled --> Published: 予約時刻到達
    Scheduled --> Draft: 予約キャンセル

    Published --> Draft: 非公開化
    Published --> [*]: アーカイブ

    Rejected --> [*]: 完全削除
    Archived --> [*]: 完全削除

    note right of Draft
        編集可能
        レビュー可能
    end note

    note right of Published
        一般公開中
        編集不可
    end note
```

### ガントチャート（プロジェクト管理）

プロジェクトのタイムラインとタスク依存関係を表現します。

```mermaid
gantt
    title Webアプリケーション開発スケジュール
    dateFormat YYYY-MM-DD
    section 要件定義
        要件ヒアリング           :a1, 2024-01-01, 7d
        要件定義書作成           :a2, after a1, 5d
        レビュー・承認           :a3, after a2, 3d
    section 設計
        システム設計             :b1, after a3, 10d
        データベース設計         :b2, after a3, 10d
        UI/UX設計               :b3, after a3, 14d
    section 開発
        フロントエンド開発       :c1, after b3, 21d
        バックエンド開発         :c2, after b1, 21d
        API統合                 :c3, after c1, 7d
    section テスト
        単体テスト              :d1, after c2, 7d
        統合テスト              :d2, after c3, 7d
        E2Eテスト               :d3, after d2, 5d
    section デプロイ
        ステージング環境         :e1, after d3, 3d
        本番環境リリース         :milestone, e2, after e1, 1d
```

### クラス図（TypeScript）

オブジェクト指向設計におけるクラス構造を表現します。

```mermaid
classDiagram
    class User {
        -id: string
        -email: string
        -name: string
        -passwordHash: string
        +authenticate(password: string): boolean
        +updateProfile(data: ProfileData): void
        +getPosts(): Post[]
    }

    class Post {
        -id: string
        -userId: string
        -title: string
        -content: string
        -status: PostStatus
        +publish(): void
        +archive(): void
        +addComment(comment: Comment): void
    }

    class Comment {
        -id: string
        -postId: string
        -userId: string
        -content: string
        +edit(newContent: string): void
        +delete(): void
    }

    class Category {
        -id: string
        -name: string
        -slug: string
        +getPosts(): Post[]
    }

    User "1" --> "*" Post : 執筆
    User "1" --> "*" Comment : 投稿
    Post "1" --> "*" Comment : 保持
    Post "*" --> "1" Category : 所属
```

## C4モデルによるアーキテクチャ設計

C4モデルは、ソフトウェアアーキテクチャを4つの抽象度レベルで表現するフレームワークです。

### Level 1: システムコンテキスト図

システム全体と外部システム・ユーザーの関係を表現します。

```mermaid
graph TB
    subgraph "外部ユーザー"
        EndUser[一般ユーザー]
        Admin[管理者]
    end

    subgraph "対象システム"
        BlogSystem[ブログシステム]
    end

    subgraph "外部システム"
        EmailService[メール送信サービス<br/>SendGrid]
        Analytics[分析サービス<br/>Google Analytics]
        CDN[CDN<br/>CloudFront]
        AuthProvider[認証プロバイダー<br/>Auth0]
    end

    EndUser -->|記事閲覧・投稿| BlogSystem
    Admin -->|コンテンツ管理| BlogSystem
    BlogSystem -->|メール通知| EmailService
    BlogSystem -->|アクセス解析| Analytics
    BlogSystem -->|静的コンテンツ配信| CDN
    BlogSystem -->|認証・認可| AuthProvider
```

### Level 2: コンテナ図

システム内の主要なアプリケーション・データストアを表現します。

```mermaid
graph TB
    subgraph "フロントエンド"
        SPA[シングルページアプリケーション<br/>React 18 + TypeScript<br/>Vite]
    end

    subgraph "バックエンド"
        APIServer[APIサーバー<br/>Next.js 14 API Routes<br/>Node.js 20]
        GraphQLServer[GraphQLサーバー<br/>Apollo Server 4<br/>Node.js 20]
        BackgroundWorker[バックグラウンドワーカー<br/>Bull Queue<br/>Node.js 20]
    end

    subgraph "データストア"
        PostgreSQL[(PostgreSQL 15<br/>RDS)]
        Redis[(Redis 7<br/>ElastiCache)]
        S3[(S3<br/>画像・ファイル)]
    end

    SPA -->|HTTPS/REST| APIServer
    SPA -->|WebSocket/GraphQL| GraphQLServer
    APIServer --> PostgreSQL
    GraphQLServer --> PostgreSQL
    GraphQLServer --> Redis
    BackgroundWorker --> PostgreSQL
    BackgroundWorker --> Redis
    APIServer --> S3
```

### Level 3: コンポーネント図

各コンテナ内の主要コンポーネントを表現します。

```mermaid
graph TB
    subgraph "APIサーバー（Next.js）"
        Router[ルーター<br/>Next.js App Router]
        AuthMiddleware[認証ミドルウェア]

        subgraph "コントローラー層"
            UserController[ユーザーコントローラー]
            PostController[投稿コントローラー]
            CommentController[コメントコントローラー]
        end

        subgraph "サービス層"
            UserService[ユーザーサービス]
            PostService[投稿サービス]
            CacheService[キャッシュサービス]
            StorageService[ストレージサービス]
        end

        subgraph "リポジトリ層"
            UserRepository[ユーザーリポジトリ]
            PostRepository[投稿リポジトリ]
        end
    end

    Router --> AuthMiddleware
    AuthMiddleware --> UserController
    AuthMiddleware --> PostController
    AuthMiddleware --> CommentController

    UserController --> UserService
    PostController --> PostService
    PostController --> CacheService

    UserService --> UserRepository
    PostService --> PostRepository
    PostService --> StorageService
```

## PlantUML活用

PlantUMLは、より高度なダイアグラム作成が可能なツールです。

### PlantUMLによるシーケンス図

```plantuml
@startuml
!define RECTANGLE class

actor "ユーザー" as User
participant "フロントエンド" as Frontend
participant "APIゲートウェイ" as Gateway
participant "認証サービス" as Auth
participant "ビジネスロジック" as Service
database "データベース" as DB

User -> Frontend: フォーム送信
activate Frontend

Frontend -> Gateway: POST /api/posts
activate Gateway

Gateway -> Auth: トークン検証
activate Auth
Auth --> Gateway: 検証成功
deactivate Auth

Gateway -> Service: 投稿作成リクエスト
activate Service

Service -> Service: バリデーション
Service -> DB: INSERT INTO posts
activate DB
DB --> Service: 投稿ID
deactivate DB

Service -> Service: 画像処理
Service --> Gateway: 投稿作成成功
deactivate Service

Gateway --> Frontend: 201 Created
deactivate Gateway

Frontend --> User: 成功通知表示
deactivate Frontend

@enduml
```

### PlantUMLによるコンポーネント図

```plantuml
@startuml
package "フロントエンド" {
  [React App] as ReactApp
  [Redux Store] as Redux
  [API Client] as APIClient
}

package "バックエンド" {
  [REST API] as REST
  [GraphQL API] as GraphQL
  [認証サービス] as AuthService
}

package "データ層" {
  database "PostgreSQL" as DB
  database "Redis" as Cache
}

ReactApp --> Redux
ReactApp --> APIClient
APIClient --> REST
APIClient --> GraphQL
REST --> AuthService
GraphQL --> AuthService
REST --> DB
GraphQL --> DB
GraphQL --> Cache

@enduml
```

## ADR（Architecture Decision Record）

ADRは、アーキテクチャに関する重要な意思決定を記録するドキュメント形式です。

### ADRテンプレート

```markdown
# ADR-001: GraphQLの採用

## ステータス
承認済み

## コンテキスト
現在のRESTful APIでは、以下の課題が存在します：
- オーバーフェッチング：必要以上のデータを取得
- アンダーフェッチング：複数リクエストが必要
- APIバージョン管理の複雑さ
- フロントエンド開発の柔軟性不足

## 決定事項
次期システムでは、GraphQLを採用します。
- Apollo Server 4を使用
- スキーマファーストアプローチ
- DataLoaderによるN+1問題対策
- GraphQL Code Generatorで型安全性確保

## 結果
期待される効果：
- クライアント側で必要なデータのみ取得可能
- 型安全性によるバグ削減
- 開発速度の向上

考慮すべき点：
- 学習コストの発生
- キャッシュ戦略の複雑化
- クエリの複雑性管理

## 代替案
1. RESTful API継続
   - メリット: 既存ナレッジ活用
   - デメリット: 上記課題の継続

2. gRPC採用
   - メリット: 高性能
   - デメリット: ブラウザ対応の複雑さ

## 参考文献
- [GraphQL公式ドキュメント](https://graphql.org/)
- [Apollo Server Documentation](https://www.apollographql.com/docs/apollo-server/)
```

### ADRファイル構成例

```
docs/
└── architecture/
    ├── decisions/
    │   ├── 0001-use-graphql.md
    │   ├── 0002-adopt-microservices.md
    │   ├── 0003-database-selection.md
    │   └── 0004-cache-strategy.md
    └── diagrams/
        ├── system-context.mmd
        ├── container-diagram.mmd
        └── component-diagram.mmd
```

## ドキュメントの自動生成

### TypeDocによるAPI自動ドキュメント化

TypeScriptプロジェクトでは、TypeDocを使用してコードから自動的にドキュメントを生成できます。

```typescript
/**
 * ユーザー情報を管理するサービスクラス
 *
 * @remarks
 * このクラスは、ユーザーの作成、更新、削除、検索機能を提供します。
 * すべてのメソッドは認証済みユーザーのみが使用できます。
 *
 * @example
 * ```typescript
 * const userService = new UserService(userRepository);
 * const user = await userService.findById('user-123');
 * ```
 */
export class UserService {
  /**
   * ユーザーサービスのコンストラクタ
   *
   * @param userRepository - ユーザーデータアクセス用リポジトリ
   */
  constructor(private readonly userRepository: UserRepository) {}

  /**
   * IDでユーザーを検索
   *
   * @param id - ユーザーID
   * @returns ユーザーオブジェクト、見つからない場合はnull
   * @throws {ValidationError} IDが不正な形式の場合
   * @throws {DatabaseError} データベースアクセスエラー
   *
   * @example
   * ```typescript
   * const user = await userService.findById('user-123');
   * if (user) {
   *   console.log(user.name);
   * }
   * ```
   */
  async findById(id: string): Promise<User | null> {
    // 実装
  }
}
```

### JSDocによるドキュメント化

JavaScriptプロジェクトでは、JSDocを使用します。

```javascript
/**
 * 投稿を管理するサービスクラス
 * @class
 */
class PostService {
  /**
   * 新規投稿を作成
   * @param {Object} data - 投稿データ
   * @param {string} data.title - 投稿タイトル
   * @param {string} data.content - 投稿本文
   * @param {string} data.userId - 著者ID
   * @returns {Promise<Post>} 作成された投稿オブジェクト
   * @throws {ValidationError} バリデーションエラー
   */
  async createPost({ title, content, userId }) {
    // 実装
  }
}
```

## Draw.io / diagrams.net活用

Draw.ioは、ブラウザベースのダイアグラム作成ツールです。VSCode拡張機能を使用すれば、エディタ内で直接編集できます。

### 推奨される活用シーン

1. **インフラ構成図**: AWS、GCP、Azureのアイコンライブラリを活用
2. **ネットワーク図**: ファイアウォール、ロードバランサーなどの配置
3. **デプロイメント図**: CI/CDパイプライン、環境構成
4. **ワイヤーフレーム**: UI/UXの初期設計

### VSCode拡張機能設定

```json
{
  "hediet.vscode-drawio.theme": "atlas",
  "hediet.vscode-drawio.customFonts": ["Noto Sans JP"],
  "hediet.vscode-drawio.exportFormats": [
    {
      "format": "png",
      "scale": 2
    },
    {
      "format": "svg"
    }
  ]
}
```

## アーキテクチャドキュメントのベストプラクティス

### 1. バージョン管理との統合

すべてのアーキテクチャ図は、コードと同じリポジトリで管理することを推奨します。

```
project/
├── docs/
│   ├── architecture/
│   │   ├── README.md
│   │   ├── diagrams/
│   │   │   ├── system-context.mmd
│   │   │   ├── container-diagram.mmd
│   │   │   └── sequence-diagrams/
│   │   │       ├── user-authentication.mmd
│   │   │       └── post-creation.mmd
│   │   └── decisions/
│   │       └── adr/
│   └── api/
└── src/
```

### 2. 定期的な更新プロセス

アーキテクチャ図は、システム変更時に必ず更新します。

```markdown
## アーキテクチャドキュメント更新チェックリスト

- [ ] 新規サービス追加時、システムコンテキスト図を更新
- [ ] データベーススキーマ変更時、ER図を更新
- [ ] API変更時、シーケンス図を更新
- [ ] 重要な技術選定時、ADRを作成
- [ ] 四半期ごとに全体レビュー実施
```

### 3. レイヤー別のドキュメント化

システムを複数の視点で文書化することを推奨します。

```mermaid
graph LR
    A[ビジネス視点] --> B[論理アーキテクチャ]
    B --> C[物理アーキテクチャ]
    C --> D[デプロイメント構成]

    style A fill:#e1f5ff
    style B fill:#fff4e1
    style C fill:#ffe1f5
    style D fill:#e1ffe1
```

## チェックリスト

アーキテクチャドキュメント作成時の品質チェック項目：

### 図の品質

- [ ] システム全体を俯瞰できるコンテキスト図が存在する
- [ ] 主要なデータフローがシーケンス図で表現されている
- [ ] データベーススキーマがER図で文書化されている
- [ ] 各コンポーネントの責務が明確に定義されている
- [ ] 図の凡例・注釈が適切に記載されている

### ドキュメントの保守性

- [ ] すべての図がテキストベース（Mermaid、PlantUML等）で管理されている
- [ ] バージョン管理システムで履歴が追跡可能
- [ ] 更新日時と更新者が記録されている
- [ ] 図とコードの整合性が保たれている
- [ ] CI/CDで図の自動生成・検証が実施されている

### ADRの品質

- [ ] 重要な技術選定がADRとして記録されている
- [ ] ADRに決定の背景・理由が明確に記載されている
- [ ] 代替案とその評価が含まれている
- [ ] 決定による影響範囲が記載されている
- [ ] 参考文献が適切に引用されている

### アクセシビリティ

- [ ] 図の代替テキストが提供されている
- [ ] カラーパレットが色覚多様性を考慮している
- [ ] 図の解像度が十分高い（300dpi以上推奨）
- [ ] テキストサイズが読みやすい（最小10pt以上）
- [ ] 複雑な図には補足説明が添付されている

## まとめ

アーキテクチャ図とドキュメントは、システムの設計思想を共有し、長期的な保守性を確保するための重要な資産です。Mermaid、PlantUML、Draw.ioなどのツールを適切に使い分け、バージョン管理システムと統合することで、常に最新の状態を維持できます。

重要な技術選定については、ADRとして明確に記録し、将来の意思決定の参考にすることを推奨します。また、TypeDocやJSDocを活用した自動ドキュメント生成により、コードとドキュメントの乖離を防ぐことができます。

## 参考文献

### 公式ドキュメント

- [Mermaid Documentation](https://mermaid.js.org/) - Mermaid記法の公式ドキュメント
- [PlantUML公式サイト](https://plantuml.com/) - PlantUMLの公式ドキュメント
- [C4 Model](https://c4model.com/) - C4モデルの公式サイト
- [TypeDoc公式ドキュメント](https://typedoc.org/) - TypeScript自動ドキュメント生成ツール
- [JSDoc公式ドキュメント](https://jsdoc.app/) - JavaScript自動ドキュメント生成ツール
- [diagrams.net](https://www.diagrams.net/) - Draw.io公式サイト

### 技術記事・ベストプラクティス

- [Architecture Decision Records (ADR)](https://adr.github.io/) - ADRの標準仕様
- [Documenting Architecture Decisions - Michael Nygard](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions) - ADRの提唱者による解説
- [The Art of Visualising Software Architecture - Simon Brown](https://www.infoq.com/articles/visualising-software-architecture/) - C4モデル提唱者による解説

## 次のステップ

最終章では、アクセシビリティ自動化について解説します。
