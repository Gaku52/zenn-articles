---
title: "第15章 Goレビューガイド"
---

# 第15章 Goレビューガイド

本章ではGoコードレビューにおける重要なチェックポイントと、具体的なレビュー観点について詳しく解説します。Goはシンプルさと並行処理の効率性を重視した言語であり、特にエラーハンドリングとgoroutineの適切な使用が重要なレビューポイントとなります。

## Goコードレビューの特性

Goは「シンプルであること」を哲学とした言語です。レビューでは以下の観点を重点的に確認することが推奨されます。

### レビューの重点領域

- 明示的なエラーハンドリング（if err != nil）
- goroutineとチャネルの適切な使用
- defer文によるリソース管理
- インターフェースの最小化（Accept interfaces, return structs）
- Effective Goの原則遵守

## エラーハンドリング: if err != nil パターン

Goでは例外機構がなく、エラーは戻り値として明示的に処理します。これによりエラーハンドリングが可視化されます。

### 悪い例: エラーの無視

```go
// 危険: エラーを無視している
func readConfig(filename string) Config {
    data, _ := os.ReadFile(filename)  // エラーを無視
    var config Config
    json.Unmarshal(data, &config)  // エラーを無視
    return config
}

// 危険: エラーの情報が失われる
func processUser(userID string) error {
    user, err := getUser(userID)
    if err != nil {
        return err  // コンテキスト情報が失われる
    }
    // 処理...
    return nil
}
```

### 良い例: 適切なエラーハンドリング

```go
import (
    "encoding/json"
    "fmt"
    "os"
)

// エラーを適切に処理し、呼び出し元に伝播
func readConfig(filename string) (*Config, error) {
    data, err := os.ReadFile(filename)
    if err != nil {
        return nil, fmt.Errorf("failed to read config file %s: %w", filename, err)
    }

    var config Config
    if err := json.Unmarshal(data, &config); err != nil {
        return nil, fmt.Errorf("failed to parse config file %s: %w", filename, err)
    }

    return &config, nil
}

// エラーにコンテキストを追加して伝播
func processUser(userID string) error {
    user, err := getUser(userID)
    if err != nil {
        return fmt.Errorf("failed to get user %s: %w", userID, err)
    }

    if err := validateUser(user); err != nil {
        return fmt.Errorf("user validation failed for %s: %w", userID, err)
    }

    if err := saveUser(user); err != nil {
        return fmt.Errorf("failed to save user %s: %w", userID, err)
    }

    return nil
}

// カスタムエラー型の定義
type UserError struct {
    UserID  string
    Message string
    Err     error
}

func (e *UserError) Error() string {
    return fmt.Sprintf("user error [%s]: %s: %v", e.UserID, e.Message, e.Err)
}

func (e *UserError) Unwrap() error {
    return e.Err
}

// カスタムエラーの使用
func getUserWithValidation(userID string) (*User, error) {
    user, err := getUser(userID)
    if err != nil {
        return nil, &UserError{
            UserID:  userID,
            Message: "failed to retrieve user",
            Err:     err,
        }
    }

    if user.IsDeleted {
        return nil, &UserError{
            UserID:  userID,
            Message: "user is deleted",
            Err:     nil,
        }
    }

    return user, nil
}
```

fmt.Errorf の %w フォーマット指定子を使うことで、エラーチェーンを保持し、errors.Is や errors.As でエラーの検証が可能になります。

## defer文によるリソース管理

defer文を使うことで、関数の終了時に確実にリソースを解放できます。これは複数のreturnパスがある場合に特に有効です。

### 悪い例: 手動でのリソース管理

```go
// 危険: すべてのreturnパスでCloseを呼ぶ必要がある
func processFile(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return err
    }

    data, err := readData(file)
    if err != nil {
        file.Close()  // 忘れやすい
        return err
    }

    if err := validateData(data); err != nil {
        file.Close()  // 重複コード
        return err
    }

    file.Close()
    return nil
}
```

### 良い例: deferを使った確実なリソース管理

