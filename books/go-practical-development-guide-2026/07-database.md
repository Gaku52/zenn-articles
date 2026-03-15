---
title: "データベース"
---

# Chapter 07: データベース

> database/sql, sqlx, GORM — データ永続化の基本

Goは標準ライブラリ `database/sql` でDB接続の基盤を提供し、`sqlx` や `GORM` で生産性をさらに高められます。この章では接続プール管理からCRUD、トランザクション、マイグレーションまで、実務に必要な知識を一通りカバーします。

---

## database/sql の基本

`database/sql` はGoの標準DB抽象レイヤーです。ドライバを差し替えるだけでPostgreSQL、MySQL、SQLiteに対応できます。

```go
package main

import (
    "context"
    "database/sql"
    "log"
    "time"

    _ "github.com/lib/pq" // PostgreSQLドライバ（init()でドライバ登録）
)

func main() {
    // sql.Open はプールを初期化するだけ。実際の接続はまだ行わない
    db, err := sql.Open("postgres",
        "postgres://user:pass@localhost/mydb?sslmode=disable")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // 本番では接続プールを必ず設定する
    db.SetMaxOpenConns(25)                 // 最大同時接続数
    db.SetMaxIdleConns(5)                  // アイドル接続の保持数
    db.SetConnMaxLifetime(5 * time.Minute) // 接続の最大生存時間

    // Ping で実際の接続を確認
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    if err := db.PingContext(ctx); err != nil {
        log.Fatal("DB接続失敗:", err)
    }
}
```

---

## CRUD操作

基本メソッドは3つです。

| メソッド | 用途 | 戻り値 |
|---------|------|--------|
| `QueryRowContext` | 1行取得 | `*sql.Row` |
| `QueryContext` | 複数行取得 | `*sql.Rows` |
| `ExecContext` | INSERT/UPDATE/DELETE | `sql.Result` |

```go
type User struct {
    ID    int64
    Name  string
    Email string
}

type UserRepo struct{ db *sql.DB }

// 1行取得
func (r *UserRepo) GetByID(ctx context.Context, id int64) (*User, error) {
    var u User
    err := r.db.QueryRowContext(ctx,
        "SELECT id, name, email FROM users WHERE id = $1", id,
    ).Scan(&u.ID, &u.Name, &u.Email)
    if errors.Is(err, sql.ErrNoRows) {
        return nil, ErrNotFound
    }
    return &u, err
}

// 複数行取得
func (r *UserRepo) List(ctx context.Context, limit, offset int) ([]User, error) {
    rows, err := r.db.QueryContext(ctx,
        "SELECT id, name, email FROM users ORDER BY id LIMIT $1 OFFSET $2",
        limit, offset)
    if err != nil {
        return nil, err
    }
    defer rows.Close() // 必ず閉じる（接続リーク防止）

    var users []User
    for rows.Next() {
        var u User
        if err := rows.Scan(&u.ID, &u.Name, &u.Email); err != nil {
            return nil, err
        }
        users = append(users, u)
    }
    return users, rows.Err() // 反復中のエラーも確認
}

// 挿入（RETURNING で生成されたIDを受け取る）
func (r *UserRepo) Create(ctx context.Context, u *User) error {
    return r.db.QueryRowContext(ctx,
        "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id",
        u.Name, u.Email,
    ).Scan(&u.ID)
}

// 更新
func (r *UserRepo) Update(ctx context.Context, u *User) error {
    result, err := r.db.ExecContext(ctx,
        "UPDATE users SET name = $1, email = $2 WHERE id = $3",
        u.Name, u.Email, u.ID)
    if err != nil {
        return err
    }
    n, _ := result.RowsAffected()
    if n == 0 {
        return ErrNotFound
    }
    return nil
}
```

**重要**: `rows` は必ず `defer rows.Close()` で閉じること。忘れると接続プールが枯渇します。

---

## トランザクション

`BeginTx` → 処理 → `Commit` / `Rollback` の流れです。ヘルパー関数を作ると安全に管理できます。

