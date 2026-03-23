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

:::details 解答例を見る

```go
package main

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"log"
	"time"

	_ "github.com/lib/pq" // PostgreSQLドライバ
)

// 前提: PostgreSQLが起動済みで、以下のテーブルが作成済みであること
// CREATE TABLE todos (
//     id BIGSERIAL PRIMARY KEY,
//     title VARCHAR(255) NOT NULL,
//     done BOOLEAN NOT NULL DEFAULT false
// );

var ErrNotFound = errors.New("not found")

type Todo struct {
	ID    int64
	Title string
	Done  bool
}

// TodoRepository インターフェースに対する具象実装
type TodoRepo struct {
	db *sql.DB // 構造体にDBを保持（グローバル変数にしない）
}

func NewTodoRepo(db *sql.DB) *TodoRepo {
	return &TodoRepo{db: db}
}

func (r *TodoRepo) Create(ctx context.Context, todo *Todo) error {
	// RETURNING で自動採番された ID を受け取る（PostgreSQL）
	return r.db.QueryRowContext(ctx,
		"INSERT INTO todos (title, done) VALUES ($1, $2) RETURNING id",
		todo.Title, todo.Done,
	).Scan(&todo.ID)
}

func (r *TodoRepo) GetByID(ctx context.Context, id int64) (*Todo, error) {
	var t Todo
	err := r.db.QueryRowContext(ctx,
		"SELECT id, title, done FROM todos WHERE id = $1", id,
	).Scan(&t.ID, &t.Title, &t.Done)

	// sql.ErrNoRows を自前のエラーに変換する（呼び出し側がDBに依存しない）
	if errors.Is(err, sql.ErrNoRows) {
		return nil, ErrNotFound
	}
	return &t, err
}

func (r *TodoRepo) List(ctx context.Context) ([]Todo, error) {
	rows, err := r.db.QueryContext(ctx,
		"SELECT id, title, done FROM todos ORDER BY id")
	if err != nil {
		return nil, err
	}
	defer rows.Close() // 必ず閉じる（接続リーク防止）

	var todos []Todo
	for rows.Next() {
		var t Todo
		if err := rows.Scan(&t.ID, &t.Title, &t.Done); err != nil {
			return nil, err
		}
		todos = append(todos, t)
	}
	return todos, rows.Err() // 反復中のエラーも確認
}

func (r *TodoRepo) ToggleDone(ctx context.Context, id int64) error {
	// done = NOT done で現在の値を反転
	result, err := r.db.ExecContext(ctx,
		"UPDATE todos SET done = NOT done WHERE id = $1", id)
	if err != nil {
		return err
	}
	n, _ := result.RowsAffected()
	if n == 0 {
		return ErrNotFound
	}
	return nil
}

func (r *TodoRepo) Delete(ctx context.Context, id int64) error {
	result, err := r.db.ExecContext(ctx,
		"DELETE FROM todos WHERE id = $1", id)
	if err != nil {
		return err
	}
	n, _ := result.RowsAffected()
	if n == 0 {
		return ErrNotFound
	}
	return nil
}

func main() {
	// 接続文字列は環境に合わせて変更してください
	db, err := sql.Open("postgres",
		"postgres://user:pass@localhost/mydb?sslmode=disable")
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	// 本番では接続プール設定を必ず行う
	db.SetMaxOpenConns(25)
	db.SetMaxIdleConns(5)
	db.SetConnMaxLifetime(5 * time.Minute)

	ctx := context.Background()
	repo := NewTodoRepo(db)

	// 作成
	todo := &Todo{Title: "Goの勉強", Done: false}
	if err := repo.Create(ctx, todo); err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Created: %+v\n", todo)
	// → Created: &{ID:1 Title:Goの勉強 Done:false}

	// 完了切替
	repo.ToggleDone(ctx, todo.ID)
	updated, _ := repo.GetByID(ctx, todo.ID)
	fmt.Printf("Toggled: %+v\n", updated)
	// → Toggled: &{ID:1 Title:Goの勉強 Done:true}

	// 一覧
	list, _ := repo.List(ctx)
	fmt.Printf("List: %+v\n", list)
	// → List: [{ID:1 Title:Goの勉強 Done:true}]
}
```