```go
import (
    "database/sql"
    "fmt"
    "os"
)

// defer で確実にリソースを解放
func processFile(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return fmt.Errorf("failed to open file: %w", err)
    }
    defer file.Close()  // 関数終了時に必ず実行される

    data, err := readData(file)
    if err != nil {
        return fmt.Errorf("failed to read data: %w", err)
    }

    if err := validateData(data); err != nil {
        return fmt.Errorf("data validation failed: %w", err)
    }

    return nil
}

// データベース接続の管理
func queryUsers(db *sql.DB, query string) ([]*User, error) {
    rows, err := db.Query(query)
    if err != nil {
        return nil, fmt.Errorf("query failed: %w", err)
    }
    defer rows.Close()  // 確実にクローズ

    var users []*User
    for rows.Next() {
        var user User
        if err := rows.Scan(&user.ID, &user.Name, &user.Email); err != nil {
            return nil, fmt.Errorf("failed to scan row: %w", err)
        }
        users = append(users, &user)
    }

    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("row iteration error: %w", err)
    }

    return users, nil
}

// 複数のdeferの使用（逆順で実行される）
func processTransaction(db *sql.DB) error {
    tx, err := db.Begin()
    if err != nil {
        return fmt.Errorf("failed to begin transaction: %w", err)
    }
    defer tx.Rollback()  // エラー時のロールバック保証

    // トランザクション処理
    if err := executeQuery1(tx); err != nil {
        return err  // deferでロールバック
    }

    if err := executeQuery2(tx); err != nil {
        return err  // deferでロールバック
    }

    // 成功時のみコミット（Commitが成功するとRollbackは何もしない）
    if err := tx.Commit(); err != nil {
        return fmt.Errorf("failed to commit transaction: %w", err)
    }

    return nil
}
```

## goroutineとチャネルの適切な使用

goroutineは軽量なスレッドであり、並行処理を簡潔に記述できます。ただし、適切に管理しないとリークやデッドロックの原因となります。

### 悪い例: goroutineリークとデッドロック

```go
// 危険: goroutineがリークする
func processItems(items []Item) {
    for _, item := range items {
        go func(i Item) {
            processItem(i)  // 終了を待たない
        }(item)
    }
    // goroutineの完了を待たずに終了
}

// 危険: デッドロックの可能性
func fetchData() []Data {
    ch := make(chan Data)  // バッファなしチャネル

    go func() {
        for i := 0; i < 100; i++ {
            ch <- fetchSingleData(i)  // 受信側がないとブロック
        }
    }()

    return <-ch  // 1つしか受信しない
}
```

### 良い例: 安全な並行処理

```go
import (
    "context"
    "fmt"
    "sync"
    "time"
)

// WaitGroupで完了を待つ
func processItems(items []Item) error {
    var wg sync.WaitGroup
    errCh := make(chan error, len(items))

    for _, item := range items {
        wg.Add(1)
        go func(i Item) {
            defer wg.Done()
            if err := processItem(i); err != nil {
                errCh <- err
            }
        }(item)
    }

    wg.Wait()
    close(errCh)

    // エラーがあれば返す
    for err := range errCh {
        if err != nil {
            return err
        }
    }

    return nil
}

// ワーカープールパターン
func processWithWorkerPool(ctx context.Context, items []Item, numWorkers int) error {
    jobs := make(chan Item, len(items))
    results := make(chan error, len(items))

    // ワーカー起動
    var wg sync.WaitGroup
    for w := 0; w < numWorkers; w++ {
        wg.Add(1)
        go worker(ctx, &wg, jobs, results)
    }

    // ジョブ投入
    for _, item := range items {
        jobs <- item
    }
    close(jobs)

    // ワーカー終了待ち
    wg.Wait()
    close(results)

    // エラーチェック
    for err := range results {
        if err != nil {
            return err
        }
    }

    return nil
}

func worker(ctx context.Context, wg *sync.WaitGroup, jobs <-chan Item, results chan<- error) {
    defer wg.Done()

    for job := range jobs {
        select {
        case <-ctx.Done():
            results <- ctx.Err()
            return
        default:
            if err := processItem(job); err != nil {
                results <- err
            }
        }
    }
}

// コンテキストでのタイムアウト制御
func fetchDataWithTimeout(userID string) (*User, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    resultCh := make(chan *User, 1)
    errCh := make(chan error, 1)

    go func() {
        user, err := fetchUserFromDB(userID)
        if err != nil {
            errCh <- err
            return
        }
        resultCh <- user
    }()

    select {
    case user := <-resultCh:
        return user, nil
    case err := <-errCh:
        return nil, fmt.Errorf("failed to fetch user: %w", err)
    case <-ctx.Done():
        return nil, fmt.Errorf("fetch timeout: %w", ctx.Err())
    }
}
```