```go
func WithTransaction(ctx context.Context, db *sql.DB, fn func(tx *sql.Tx) error) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer func() {
        if p := recover(); p != nil {
            _ = tx.Rollback()
            panic(p)
        }
    }()

    if err := fn(tx); err != nil {
        _ = tx.Rollback()
        return err
    }
    return tx.Commit()
}

// 使用例：送金処理
func (s *AccountService) Transfer(ctx context.Context, from, to int64, amount float64) error {
    return WithTransaction(ctx, s.db, func(tx *sql.Tx) error {
        var balance float64
        err := tx.QueryRowContext(ctx,
            "SELECT balance FROM accounts WHERE id = $1 FOR UPDATE", from,
        ).Scan(&balance)
        if err != nil {
            return err
        }
        if balance < amount {
            return fmt.Errorf("残高不足: %.2f < %.2f", balance, amount)
        }
        _, err = tx.ExecContext(ctx,
            "UPDATE accounts SET balance = balance - $1 WHERE id = $2", amount, from)
        if err != nil {
            return err
        }
        _, err = tx.ExecContext(ctx,
            "UPDATE accounts SET balance = balance + $1 WHERE id = $2", amount, to)
        return err
    })
}
```

`FOR UPDATE` で行ロックを取得し、並行する送金との競合を防いでいます。

**分離レベル**: `ReadCommitted`（PostgreSQLデフォルト）で通常は十分。送金・在庫管理など厳密な整合性には `Serializable` を検討してください。

---

## sqlx と GORM

### sqlx -- 構造体マッピング付きの生SQL

`sqlx` は `database/sql` のラッパーで、`db` タグによる自動マッピングが最大の利点です。

```go
import "github.com/jmoiron/sqlx"

type User struct {
    ID    int64  `db:"id" json:"id"`
    Name  string `db:"name" json:"name"`
    Email string `db:"email" json:"email"`
}

db, _ := sqlx.Connect("postgres", dsn) // Open + Ping

// 1行取得 → 構造体に直接マッピング
var user User
db.GetContext(ctx, &user, "SELECT id, name, email FROM users WHERE id = $1", id)

// 複数行取得 → スライスに直接マッピング
var users []User
db.SelectContext(ctx, &users,
    "SELECT id, name, email FROM users ORDER BY id LIMIT $1 OFFSET $2", limit, offset)

// IN句の展開
query, args, _ := sqlx.In("SELECT * FROM users WHERE name IN (?)", names)
query = db.Rebind(query) // $1形式に変換
db.SelectContext(ctx, &users, query, args...)
```

### GORM -- フルORM

構造体定義からCRUDが即座にでき、リレーションやソフトデリートも組み込みで対応します。

```go
import (
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

type User struct {
    ID        uint           `gorm:"primarykey"`
    Name      string         `gorm:"size:100;not null"`
    Email     string         `gorm:"uniqueIndex;not null"`
    Orders    []Order        `gorm:"foreignKey:UserID"`
    DeletedAt gorm.DeletedAt `gorm:"index"` // ソフトデリート
}

db, _ := gorm.Open(postgres.Open(dsn), &gorm.Config{})

db.Create(&User{Name: "Gopher", Email: "go@example.com"})       // Create
db.Preload("Orders").First(&user, 1)                             // Read + Eager Load
db.Model(&User{}).Where("id = ?", 1).Update("name", "New Name") // Update
db.Delete(&User{}, 1)                                            // Soft Delete
```

### どれを選ぶ？

| 基準 | database/sql | sqlx | GORM |
|------|-------------|------|------|
| SQL記述 | 生SQL | 生SQL | メソッドチェーン |
| 構造体マッピング | 手動Scan | 自動（dbタグ） | 完全自動 |
| パフォーマンス | 最高 | 高 | 中 |
| 推奨場面 | 高性能・学習用 | **新規PJの第一選択** | CRUD中心・高速開発 |