`defer rows.Close()` と `rows.Err()` のチェックは `database/sql` のCRUD操作で最も忘れがちなポイントです。`RowsAffected()` で更新対象が存在したか確認するパターンも実務で頻出します。

:::

### 演習2: トランザクションでバッチ処理

本章の `WithTransaction` ヘルパーを使い、CSVから読み込んだユーザーを一括登録する関数を書いてください。途中でエラーが発生した場合に全件ロールバックされることを確認しましょう。

:::details 解答例を見る

```go
package main

import (
	"context"
	"database/sql"
	"encoding/csv"
	"fmt"
	"log"
	"os"
	"time"

	_ "github.com/lib/pq"
)

// 前提: PostgreSQLが起動済みで、以下のテーブルが作成済みであること
// CREATE TABLE users (
//     id BIGSERIAL PRIMARY KEY,
//     name VARCHAR(100) NOT NULL,
//     email VARCHAR(255) NOT NULL UNIQUE
// );

// WithTransaction はトランザクションを安全に管理するヘルパー
func WithTransaction(ctx context.Context, db *sql.DB, fn func(tx *sql.Tx) error) error {
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return err
	}
	defer func() {
		if p := recover(); p != nil {
			_ = tx.Rollback()
			panic(p) // パニックは再送出
		}
	}()

	if err := fn(tx); err != nil {
		_ = tx.Rollback() // エラー時は全件ロールバック
		return err
	}
	return tx.Commit()
}

// BatchCreateUsersFromCSV はCSVファイルからユーザーを一括登録する
// 途中でエラーが発生した場合、全件ロールバックされる
func BatchCreateUsersFromCSV(ctx context.Context, db *sql.DB, csvPath string) (int, error) {
	// CSVファイルを開く
	f, err := os.Open(csvPath)
	if err != nil {
		return 0, fmt.Errorf("open csv: %w", err)
	}
	defer f.Close()

	records, err := csv.NewReader(f).ReadAll()
	if err != nil {
		return 0, fmt.Errorf("read csv: %w", err)
	}

	var count int
	err = WithTransaction(ctx, db, func(tx *sql.Tx) error {
		for i, record := range records {
			if i == 0 {
				continue // ヘッダー行をスキップ
			}
			if len(record) < 2 {
				return fmt.Errorf("行 %d: カラム不足（name, email が必要）", i+1)
			}

			name, email := record[0], record[1]
			if name == "" || email == "" {
				// バリデーションエラー → 全件ロールバック
				return fmt.Errorf("行 %d: name または email が空です", i+1)
			}

			_, err := tx.ExecContext(ctx,
				"INSERT INTO users (name, email) VALUES ($1, $2)",
				name, email)
			if err != nil {
				// UNIQUE制約違反等もここでキャッチ → 全件ロールバック
				return fmt.Errorf("行 %d: %w", i+1, err)
			}
			count++
		}
		return nil // nil を返すと Commit される
	})

	if err != nil {
		return 0, err
	}
	return count, nil
}

func main() {
	db, err := sql.Open("postgres",
		"postgres://user:pass@localhost/mydb?sslmode=disable")
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	db.SetMaxOpenConns(25)
	db.SetMaxIdleConns(5)
	db.SetConnMaxLifetime(5 * time.Minute)

	// CSVファイル例（users.csv）:
	// name,email
	// Tanaka,tanaka@example.com
	// Suzuki,suzuki@example.com
	// ,invalid@example.com       ← name が空 → 全件ロールバック

	ctx := context.Background()
	count, err := BatchCreateUsersFromCSV(ctx, db, "users.csv")
	if err != nil {
		log.Printf("バッチ登録失敗（全件ロールバック）: %v", err)
		return
	}
	fmt.Printf("%d 件のユーザーを登録しました\n", count)
}

// 実行例（正常時）:
// 2 件のユーザーを登録しました
//
// 実行例（エラー時）:
// バッチ登録失敗（全件ロールバック）: 行 3: name または email が空です
```