## インターフェースの設計

Goでは「Accept interfaces, return structs」という原則が推奨されます。小さなインターフェースを定義し、柔軟性を保ちます。

### 悪い例: 大きすぎるインターフェース

```go
// 悪い例: 大きすぎるインターフェース
type UserRepository interface {
    Create(user *User) error
    Update(user *User) error
    Delete(userID string) error
    FindByID(userID string) (*User, error)
    FindByEmail(email string) (*User, error)
    FindAll() ([]*User, error)
    Count() (int, error)
    // 20個以上のメソッドが定義されている...
}
```

### 良い例: 小さなインターフェース

```go
// インターフェースは小さく保つ
type UserGetter interface {
    GetUser(userID string) (*User, error)
}

type UserSaver interface {
    SaveUser(user *User) error
}

type UserDeleter interface {
    DeleteUser(userID string) error
}

// 必要に応じて組み合わせる
type UserRepository interface {
    UserGetter
    UserSaver
    UserDeleter
}

// 具体的な実装
type SQLUserRepository struct {
    db *sql.DB
}

func (r *SQLUserRepository) GetUser(userID string) (*User, error) {
    var user User
    err := r.db.QueryRow(
        "SELECT id, name, email FROM users WHERE id = ?",
        userID,
    ).Scan(&user.ID, &user.Name, &user.Email)

    if err != nil {
        return nil, fmt.Errorf("failed to get user: %w", err)
    }

    return &user, nil
}

func (r *SQLUserRepository) SaveUser(user *User) error {
    _, err := r.db.Exec(
        "INSERT INTO users (id, name, email) VALUES (?, ?, ?) ON DUPLICATE KEY UPDATE name = ?, email = ?",
        user.ID, user.Name, user.Email, user.Name, user.Email,
    )

    if err != nil {
        return fmt.Errorf("failed to save user: %w", err)
    }

    return nil
}

func (r *SQLUserRepository) DeleteUser(userID string) error {
    _, err := r.db.Exec("DELETE FROM users WHERE id = ?", userID)
    if err != nil {
        return fmt.Errorf("failed to delete user: %w", err)
    }
    return nil
}

// 関数は小さなインターフェースを受け取る
func displayUser(getter UserGetter, userID string) error {
    user, err := getter.GetUser(userID)
    if err != nil {
        return err
    }

    fmt.Printf("User: %s (%s)\n", user.Name, user.Email)
    return nil
}
```

## Goレビューチェックリスト

レビュー時に確認すべき項目をチェックリストにまとめました。

### エラーハンドリング

- [ ] すべてのエラーが適切に処理されているか（_ で無視していないか）
- [ ] エラーにコンテキスト情報が追加されているか（fmt.Errorf with %w）
- [ ] カスタムエラー型が適切に定義されているか
- [ ] panicの使用が適切か（プログラムを続行できない場合のみ）

### 並行処理

- [ ] goroutineのリークが発生していないか
- [ ] sync.WaitGroupやcontextで適切に管理されているか
- [ ] チャネルのバッファサイズが適切か
- [ ] データ競合（data race）が発生していないか
- [ ] contextによるキャンセル処理が実装されているか

### リソース管理

- [ ] defer文でリソースが確実に解放されているか
- [ ] ファイル、データベース接続、HTTPレスポンスボディがクローズされているか
- [ ] deferの順序が適切か（後入れ先出し）

### インターフェース設計

- [ ] インターフェースが小さく保たれているか
- [ ] 「Accept interfaces, return structs」の原則に従っているか
- [ ] 不要なインターフェース定義がないか

### コーディング規約

