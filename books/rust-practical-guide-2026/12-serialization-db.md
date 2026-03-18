---
title: "シリアライズとデータベース"
---

## serde の基本 ── Serialize と Deserialize

Rust におけるデータのシリアライゼーションは、事実上の標準である **serde** クレートが担います。`Serialize`（直列化）と `Deserialize`（逆直列化）の 2 つのトレイトを中心に設計されており、derive マクロを付けるだけで JSON・TOML・YAML など多数のフォーマットに一度に対応できます。

```toml
# Cargo.toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
toml = "0.8"
serde_yaml = "0.9"
```

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct User {
    id: u64,
    name: String,
    email: String,
    #[serde(default)]
    is_active: bool,
}
```

`#[derive(Serialize, Deserialize)]` を付与すると、serde のマクロが内部の **Data Model**（29 種の型）へのマッピングコードを自動生成します。1 回の derive で全フォーマットに変換できるのはこの仕組みのおかげです。

---

## JSON / TOML / YAML の読み書き

### JSON ── API 通信の標準

```rust
// シリアライズ（Rust -> JSON）
let json = serde_json::to_string_pretty(&user)?;

// デシリアライズ（JSON -> Rust）
let parsed: User = serde_json::from_str(&json)?;

// json! マクロで動的に構築
let response = serde_json::json!({
    "status": "ok",
    "data": { "id": 1, "name": "Alice" }
});
```

### TOML ── 設定ファイルに最適

コメントが書ける点が JSON との大きな違いです。Rust の `Cargo.toml` にも採用されています。

```rust
let toml_str = r#"
[server]
host = "0.0.0.0"
port = 3000
"#;

#[derive(Debug, Deserialize)]
struct Config { server: ServerConfig }

#[derive(Debug, Deserialize)]
struct ServerConfig { host: String, port: u16 }

let config: Config = toml::from_str(toml_str)?;
```

### YAML ── CI/CD で頻出

```rust
let yaml_str = "name: web-app\nreplicas: 3\n";

#[derive(Debug, Deserialize)]
struct Deployment { name: String, replicas: u32 }

let deploy: Deployment = serde_yaml::from_str(yaml_str)?;
```

拡張子でフォーマットを自動判別するローダーも簡潔に書けます。

```rust
fn load_config<T: serde::de::DeserializeOwned>(path: &std::path::Path) -> anyhow::Result<T> {
    let s = std::fs::read_to_string(path)?;
    match path.extension().and_then(|e| e.to_str()) {
        Some("json") => Ok(serde_json::from_str(&s)?),
        Some("toml") => Ok(toml::from_str(&s)?),
        Some("yaml" | "yml") => Ok(serde_yaml::from_str(&s)?),
        _ => anyhow::bail!("Unsupported format"),
    }
}
```

---

## カスタムシリアライズ ── #[serde(...)] アトリビュート

serde の真価は属性マクロによる柔軟なカスタマイズにあります。

| 属性 | 効果 |
|---|---|
| `rename = "name"` | フィールド名を変更します |
| `rename_all = "camelCase"` | 全フィールドの命名規則を一括変更します |
| `default` / `default = "fn"` | 値が無い場合にデフォルト値を使います |
| `skip_serializing_if = "..."` | 条件付きでフィールドを省略します |
| `skip` / `skip_serializing` | シリアライズ対象から除外します |
| `flatten` | ネストを解消してフラットに展開します |
| `tag = "type"` | 列挙体の内部タグ形式を指定します |
| `deny_unknown_fields` | 未知フィールドをエラーにします |

```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
struct ApiUser {
    user_id: u64,                               // -> "userId"
    display_name: String,                        // -> "displayName"
    #[serde(skip_serializing_if = "Option::is_none")]
    bio: Option<String>,
    #[serde(skip_serializing)]
    password_hash: String,                       // レスポンスに含めない
}
```

列挙体では内部タグ形式が REST API で頻出します。

```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
enum Event {
    #[serde(rename = "user.created")]
    UserCreated { id: u64, name: String },
    #[serde(rename = "user.deleted")]
    UserDeleted { id: u64 },
}
// JSON: {"type": "user.created", "id": 1, "name": "Alice"}
```

---

## SQLx によるデータベースアクセス

