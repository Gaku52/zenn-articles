---
title: "CLI開発"
---

# Chapter 10: CLI開発

> cobra, viper, クロスコンパイル — 本格CLIツールの作り方

シングルバイナリで配布でき、クロスコンパイルも標準サポート。Docker、Kubernetes、GitHub CLIなど主要CLIツールの多くがGoで書かれています。標準`flag`から業界標準の`cobra`+`viper`まで、本格CLIツールの作り方を学びます。

---

## 標準flagパッケージ -- シンプルなCLIの出発点

フラグが数個のシンプルなツールなら、標準ライブラリだけで十分です。

```go
package main

import (
    "flag"
    "fmt"
    "os"
)

func main() {
    host := flag.String("host", "localhost", "サーバーホスト名")
    port := flag.Int("port", 8080, "サーバーポート番号")
    verbose := flag.Bool("verbose", false, "詳細出力を有効にする")

    flag.Usage = func() {
        fmt.Fprintf(os.Stderr, "Usage: %s [options]\n\nOptions:\n", os.Args[0])
        flag.PrintDefaults()
    }
    flag.Parse()

    if *verbose {
        fmt.Printf("Host: %s, Port: %d\n", *host, *port)
    }
    fmt.Printf("サーバー起動: %s:%d\n", *host, *port)
}
```

**標準flagの制約:** `-flag`形式のみ（`--flag`非対応）、短縮形`-v`非対応、サブコマンド非対応。サブコマンドが必要になったら`cobra`へ移行しましょう。

| 機能 | 標準flag | cobra |
|------|---------|-------|
| POSIX形式 `--flag` | `-flag`のみ | 対応 |
| 短縮形 `-v` | 非対応 | 対応 |
| サブコマンド | 非対応 | 対応 |
| シェル補完 | 非対応 | bash/zsh/fish/powershell |
| 設定ファイル連携 | 手動 | viper統合 |

---

## cobraフレームワーク -- 業界標準のCLI基盤

`cobra`はサブコマンド、自動ヘルプ、シェル補完、引数バリデーションを宣言的に定義できるフレームワークです。

```bash
go get github.com/spf13/cobra@latest
```

cobraプロジェクトの標準構成は `main.go`(エントリポイント) + `cmd/`(コマンド定義) + `internal/`(ロジック) です。

### ルートコマンド

```go
// cmd/root.go
package cmd

import (
    "fmt"
    "os"
    "github.com/spf13/cobra"
    "github.com/spf13/viper"
)

var rootCmd = &cobra.Command{
    Use:          "mytool",
    Short:        "My awesome CLI tool",
    Version:      "1.0.0",
    SilenceUsage: true, // エラー時にUsageを表示しない
}

func Execute() {
    if err := rootCmd.Execute(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}

func init() {
    cobra.OnInitialize(initConfig)
    // PersistentFlags: 全サブコマンドで使える
    rootCmd.PersistentFlags().String("config", "", "設定ファイルパス")
    rootCmd.PersistentFlags().BoolP("verbose", "v", false, "詳細出力")
    viper.BindPFlag("verbose", rootCmd.PersistentFlags().Lookup("verbose"))
}

func initConfig() {
    home, _ := os.UserHomeDir()
    viper.AddConfigPath(home)
    viper.AddConfigPath(".")
    viper.SetConfigName(".mytool")
    viper.SetConfigType("yaml")
    viper.AutomaticEnv()
    viper.ReadInConfig()
}
```

---

## サブコマンドとフラグ