- [ ] gofmtが適用されているか
- [ ] golint/golangci-lintの警告が解消されているか
- [ ] パッケージ名が適切か（短く、明確）
- [ ] エクスポートされた関数・型にドキュメントコメントがあるか

### パフォーマンス

- [ ] 不要なメモリアロケーションがないか
- [ ] stringの結合にstrings.Builderが使われているか
- [ ] スライスの容量が適切に指定されているか（make([]T, 0, capacity)）

## 実践例: HTTPサーバーのレビュー

実際のコードレビューの例を見てみましょう。

```go
package main

import (
    "context"
    "encoding/json"
    "errors"
    "fmt"
    "log"
    "net/http"
    "time"
)

// カスタムエラー
var (
    ErrUserNotFound = errors.New("user not found")
    ErrInvalidInput = errors.New("invalid input")
)

// User はユーザー情報を表す
type User struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

// UserService はユーザー操作のインターフェース
type UserService interface {
    GetUser(ctx context.Context, id string) (*User, error)
    CreateUser(ctx context.Context, user *User) error
}

// UserHandler はHTTPハンドラー
type UserHandler struct {
    service UserService
    logger  *log.Logger
}

// NewUserHandler は新しいハンドラーを作成する
func NewUserHandler(service UserService, logger *log.Logger) *UserHandler {
    return &UserHandler{
        service: service,
        logger:  logger,
    }
}

// GetUserHandler はGETリクエストを処理する
func (h *UserHandler) GetUserHandler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()

    userID := r.URL.Query().Get("id")
    if userID == "" {
        h.respondError(w, http.StatusBadRequest, ErrInvalidInput)
        return
    }

    user, err := h.service.GetUser(ctx, userID)
    if err != nil {
        if errors.Is(err, ErrUserNotFound) {
            h.respondError(w, http.StatusNotFound, err)
            return
        }
        h.logger.Printf("failed to get user: %v", err)
        h.respondError(w, http.StatusInternalServerError, err)
        return
    }

    h.respondJSON(w, http.StatusOK, user)
}

// CreateUserHandler はPOSTリクエストを処理する
func (h *UserHandler) CreateUserHandler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()

    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        h.respondError(w, http.StatusBadRequest, fmt.Errorf("invalid request body: %w", err))
        return
    }
    defer r.Body.Close()

    if err := validateUser(&user); err != nil {
        h.respondError(w, http.StatusBadRequest, err)
        return
    }

    if err := h.service.CreateUser(ctx, &user); err != nil {
        h.logger.Printf("failed to create user: %v", err)
        h.respondError(w, http.StatusInternalServerError, err)
        return
    }

    h.respondJSON(w, http.StatusCreated, user)
}

// respondJSON はJSON形式でレスポンスを返す
func (h *UserHandler) respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)

    if err := json.NewEncoder(w).Encode(data); err != nil {
        h.logger.Printf("failed to encode response: %v", err)
    }
}

// respondError はエラーレスポンスを返す
func (h *UserHandler) respondError(w http.ResponseWriter, status int, err error) {
    h.respondJSON(w, status, map[string]string{
        "error": err.Error(),
    })
}

// validateUser はユーザー情報を検証する
func validateUser(user *User) error {
    if user.Name == "" {
        return fmt.Errorf("%w: name is required", ErrInvalidInput)
    }
    if user.Email == "" {
        return fmt.Errorf("%w: email is required", ErrInvalidInput)
    }
    return nil
}
```

このコードは、エラーハンドリング、コンテキスト管理、インターフェース設計のベストプラクティスを実践しています。

## まとめ

本章ではGoコードレビューの重要なポイントを解説しました。

主なポイント:
- すべてのエラーを明示的に処理し、コンテキストを追加する
- defer文でリソース管理を確実に行う
- goroutineとチャネルを適切に管理し、リークを防ぐ
- 小さなインターフェースを定義し、柔軟性を保つ
- Effective Goの原則に従う

次章では、Danger.jsによるコードレビュー自動化について学びます。

## 参考文献

- [Effective Go](https://go.dev/doc/effective_go)
- [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)
- [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)
- [Go Blog - Error handling and Go](https://go.dev/blog/error-handling-and-go)
- [Go Concurrency Patterns](https://go.dev/blog/pipelines)
