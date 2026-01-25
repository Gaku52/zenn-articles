# Go CLIベストプラクティス

本章では、プロフェッショナルなGo CLIツールを開発するためのベストプラクティスを学びます。Goは静的型付け、高速なコンパイル、優れた並行処理機能を持ち、CLI開発に適した言語です。

## エラーハンドリングパターン

Go言語では明示的なエラーハンドリングが推奨されており、CLIツールにおいても適切なエラー処理が信頼性を大きく左右します。

### 基本的なエラーハンドリング

```go
package main

import (
    "fmt"
    "os"
)

func processFile(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return fmt.Errorf("failed to open file %s: %w", filename, err)
    }
    defer file.Close()

    // ファイル処理
    return nil
}

func main() {
    if err := processFile("config.yaml"); err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }
}
```

### カスタムエラー型

より詳細なエラー情報を提供するため、カスタムエラー型を定義できます。

```go
package main

import (
    "errors"
    "fmt"
)

// カスタムエラー型
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error in field '%s': %s", e.Field, e.Message)
}

// エラーのチェック
func validateConfig(config map[string]string) error {
    if config["name"] == "" {
        return &ValidationError{
            Field:   "name",
            Message: "name is required",
        }
    }
    return nil
}

func main() {
    config := map[string]string{}

    if err := validateConfig(config); err != nil {
        var validationErr *ValidationError
        if errors.As(err, &validationErr) {
            fmt.Printf("Validation failed: %s\n", validationErr.Message)
        }
    }
}
```

## 構造体による設定管理

CLIツールの設定は構造体で管理し、フラグやファイルから読み込む設計が推奨されます。

```go
package main

import (
    "flag"
    "fmt"
    "os"

    "gopkg.in/yaml.v3"
)

// Config 設定構造体
type Config struct {
    ProjectName string `yaml:"project_name"`
    Environment string `yaml:"environment"`
    Debug       bool   `yaml:"debug"`
    Timeout     int    `yaml:"timeout"`
}

// LoadConfig ファイルから設定を読み込む
func LoadConfig(filename string) (*Config, error) {
    data, err := os.ReadFile(filename)
    if err != nil {
        return nil, fmt.Errorf("failed to read config file: %w", err)
    }

    var config Config
    if err := yaml.Unmarshal(data, &config); err != nil {
        return nil, fmt.Errorf("failed to parse config: %w", err)
    }

    return &config, nil
}

func main() {
    configFile := flag.String("config", "config.yaml", "Path to config file")
    projectName := flag.String("name", "", "Project name (overrides config)")
    flag.Parse()

    // 設定ファイル読み込み
    config, err := LoadConfig(*configFile)
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error loading config: %v\n", err)
        os.Exit(1)
    }

    // フラグでのオーバーライド
    if *projectName != "" {
        config.ProjectName = *projectName
    }

    fmt.Printf("Running project: %s\n", config.ProjectName)
}
```

## コンテキストの活用

タイムアウトやキャンセル処理には、contextパッケージを活用します。

```go
package main

import (
    "context"
    "fmt"
    "time"
)

// タイムアウト付き処理
func processWithTimeout(ctx context.Context) error {
    select {
    case <-time.After(5 * time.Second):
        return fmt.Errorf("processing completed")
    case <-ctx.Done():
        return ctx.Err()
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    if err := processWithTimeout(ctx); err != nil {
        fmt.Printf("Process result: %v\n", err)
    }
}
```

## ログ出力

標準のlogパッケージまたはlog/slogを使用した構造化ログが推奨されます。

### 基本的なログ出力

```go
package main

import (
    "log"
    "os"
)

func main() {
    // ログのプレフィックス設定
    log.SetPrefix("[CLI] ")
    log.SetFlags(log.Ldate | log.Ltime | log.Lshortfile)

    log.Println("Starting application...")

    if err := processData(); err != nil {
        log.Fatalf("Fatal error: %v", err)
    }
}

func processData() error {
    log.Println("Processing data...")
    return nil
}
```

### 構造化ログ（slog）

```go
package main

import (
    "log/slog"
    "os"
)

func main() {
    // JSON形式のログハンドラー
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }))

    logger.Info("Starting application",
        slog.String("version", "1.0.0"),
        slog.String("env", "production"),
    )

    logger.Warn("Resource usage high",
        slog.Int("cpu", 80),
        slog.Int("memory", 90),
    )

    logger.Error("Failed to connect",
        slog.String("host", "db.example.com"),
        slog.Int("port", 5432),
    )
}
```

## 並行処理

Goの強みであるgoroutineとchannelを活用した並行処理の実装例です。

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Worker 並行処理のワーカー
func worker(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()

    for job := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, job)
        time.Sleep(time.Second)
        results <- job * 2
    }
}

func main() {
    const numWorkers = 3
    const numJobs = 10

    jobs := make(chan int, numJobs)
    results := make(chan int, numJobs)
    var wg sync.WaitGroup

    // ワーカー起動
    for w := 1; w <= numWorkers; w++ {
        wg.Add(1)
        go worker(w, jobs, results, &wg)
    }

    // ジョブ投入
    for j := 1; j <= numJobs; j++ {
        jobs <- j
    }
    close(jobs)

    // 全ワーカー完了を待機
    wg.Wait()
    close(results)

    // 結果の取得
    for result := range results {
        fmt.Printf("Result: %d\n", result)
    }
}
```

### エラーハンドリング付き並行処理

```go
package main