**SQLx** は生の SQL を書きつつコンパイル時に型安全性を検証できる非同期クエリライブラリです。PostgreSQL・MySQL・SQLite をサポートしています。

```toml
[dependencies]
sqlx = { version = "0.8", features = [
    "runtime-tokio", "tls-rustls", "postgres",
    "macros", "migrate", "chrono", "uuid",
] }
tokio = { version = "1", features = ["full"] }
```

### モデル定義と CRUD

```rust
use sqlx::{PgPool, FromRow};
use uuid::Uuid;
use chrono::{DateTime, Utc};

#[derive(Debug, FromRow, Serialize)]
struct User { id: Uuid, name: String, email: String, created_at: DateTime<Utc> }

// CREATE
async fn create_user(pool: &PgPool, name: &str, email: &str) -> Result<User, sqlx::Error> {
    sqlx::query_as!(User,
        r#"INSERT INTO users (id, name, email, created_at)
           VALUES ($1, $2, $3, NOW())
           RETURNING id, name, email, created_at"#,
        Uuid::new_v4(), name, email,
    ).fetch_one(pool).await
}

// READ（1 件）
async fn find_user(pool: &PgPool, id: Uuid) -> Result<Option<User>, sqlx::Error> {
    sqlx::query_as!(User,
        "SELECT id, name, email, created_at FROM users WHERE id = $1", id,
    ).fetch_optional(pool).await
}

// UPDATE
async fn update_email(pool: &PgPool, id: Uuid, email: &str) -> Result<bool, sqlx::Error> {
    let r = sqlx::query!("UPDATE users SET email = $1 WHERE id = $2", email, id)
        .execute(pool).await?;
    Ok(r.rows_affected() > 0)
}

// DELETE
async fn delete_user(pool: &PgPool, id: Uuid) -> Result<bool, sqlx::Error> {
    let r = sqlx::query!("DELETE FROM users WHERE id = $1", id)
        .execute(pool).await?;
    Ok(r.rows_affected() > 0)
}
```

---

## コンパイル時 SQL チェック

`query!` / `query_as!` マクロはビルド時に `DATABASE_URL` で指定された DB へ接続し、SQL の構文やカラム型を検証します。カラム名の誤りや型の不一致はコンパイルエラーになるため、実行時エラーを大幅に削減できます。

CI 環境など DB に接続できない場合は**オフラインモード**を使います。

```bash
cargo sqlx prepare        # .sqlx/ にメタデータを生成
# .sqlx/ を Git にコミット -> CI で DB 接続なしにビルド可能
```

---

## マイグレーション管理

```bash
cargo install sqlx-cli --no-default-features --features postgres
sqlx migrate add create_users_table   # ファイル作成
sqlx migrate run                       # 実行
sqlx migrate info                      # 状態確認
```

```sql
-- migrations/20260101000000_create_users_table.sql
CREATE TABLE users (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name       VARCHAR(255) NOT NULL,
    email      VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_users_email ON users (email);
```

アプリ起動時にプログラムから実行することもできます。

```rust
sqlx::migrate!("./migrations").run(&pool).await?;
```

---

## コネクションプール

DB 接続はコストが高いため、コネクションプールで再利用するのが鉄則です。

```rust
use sqlx::postgres::PgPoolOptions;

let pool = PgPoolOptions::new()
    .max_connections(20)
    .min_connections(5)
    .acquire_timeout(std::time::Duration::from_secs(5))
    .idle_timeout(std::time::Duration::from_secs(600))
    .max_lifetime(std::time::Duration::from_secs(1800))
    .connect(&database_url)
    .await?;
```

プールサイズの目安: `(CPU コア数 x 2) + 有効ディスクスピンドル数`。4 コア + SSD なら約 10 が適切です。PostgreSQL のデフォルト最大接続は 100 なので、複数インスタンスがある場合は合計を考慮してください。

---

## 実務パターン：REST API + DB の CRUD 実装

Axum + SQLx + serde を統合した実践パターンです。