```go
// cmd/serve.go
package cmd

import (
    "fmt"
    "net/http"
    "github.com/spf13/cobra"
)

var serveCmd = &cobra.Command{
    Use:   "serve",
    Short: "HTTPサーバーを起動する",
    RunE: func(cmd *cobra.Command, args []string) error {
        port, _ := cmd.Flags().GetInt("port")
        host, _ := cmd.Flags().GetString("host")
        addr := fmt.Sprintf("%s:%d", host, port)
        fmt.Printf("サーバー起動: http://%s\n", addr)

        mux := http.NewServeMux()
        mux.HandleFunc("GET /", func(w http.ResponseWriter, r *http.Request) {
            fmt.Fprintln(w, "Hello from mytool!")
        })
        return http.ListenAndServe(addr, mux)
    },
}

func init() {
    rootCmd.AddCommand(serveCmd)
    serveCmd.Flags().IntP("port", "p", 8080, "ポート番号")      // Localフラグ
    serveCmd.Flags().String("host", "localhost", "ホスト名")
}
```

**PersistentFlags vs Flags:** `PersistentFlags()`は全サブコマンドに継承される（`--verbose`など共通設定向き）。`Flags()`は定義コマンド専用（`--port`などコマンド固有の設定向き）。

### ネストしたサブコマンド

`mytool config set key value`のような多階層コマンドも簡単に作れます。

```go
// cmd/config.go
package cmd

import (
    "fmt"
    "github.com/spf13/cobra"
    "github.com/spf13/viper"
)

var configCmd = &cobra.Command{Use: "config", Short: "設定を管理する"}

var configSetCmd = &cobra.Command{
    Use:  "set [key] [value]",
    Args: cobra.ExactArgs(2),
    RunE: func(cmd *cobra.Command, args []string) error {
        viper.Set(args[0], args[1])
        return viper.WriteConfig()
    },
}

var configListCmd = &cobra.Command{
    Use: "list",
    Run: func(cmd *cobra.Command, args []string) {
        for k, v := range viper.AllSettings() {
            fmt.Printf("%s = %v\n", k, v)
        }
    },
}

func init() {
    rootCmd.AddCommand(configCmd)
    configCmd.AddCommand(configSetCmd, configListCmd)
}
```

cobraの`Args`で引数の数を宣言的にバリデーションできます: `NoArgs`, `ExactArgs(n)`, `MinimumNArgs(n)`, `RangeArgs(min, max)`。

---

## viperによる設定管理

`viper`はフラグ・環境変数・設定ファイルを統合管理します。優先順位は以下の通りです。

```
フラグ（--port 3000）      ← 最優先
  ↓
環境変数（MYTOOL_PORT=3000）
  ↓
設定ファイル（.mytool.yaml）
  ↓
デフォルト値               ← 最低優先
```

### 構造体へのマッピング

`mapstructure`タグで設定ファイルの値をGoの構造体に自動マッピングできます。

```go
type Config struct {
    Server   ServerConfig   `mapstructure:"server"`
    Database DatabaseConfig `mapstructure:"database"`
}
type ServerConfig struct {
    Host string `mapstructure:"host"`
    Port int    `mapstructure:"port"`
}

func Load() (*Config, error) {
    viper.SetDefault("server.host", "0.0.0.0")
    viper.SetDefault("server.port", 8080)

    viper.SetEnvPrefix("MYTOOL")
    viper.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))
    viper.AutomaticEnv() // MYTOOL_SERVER_PORT → server.port

    if err := viper.ReadInConfig(); err != nil {
        if _, ok := err.(viper.ConfigFileNotFoundError); !ok {
            return nil, fmt.Errorf("設定ファイル読み込み失敗: %w", err)
        }
    }
    var cfg Config
    if err := viper.Unmarshal(&cfg); err != nil {
        return nil, fmt.Errorf("設定デコード失敗: %w", err)
    }
    return &cfg, nil
}
```

対応する設定ファイル:

```yaml
# .mytool.yaml
server:
  host: "0.0.0.0"
  port: 8080
database:
  driver: "postgres"
  dsn: "postgres://user:pass@localhost/mydb?sslmode=disable"
```

`MYTOOL_SERVER_PORT=3000`で環境変数から上書きでき、`--port 9090`フラグでさらに上書きできます。

---

## クロスコンパイルと配布

Goはクロスコンパイルが標準でサポートされています。

