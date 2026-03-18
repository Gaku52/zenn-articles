---
title: "Axum Web開発"
---

## Axum の概要と設計思想

Axum は Tokio チームが開発する Rust の Web フレームワークです。内部で hyper を HTTP エンジンとして使用し、ミドルウェア層には Tower エコシステムを全面的に採用しています。この設計により、Tower 互換のあらゆるミドルウェアをそのまま利用できます。

Axum の設計思想を整理すると、以下の4つが核となります。

| 設計原則 | 説明 |
|---|---|
| Tower 統合 | ミドルウェアは全て Tower Service/Layer として実装されます |
| 型安全 | Extractor による型レベルでのリクエスト解析が行われます |
| Macro-free | derive マクロに頼らず、コンパイラが型推論でルーティングを検証します |
| hyper ベース | 内部で hyper を使用し、高いパフォーマンスを実現しています |

Cargo.toml の依存関係は次のように記述します。

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
tower-http = { version = "0.6", features = ["cors", "trace", "compression-gzip"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

## ルーティングとハンドラ

Axum のルーティングは `Router` 構造体を中心に構築します。各ルートにはパスとHTTPメソッドに対応するハンドラ関数を紐づけます。

```rust
use axum::{routing::{get, post, put, delete}, Router};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/users", get(list_users).post(create_user))
        .route("/users/{id}", get(get_user).put(update_user).delete(delete_user));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

ルーターはネスティングやマージによって階層的に構成できます。API のバージョニングにはネスティングが便利です。

```rust
fn api_v1() -> Router {
    let users = Router::new()
        .route("/", get(list_users).post(create_user))
        .route("/{id}", get(get_user));
    let posts = Router::new()
        .route("/", get(list_posts).post(create_post));

    Router::new()
        .nest("/users", users)
        .nest("/posts", posts)
}

fn build_app() -> Router {
    Router::new()
        .nest("/api/v1", api_v1())
        .route("/health", get(|| async { "OK" }))
}
```

## パスパラメータとクエリパラメータの抽出

Axum は **Extractor** パターンでリクエストの各要素を型安全に取り出します。ハンドラの引数に Extractor を記述するだけで、フレームワークが自動的にデシリアライズを行います。

```rust
use axum::extract::{Path, Query};
use serde::Deserialize;

// パスパラメータ: /users/42
async fn get_user(Path(id): Path<u64>) -> String {
    format!("User ID: {}", id)
}

// 複数パスパラメータ: /users/1/posts/5
async fn get_user_post(
    Path((user_id, post_id)): Path<(u64, u64)>,
) -> String {
    format!("User {} の Post {}", user_id, post_id)
}

// クエリパラメータ: /users?page=2&limit=20
#[derive(Deserialize)]
struct ListParams {
    page: Option<u32>,
    limit: Option<u32>,
}

async fn list_users(Query(params): Query<ListParams>) -> String {
    let page = params.page.unwrap_or(1);
    let limit = params.limit.unwrap_or(10);
    format!("Page: {}, Limit: {}", page, limit)
}
```

構造体でパスパラメータを受け取ることも可能です。フィールド名がパスのプレースホルダ名と一致する必要があります。

```rust
#[derive(Deserialize)]
struct PostPath { user_id: u64, post_id: u64 }

async fn get_post(Path(params): Path<PostPath>) -> String {
    format!("User {} / Post {}", params.user_id, params.post_id)
}
// ルート: .route("/users/{user_id}/posts/{post_id}", get(get_post))
```

## JSON リクエスト/レスポンス（serde 連携）

`Json<T>` Extractor を使うと、リクエストボディの JSON を自動でデシリアライズできます。レスポンスも `Json<T>` で返すだけで `Content-Type: application/json` が設定されます。

```rust
use axum::{extract::Json, http::StatusCode};
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct CreateUser { name: String, email: String }

#[derive(Serialize)]
struct User { id: u64, name: String, email: String }

async fn create_user(
    Json(input): Json<CreateUser>,
) -> (StatusCode, Json<User>) {
    let user = User { id: 1, name: input.name, email: input.email };
    (StatusCode::CREATED, Json(user))
}
```

:::message
**Extractor の順序に注意してください。** `Json` のようにリクエストボディを消費する Extractor は、ハンドラ引数の **最後** に置く必要があります。`State`、`Path`、`Query` などボディを消費しない Extractor を先に記述しましょう。
:::

## State によるアプリケーション状態の共有

データベース接続プールや設定値など、ハンドラ間で共有したい状態は `State` Extractor で受け取ります。`with_state()` でルーターに状態を注入します。

```rust
use axum::extract::State;
use std::sync::Arc;
use tokio::sync::RwLock;

type SharedState = Arc<RwLock<Vec<User>>>;

async fn list_users(State(state): State<SharedState>) -> Json<Vec<User>> {
    let users = state.read().await;
    Json(users.clone())
}

async fn create_user(
    State(state): State<SharedState>,
    Json(input): Json<CreateUser>,
) -> (StatusCode, Json<User>) {
    let mut users = state.write().await;
    let user = User { id: users.len() as u64 + 1, name: input.name, email: input.email };
    users.push(user.clone());
    (StatusCode::CREATED, Json(user))
}

#[tokio::main]
async fn main() {
    let state: SharedState = Arc::new(RwLock::new(Vec::new()));
    let app = Router::new()
        .route("/users", get(list_users).post(create_user))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

実際のアプリケーションでは、DB プールや設定を構造体にまとめるのが一般的です。`FromRef` トレイトを実装すると、個別のフィールドをハンドラで直接抽出できます。

```rust
use axum::extract::FromRef;
use sqlx::PgPool;

#[derive(Clone)]
struct AppState { db: PgPool, config: AppConfig }

#[derive(Clone)]
struct AppConfig { jwt_secret: String }

impl FromRef<AppState> for PgPool {
    fn from_ref(state: &AppState) -> Self { state.db.clone() }
}

// ハンドラで PgPool だけを直接取得できる
async fn handler(State(db): State<PgPool>) -> &'static str { "OK" }
```

## ミドルウェア（ロギング、認証、CORS）

Axum は Tower のレイヤーシステムを採用しているため、`tower-http` クレートが提供する豊富なミドルウェアをそのまま利用できます。

```rust
use tower_http::{
    cors::{CorsLayer, Any}, trace::TraceLayer,
    compression::CompressionLayer, timeout::TimeoutLayer,
};
use std::time::Duration;

let app = Router::new()
    .route("/api/data", get(handler))
    // ミドルウェアは下から上に実行される（最後に追加したものが最初に実行）
    .layer(CompressionLayer::new())
    .layer(TimeoutLayer::new(Duration::from_secs(30)))
    .layer(CorsLayer::new().allow_origin(Any).allow_methods(Any).allow_headers(Any))
    .layer(TraceLayer::new_for_http());
```

カスタムミドルウェアは `axum::middleware::from_fn` で作成できます。以下はリクエスト処理時間を計測する例です。

```rust
use axum::{extract::Request, middleware::Next, response::Response};
use std::time::Instant;

async fn timing_middleware(request: Request, next: Next) -> Response {
    let method = request.method().clone();
    let uri = request.uri().clone();
    let start = Instant::now();
    let response = next.run(request).await;
    tracing::info!("{} {} -> {} ({:?})", method, uri, response.status(), start.elapsed());
    response
}

// 適用
let app = Router::new()
    .route("/api/data", get(handler))
    .layer(axum::middleware::from_fn(timing_middleware));
```

認証ミドルウェアをルート単位で適用することも可能です。保護されたルートと公開ルートを分離できます。

```rust
let protected = Router::new()
    .route("/profile", get(get_profile))
    .route("/settings", get(get_settings))
    .layer(axum::middleware::from_fn(auth_middleware));

let public = Router::new()
    .route("/login", post(login))
    .route("/register", post(register));

let app = Router::new().merge(public).merge(protected);
```

## エラーハンドリング（IntoResponse）

Axum では `IntoResponse` トレイトを実装した型をハンドラの戻り値として使えます。アプリケーション共通のエラー型を定義し、`IntoResponse` を実装するのが定番パターンです。

```rust
use axum::{http::StatusCode, response::{IntoResponse, Response}, Json};

#[derive(Debug)]
enum AppError {
    NotFound(String),
    BadRequest(String),
    Internal(anyhow::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match &self {
            AppError::NotFound(msg) => (StatusCode::NOT_FOUND, msg.clone()),
            AppError::BadRequest(msg) => (StatusCode::BAD_REQUEST, msg.clone()),
            AppError::Internal(e) => {
                tracing::error!("内部エラー: {:?}", e);
                (StatusCode::INTERNAL_SERVER_ERROR, "内部サーバーエラー".into())
            }
        };
        let body = serde_json::json!({
            "error": { "status": status.as_u16(), "message": message }
        });
        (status, Json(body)).into_response()
    }
}
```

`From` トレイトを実装しておくと、`?` 演算子で自動的にエラー変換が行われます。

```rust
impl From<sqlx::Error> for AppError {
    fn from(e: sqlx::Error) -> Self {
        match e {
            sqlx::Error::RowNotFound => AppError::NotFound("リソースが見つかりません".into()),
            _ => AppError::Internal(e.into()),
        }
    }
}

// ハンドラでは ? でエラーを伝播するだけ
async fn get_user(
    State(db): State<PgPool>, Path(id): Path<i64>,
) -> Result<Json<User>, AppError> {
    let user = sqlx::query_as::<_, User>("SELECT * FROM users WHERE id = $1")
        .bind(id).fetch_one(&db).await?;  // sqlx::Error -> AppError に自動変換
    Ok(Json(user))
}
```

## レイヤードアーキテクチャの構成例

実際のプロダクションコードでは、ハンドラ・サービス・リポジトリの3層構成がよく採用されます。責務を分離することでテスタビリティが向上します。

```
src/
├── main.rs          # エントリポイント、サーバー起動
├── routes/          # ルーター定義
├── handlers/        # リクエスト/レスポンスの変換
├── services/        # ビジネスロジック
├── repositories/    # データアクセス
├── models/          # ドメインモデル
├── errors.rs        # 統一エラー型
└── config.rs        # 設定管理
```

各層の依存関係は「ハンドラ -> サービス -> リポジトリ」の一方向です。サービス層をトレイトとして定義すれば、テスト時にモックへ差し替えられます。

```rust
// repositories/user_repo.rs
pub struct UserRepository { db: PgPool }
impl UserRepository {
    pub async fn find_by_id(&self, id: i64) -> Result<User, sqlx::Error> {
        sqlx::query_as("SELECT * FROM users WHERE id = $1")
            .bind(id).fetch_one(&self.db).await
    }
}

// services/user_service.rs
pub struct UserService { repo: UserRepository }
impl UserService {
    pub async fn get_user(&self, id: i64) -> Result<User, AppError> {
        self.repo.find_by_id(id).await.map_err(Into::into)
    }
}

// handlers/users.rs
pub async fn get_user(
    State(svc): State<Arc<UserService>>, Path(id): Path<i64>,
) -> Result<Json<User>, AppError> {
    let user = svc.get_user(id).await?;
    Ok(Json(user))
}
```

## Express/Gin/Echo との比較

他言語の代表的な Web フレームワークと比較すると、Axum の特徴がより明確になります。

| 観点 | Axum (Rust) | Express (Node.js) | Gin (Go) |
|---|---|---|---|
| 型安全性 | コンパイル時に保証 | 実行時チェック | 部分的 |
| パフォーマンス | 非常に高い | 中程度 | 高い |
| メモリ安全性 | 所有権システム | GC | GC |
| ミドルウェア | Tower Layer | use() チェーン | HandlerFunc |
| エラーハンドリング | Result + IntoResponse | next(err) | c.AbortWithStatus |
| 非同期モデル | async/await (Tokio) | イベントループ | goroutine |

Axum が他と大きく異なるのは、**コンパイル時の型安全性**です。Express では `req.params.id` の型は実行時まで不明ですが、Axum では `Path(id): Path<u64>` と書いた時点で、パスパラメータが `u64` に変換できなければコンパイルエラーになります。

パフォーマンス面では、Rust のゼロコスト抽象化により GC のオーバーヘッドがなく、Go の Gin/Echo と比較しても高いスループットを実現します。一方で学習コストは高めで、所有権・ライフタイム・トレイトの理解が前提となるため、チームの習熟度を考慮して選定する必要があります。

## まとめ

| 項目 | 要点 |
|---|---|
| 設計思想 | Tower ベースのミドルウェア統合と型安全な Extractor パターンが基盤です |
| ルーティング | `route()` でパス+メソッド、`nest()` でグループ化、`merge()` で結合します |
| パラメータ抽出 | `Path<T>`, `Query<T>` で型安全にデシリアライズされます |
| JSON 処理 | `Json<T>` でリクエスト/レスポンスの両方に対応し、serde が変換を担います |
| State | `with_state()` で共有状態を注入し、`FromRef` で個別フィールドの抽出も可能です |
| ミドルウェア | `tower-http` の既存レイヤーに加え、`from_fn` でカスタム実装が可能です |
| エラーハンドリング | `IntoResponse` を実装した統一エラー型と `From` 変換で `?` 演算子が使えます |
| アーキテクチャ | ハンドラ/サービス/リポジトリの3層構成がテスタビリティを高めます |
| 他言語比較 | コンパイル時型安全性とゼロコスト抽象化が最大の差別化要因です |

## やってみよう！

以下の演習で Axum の理解を深めましょう。

**演習1: 基本CRUD API**
Todo アプリの REST API を構築してください。`Vec<Todo>` をインメモリで保持し、一覧取得（GET）、作成（POST）、更新（PUT）、削除（DELETE）の4エンドポイントを実装します。クエリパラメータで `?completed=true` のフィルタリングにも対応させましょう。

**演習2: ミドルウェアの追加**
演習1のAPIに以下のミドルウェアを追加してください。
- リクエスト処理時間を `tracing::info!` で出力するタイミングミドルウェア
- `X-API-Key` ヘッダーを検証する認証ミドルウェア（認証不要の `/health` エンドポイントも用意する）

**演習3: エラーハンドリング**
`AppError` 型を定義し、バリデーションエラー（422）、リソース未発見（404）、内部エラー（500）を JSON 形式で返す仕組みを実装してください。`From<serde_json::Error>` も実装し、不正な JSON リクエストにも対応させましょう。