```rust
use axum::{extract::{Path, State, Json}, http::StatusCode, routing::{get, post}, Router};

#[derive(Deserialize)]
struct CreateUserRequest { name: String, email: String }

#[derive(Serialize)]
#[serde(rename_all = "camelCase")]
struct UserResponse { id: Uuid, name: String, email: String, created_at: DateTime<Utc> }

async fn create_handler(
    State(pool): State<PgPool>,
    Json(req): Json<CreateUserRequest>,
) -> Result<Json<UserResponse>, StatusCode> {
    let user = create_user(&pool, &req.name, &req.email)
        .await.map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    Ok(Json(UserResponse {
        id: user.id, name: user.name, email: user.email, created_at: user.created_at,
    }))
}

async fn get_handler(
    State(pool): State<PgPool>, Path(id): Path<Uuid>,
) -> Result<Json<UserResponse>, StatusCode> {
    let user = find_user(&pool, id)
        .await.map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?
        .ok_or(StatusCode::NOT_FOUND)?;
    Ok(Json(UserResponse {
        id: user.id, name: user.name, email: user.email, created_at: user.created_at,
    }))
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let pool = PgPoolOptions::new()
        .max_connections(20)
        .connect(&std::env::var("DATABASE_URL")?)
        .await?;
    sqlx::migrate!("./migrations").run(&pool).await?;

    let app = Router::new()
        .route("/users", post(create_handler))
        .route("/users/{id}", get(get_handler))
        .with_state(pool);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
    axum::serve(listener, app).await?;
    Ok(())
}
```

複数テーブルへの書き込みには必ずトランザクションを使います。`Transaction` は Drop 時に自動ロールバックされるため安全です。

```rust
async fn create_user_with_profile(pool: &PgPool, name: &str, email: &str, bio: &str)
    -> Result<User, sqlx::Error>
{
    let mut tx = pool.begin().await?;
    let user = sqlx::query_as!(User,
        r#"INSERT INTO users (id, name, email, created_at)
           VALUES ($1, $2, $3, NOW()) RETURNING id, name, email, created_at"#,
        Uuid::new_v4(), name, email,
    ).fetch_one(&mut *tx).await?;

    sqlx::query!("INSERT INTO profiles (user_id, bio) VALUES ($1, $2)", user.id, bio)
        .execute(&mut *tx).await?;

    tx.commit().await?;
    Ok(user)
}
```

---

## まとめ

| 項目 | 要点 |
|---|---|
| serde 基本 | `#[derive(Serialize, Deserialize)]` で自動実装。1 回で全フォーマット対応 |
| JSON | `serde_json`。API 通信の標準。`json!` マクロで動的構築も可能 |
| TOML | `toml` クレート。設定ファイル向け。コメント対応 |
| YAML | `serde_yaml`。CI/CD 設定で頻出 |
| 属性マクロ | `rename_all`, `default`, `skip`, `flatten` 等で柔軟にカスタマイズ |
| SQLx | 生 SQL + コンパイル時検証。非同期ネイティブ |
| コンパイル時チェック | `query!` マクロがビルド時に DB 接続して SQL を検証 |
| マイグレーション | `sqlx-cli` で作成・実行。アプリ起動時の自動実行も可能 |
| コネクションプール | `PgPoolOptions` でサイズ・タイムアウトを制御 |
| REST + DB | Axum の `State` でプール共有、serde でリクエスト/レスポンス変換 |
| トランザクション | `pool.begin()` で開始。Drop 時に自動ロールバック |

---

## やってみよう！

1. **設定ファイル変換ツール** ── JSON / TOML / YAML を相互変換する CLI ツールを `clap` で作ってみましょう。`serde_json::Value` を中間表現に使うと、任意のフォーマット間変換が簡潔に書けます。

2. **serde アトリビュート実験** ── `rename_all`, `skip_serializing_if`, `flatten`, `tag` を組み合わせた構造体を定義し、`serde_json::to_string_pretty` で出力を確認してみましょう。JSON の構造がどう変わるかを体感できます。

3. **SQLx CRUD アプリ** ── PostgreSQL または SQLite で Axum + SQLx の REST API を構築しましょう。意図的にカラム名を間違えて、コンパイル時エラーが出ることを体感してください。

4. **トランザクション演習** ── `accounts` と `transfers` テーブルを用意し、送金処理をトランザクションで実装しましょう。途中で意図的にエラーを発生させ、ロールバックが正しく動作することを確認します。