```bash
# Linux AMD64向け
GOOS=linux GOARCH=amd64 go build -o mytool-linux-amd64

# macOS ARM64 (Apple Silicon)向け
GOOS=darwin GOARCH=arm64 go build -o mytool-darwin-arm64

# Windows AMD64向け
GOOS=windows GOARCH=amd64 go build -o mytool-windows-amd64.exe
```

### ldflagsでバージョン情報を埋め込む

```bash
go build -ldflags "-s -w \
  -X main.version=1.2.0 \
  -X main.commit=$(git rev-parse --short HEAD) \
  -X main.date=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  -o mytool
```

`-s -w`はデバッグ情報を除去してバイナリサイズを削減します。

### GoReleaserで配布を自動化

本格的な配布には[GoReleaser](https://goreleaser.com/)がおすすめです。`git tag`をトリガーに、クロスコンパイル・GitHub Releases・Homebrew Tapを自動生成します。`.goreleaser.yaml`にビルド対象OS/Arch、ldflags、アーカイブ形式を定義するだけで運用できます。

---

## アンチパターン: mainにロジックを直書き

```go
// NG: main()にDB接続・クエリ・出力を全部書く → テスト不能
// OK: main()はエントリポイントのみ、ロジックはrun()に分離
func main() {
    if err := run(os.Args[1:], os.Stdout); err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
}

func run(args []string, stdout io.Writer) error {
    cfg, err := parseFlags(args)
    if err != nil {
        return err
    }
    return execute(cfg, stdout)
}
```

`io.Writer`を引数で受け取ると、テスト時に`bytes.Buffer`を渡して出力を検証できます。

---

## まとめ

| 概念 | 要点 |
|------|------|
| flag | 標準ライブラリのみでシンプルなCLIを構築。`-flag`形式のみ対応でサブコマンドは非対応 |
| cobra | サブコマンド・自動ヘルプ・シェル補完・引数バリデーションを備えた業界標準CLIフレームワーク |
| PersistentFlags vs Flags | `PersistentFlags`は全サブコマンドに継承、`Flags`は定義コマンド専用 |
| viper | フラグ・環境変数・設定ファイルを統合管理。優先順位: フラグ > 環境変数 > 設定ファイル > デフォルト値 |
| 構造体マッピング | `mapstructure`タグでviperの設定値をGoの構造体に自動デコード |
| クロスコンパイル | `GOOS`/`GOARCH`指定だけで複数OS/Arch向けバイナリを生成 |
| ldflags | `-ldflags -X`でビルド時にバージョン・コミットハッシュ・日時を埋め込み |
| ロジック分離 | `main()`はエントリポイントのみ、ロジックは`run()`に分離して`io.Writer`でテスト可能に |

---

## やってみよう！

### 演習1: flagパッケージでファイル検索ツール

標準`flag`パッケージを使って、`findgo`コマンドを作ってみましょう。

- `-dir`フラグで検索ディレクトリを指定（デフォルト: `.`）
- `-ext`フラグで拡張子フィルタ（デフォルト: `.go`）
- `-count`フラグでファイル数のみ表示（bool）
- `filepath.WalkDir`を使ってファイルを列挙

### 演習2: cobra + viperでタスク管理CLI

`cobra`と`viper`を使って、`task`コマンドを作ってみましょう。

- `task add "タスク名"` -- タスクを追加
- `task list` -- タスク一覧を表示（`--output json`でJSON出力対応）
- `task done [id]` -- タスクを完了にする
- タスクデータは`~/.task.json`に保存
- ヒント: ルートコマンド + 3つのサブコマンドを`cmd/`に配置

### 演習3: クロスコンパイルとバージョン表示

演習2のツールに以下を追加してみましょう。

- `task version`でバージョン・コミットハッシュ・ビルド日時を表示
- `-ldflags`でビルド時にバージョン情報を埋め込む
- Makefileを作成し、`make build-all`で3プラットフォーム向けにビルド