`WithTransaction` ヘルパーの利点は、`fn` が `error` を返すだけで自動的に Rollback が走る点です。呼び出し側がCommit/Rollbackの管理を意識する必要がなく、「全件成功 or 全件失敗」を確実に保証できます。

:::

### 演習3: sqlx への移行

演習1のTodoリポジトリを `sqlx` に書き換えてください。`GetContext` / `SelectContext` で `Scan` のコードがどれだけ減るか体感できます。

:::details 解答例を見る

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"log"
	"time"

	"database/sql"

	"github.com/jmoiron/sqlx"
	_ "github.com/lib/pq"
)

// 前提: PostgreSQLが起動済みで、todosテーブルが作成済みであること

var ErrNotFound = errors.New("not found")

// db タグで構造体フィールドとカラムを対応づける
type Todo struct {
	ID    int64  `db:"id" json:"id"`
	Title string `db:"title" json:"title"`
	Done  bool   `db:"done" json:"done"`
}

type TodoRepo struct {
	db *sqlx.DB
}

func NewTodoRepo(db *sqlx.DB) *TodoRepo {
	return &TodoRepo{db: db}
}

func (r *TodoRepo) Create(ctx context.Context, todo *Todo) error {
	// sqlx でも RETURNING + Scan は QueryRowContext を使う
	return r.db.QueryRowContext(ctx,
		"INSERT INTO todos (title, done) VALUES ($1, $2) RETURNING id",
		todo.Title, todo.Done,
	).Scan(&todo.ID)
}

func (r *TodoRepo) GetByID(ctx context.Context, id int64) (*Todo, error) {
	var t Todo
	// GetContext: 1行取得 + 構造体への自動マッピング（Scan不要！）
	err := r.db.GetContext(ctx, &t,
		"SELECT id, title, done FROM todos WHERE id = $1", id)
	if errors.Is(err, sql.ErrNoRows) {
		return nil, ErrNotFound
	}
	return &t, err
}

func (r *TodoRepo) List(ctx context.Context) ([]Todo, error) {
	var todos []Todo
	// SelectContext: 複数行取得 + スライスへの自動マッピング
	// rows.Next() + Scan のループが丸ごと不要になる
	err := r.db.SelectContext(ctx, &todos,
		"SELECT id, title, done FROM todos ORDER BY id")
	return todos, err
}

func (r *TodoRepo) ToggleDone(ctx context.Context, id int64) error {
	result, err := r.db.ExecContext(ctx,
		"UPDATE todos SET done = NOT done WHERE id = $1", id)
	if err != nil {
		return err
	}
	n, _ := result.RowsAffected()
	if n == 0 {
		return ErrNotFound
	}
	return nil
}

func (r *TodoRepo) Delete(ctx context.Context, id int64) error {
	result, err := r.db.ExecContext(ctx,
		"DELETE FROM todos WHERE id = $1", id)
	if err != nil {
		return err
	}
	n, _ := result.RowsAffected()
	if n == 0 {
		return ErrNotFound
	}
	return nil
}

func main() {
	// sqlx.Connect は Open + Ping を一度に行う
	db, err := sqlx.Connect("postgres",
		"postgres://user:pass@localhost/mydb?sslmode=disable")
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	db.SetMaxOpenConns(25)
	db.SetMaxIdleConns(5)
	db.SetConnMaxLifetime(5 * time.Minute)

	ctx := context.Background()
	repo := NewTodoRepo(db)

	todo := &Todo{Title: "sqlxを試す", Done: false}
	repo.Create(ctx, todo)
	fmt.Printf("Created: %+v\n", todo)
	// → Created: &{ID:1 Title:sqlxを試す Done:false}

	list, _ := repo.List(ctx)
	fmt.Printf("List: %+v\n", list)
	// → List: [{ID:1 Title:sqlxを試す Done:false}]
}
```

`database/sql` 版と比較すると、`GetByID` で `Scan(&t.ID, &t.Title, &t.Done)` が不要になり、`List` では `rows.Next()` ループ全体が `SelectContext` 一行に置き換わりました。SQLはそのままで構造体マッピングだけ楽になるのが `sqlx` の最大の利点です。

:::
