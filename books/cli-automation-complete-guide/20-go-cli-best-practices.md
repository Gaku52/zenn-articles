# Go CLIベストプラクティス

本章では、プロフェッショナルなGo CLIツールを開発するためのベストプラクティスを学びます。

## エラーハンドリング

```go
if err != nil {
    fmt.Fprintf(os.Stderr, "Error: %v\n", err)
    os.Exit(1)
}
```

## 構造化ログ

```go
import "log/slog"

logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
logger.Info("Creating project", "name", name)
```

## テスト

```go
func TestCreateCommand(t *testing.T) {
    cmd := createCmd
    cmd.SetArgs([]string{"myapp", "--template", "react"})

    if err := cmd.Execute(); err != nil {
        t.Fatalf("Expected no error, got %v", err)
    }
}
```

## まとめ

適切なエラーハンドリング、ログ、テストにより、信頼性の高いCLIツールを開発できます。