迷ったら **sqlx** がバランスの良い選択です。

---

## マイグレーション

スキーマ変更はマイグレーションツールで管理します。**本番で `AutoMigrate` は使わないでください**（カラム削除されない、ロールバック不可）。

```bash
# golang-migrate のインストール
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# ファイル作成
migrate create -ext sql -dir migrations -seq create_users
```

```sql
-- migrations/000001_create_users.up.sql
CREATE TABLE IF NOT EXISTS users (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_users_email ON users(email);

-- migrations/000001_create_users.down.sql
DROP TABLE IF EXISTS users;
```

```bash
migrate -path ./migrations -database "$DB_URL" up      # 適用
migrate -path ./migrations -database "$DB_URL" down 1   # 1つ戻す
```

**マイグレーションのルール**: (1) 必ずdownファイルも作る (2) 1ファイル1変更 (3) `IF NOT EXISTS` で冪等に (4) Gitで管理しコードレビュー対象に

---

## 避けるべきアンチパターン

**rows.Close() の忘れ** -- `defer rows.Close()` を必ず書く。忘れると接続プールが枯渇する。

**SQLインジェクション** -- 文字列連結でSQLを組み立てない。必ずプレースホルダ（`$1`, `?`）を使う。

```go
// NG: 攻撃可能
query := fmt.Sprintf("SELECT * FROM users WHERE name = '%s'", name)
// OK: プレースホルダ
db.QueryContext(ctx, "SELECT * FROM users WHERE name = $1", name)
```

**Contextを渡さない** -- `Query` ではなく `QueryContext` を使う。Contextがないとリクエストキャンセル後もクエリが走り続ける。

**グローバル変数にDBを保持** -- 構造体のフィールドに `*sql.DB` を持たせ、依存注入する。テスト時にモック差し替えが可能になる。

---

## まとめ

| 概念 | 要点 |
|------|------|
| database/sql | Go標準のDB抽象レイヤー。ドライバ差し替えでPostgreSQL/MySQL/SQLiteに対応 |
| 接続プール設定 | `SetMaxOpenConns` / `SetMaxIdleConns` / `SetConnMaxLifetime` を本番では必ず設定する |
| CRUD操作 | `QueryRowContext`(1行), `QueryContext`(複数行), `ExecContext`(更新系)の3メソッドが基本 |
| rows.Close() | `defer rows.Close()` を必ず書く。忘れると接続プールが枯渇する |
| トランザクション | `BeginTx` → 処理 → `Commit` / `Rollback`。ヘルパー関数で安全に管理する |
| sqlx vs GORM | sqlxは生SQL+自動マッピングでバランス良好。GORMはCRUD中心の高速開発向き |
| マイグレーション | golang-migrateでSQL管理。本番で `AutoMigrate` は使わない。up/downファイルを必ずペアで作成 |
| SQLインジェクション対策 | 文字列連結でSQL組み立て禁止。必ずプレースホルダ（`$1`, `?`）を使う |

---

## やってみよう！

### 演習1: Todoリポジトリ

`database/sql` を使って以下のインターフェースを実装してください。

```go
type Todo struct {
    ID    int64
    Title string
    Done  bool
}

type TodoRepository interface {
    Create(ctx context.Context, todo *Todo) error
    GetByID(ctx context.Context, id int64) (*Todo, error)
    List(ctx context.Context) ([]Todo, error)
    ToggleDone(ctx context.Context, id int64) error // UPDATE SET done = NOT done
    Delete(ctx context.Context, id int64) error
}
```

### 演習2: トランザクションでバッチ処理

本章の `WithTransaction` ヘルパーを使い、CSVから読み込んだユーザーを一括登録する関数を書いてください。途中でエラーが発生した場合に全件ロールバックされることを確認しましょう。

### 演習3: sqlx への移行

演習1のTodoリポジトリを `sqlx` に書き換えてください。`GetContext` / `SelectContext` で `Scan` のコードがどれだけ減るか体感できます。