import (
    "fmt"
    "sync"
)

type Result struct {
    ID    int
    Value int
    Err   error
}

func workerWithError(id int, jobs <-chan int, results chan<- Result, wg *sync.WaitGroup) {
    defer wg.Done()

    for job := range jobs {
        var result Result
        result.ID = job

        // エラーが発生する可能性のある処理
        if job%2 == 0 {
            result.Err = fmt.Errorf("even number not allowed: %d", job)
        } else {
            result.Value = job * 2
        }

        results <- result
    }
}

func main() {
    jobs := make(chan int, 5)
    results := make(chan Result, 5)
    var wg sync.WaitGroup

    // ワーカー起動
    for w := 1; w <= 2; w++ {
        wg.Add(1)
        go workerWithError(w, jobs, results, &wg)
    }

    // ジョブ投入
    for j := 1; j <= 5; j++ {
        jobs <- j
    }
    close(jobs)

    // ワーカー完了待機
    go func() {
        wg.Wait()
        close(results)
    }()

    // 結果処理
    for result := range results {
        if result.Err != nil {
            fmt.Printf("Job %d failed: %v\n", result.ID, result.Err)
        } else {
            fmt.Printf("Job %d succeeded: %d\n", result.ID, result.Value)
        }
    }
}
```

## テスト

CLIツールのテストは、コマンド実行とその出力を検証します。

```go
// main_test.go
package main

import (
    "bytes"
    "os"
    "testing"

    "github.com/spf13/cobra"
)

func TestCreateCommand(t *testing.T) {
    var rootCmd = &cobra.Command{
        Use:   "app",
        Short: "Test application",
    }

    var createCmd = &cobra.Command{
        Use:   "create [name]",
        Short: "Create new project",
        Args:  cobra.ExactArgs(1),
        RunE: func(cmd *cobra.Command, args []string) error {
            name := args[0]
            cmd.Printf("Creating project: %s\n", name)
            return nil
        },
    }

    rootCmd.AddCommand(createCmd)

    // 出力のキャプチャ
    buf := new(bytes.Buffer)
    rootCmd.SetOut(buf)
    rootCmd.SetErr(buf)
    rootCmd.SetArgs([]string{"create", "myapp"})

    err := rootCmd.Execute()
    if err != nil {
        t.Fatalf("Expected no error, got %v", err)
    }

    output := buf.String()
    expected := "Creating project: myapp\n"
    if output != expected {
        t.Errorf("Expected %q, got %q", expected, output)
    }
}

func TestValidation(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        wantErr bool
    }{
        {"valid name", "myapp", false},
        {"empty name", "", true},
        {"invalid chars", "my@app", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := validateProjectName(tt.input)
            if (err != nil) != tt.wantErr {
                t.Errorf("validateProjectName() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}

func validateProjectName(name string) error {
    if name == "" {
        return fmt.Errorf("project name is required")
    }
    // 簡易的なバリデーション
    for _, c := range name {
        if !((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || c == '-' || c == '_') {
            return fmt.Errorf("invalid character in project name: %c", c)
        }
    }
    return nil
}
```

## Go CLI開発チェックリスト

プロフェッショナルなGo CLIツールを開発するためのチェックリストです。

### エラーハンドリング
- [ ] すべてのエラーを適切にハンドリングしている
- [ ] エラーメッセージに十分なコンテキスト情報を含めている
- [ ] fmt.Errorfで%wを使ってエラーをラップしている
- [ ] カスタムエラー型で詳細情報を提供している
- [ ] os.Exit()の使用を適切に制限している

### 設定管理
- [ ] 構造体で設定を管理している
- [ ] 環境変数、フラグ、設定ファイルをサポートしている
- [ ] デフォルト値を適切に設定している
- [ ] 設定の優先順位が明確である

### ログ出力
- [ ] 適切なログレベルを使用している
- [ ] 構造化ログを採用している
- [ ] 機密情報をログに出力していない

### 並行処理
- [ ] goroutineのリーク対策を行っている
- [ ] sync.WaitGroupで適切に待機している
- [ ] channelの適切なクローズを実施している
- [ ] context.Contextでキャンセル処理を実装している

### テスト
- [ ] ユニットテストを記述している
- [ ] テストカバレッジが十分である
- [ ] エラーケースのテストを含めている
- [ ] テーブル駆動テストを活用している

### パフォーマンス
- [ ] 不要なメモリアロケーションを削減している
- [ ] バッファリングを適切に使用している
- [ ] goroutineの数を制御している

## まとめ

本章では、Go言語によるCLIツール開発のベストプラクティスを学びました。

**重要ポイント**:
- 明示的なエラーハンドリングで信頼性を確保
- 構造体による設定管理で保守性を向上
- contextパッケージでタイムアウトとキャンセル処理を実装
- 構造化ログで運用時のデバッグを容易に
- goroutineとchannelで効率的な並行処理を実現
- テストによる品質保証

これらのベストプラクティスを実践することで、保守性と信頼性の高いGo CLIツールを開発できます。

## 参考文献

- [Effective Go - The Go Programming Language](https://go.dev/doc/effective_go)
- [Go by Example](https://gobyexample.com/)
- [Cobra - GitHub](https://github.com/spf13/cobra)
- [log/slog package - Go](https://pkg.go.dev/log/slog)
