---
title: "Cobraフレームワーク"
---

# Cobraフレームワーク

Cobraは、Goで最も人気のあるCLIフレームワークで、kubectl、Hugo、GitHub CLIなど、多くの著名なツールで採用されています。本章では、Cobraを使った高機能なCLIツールの開発方法を学びます。

## Cobraの特徴

### 強力なサブコマンドサポート

階層的なサブコマンド構造を簡単に実装できます。`mycli create project`のような複雑なコマンド階層も直感的に定義できます。

### フラグ管理

pflag（POSIX準拠のflagパッケージ）を統合し、グローバルフラグとローカルフラグを柔軟に管理できます。

### 自動ヘルプ生成

コマンド定義から自動的に美しいヘルプメッセージを生成します。

### Viperとの統合

設定ファイル管理ライブラリのViperとシームレスに統合でき、設定ファイル、環境変数、フラグを統一的に扱えます。

## Cobraのインストールとセットアップ

### cobra-cliのインストール

```bash
go install github.com/spf13/cobra-cli@latest
```

### プロジェクトの初期化

```bash
# プロジェクトディレクトリ作成
mkdir my-cli-tool
cd my-cli-tool

# Goモジュール初期化
go mod init github.com/username/my-cli-tool

# Cobraプロジェクトの初期化
cobra-cli init

# 依存関係の取得
go mod tidy
```

これにより、以下の構造が生成されます。

```
my-cli-tool/
├── cmd/
│   └── root.go
├── main.go
├── go.mod
└── go.sum
```

## 基本的なCobra実装

### ルートコマンド

cobra-cliが生成する基本的なルートコマンドを見ていきましょう。

```go
// cmd/root.go
package cmd

import (
    "fmt"
    "os"

    "github.com/spf13/cobra"
)

var rootCmd = &cobra.Command{
    Use:   "mycli",
    Short: "A powerful CLI tool",
    Long: `My CLI Tool is a powerful command-line tool for project management.

It provides various commands to create, manage, and deploy projects
with different templates and configurations.`,
    Version: "1.0.0",
}

func Execute() {
    if err := rootCmd.Execute(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}

func init() {
    // グローバルフラグの定義
    rootCmd.PersistentFlags().StringP("config", "c", "", "config file (default is $HOME/.mycli.yaml)")
    rootCmd.PersistentFlags().BoolP("verbose", "v", false, "verbose output")
}
```

```go
// main.go
package main

import "github.com/username/my-cli-tool/cmd"

func main() {
    cmd.Execute()
}
```

## サブコマンドの追加

### cobra-cliでサブコマンドを生成

```bash
# createコマンドの追加
cobra-cli add create

# listコマンドの追加
cobra-cli add list

# deleteコマンドの追加
cobra-cli add delete
```

### createコマンドの実装

```go
// cmd/create.go
package cmd

import (
    "fmt"
    "os"

    "github.com/spf13/cobra"
)

var (
    createTemplate string
    createFeatures []string
    createOutput   string
)

var createCmd = &cobra.Command{
    Use:   "create [name]",
    Short: "Create a new project",
    Long: `Create a new project with the specified name and template.

The project will be initialized with the selected template and features.`,
    Args: cobra.ExactArgs(1),
    RunE: func(cmd *cobra.Command, args []string) error {
        name := args[0]

        // Verbose output
        verbose, _ := cmd.Flags().GetBool("verbose")
        if verbose {
            fmt.Printf("Creating project: %s\n", name)
            fmt.Printf("Template: %s\n", createTemplate)
            fmt.Printf("Features: %v\n", createFeatures)
            fmt.Printf("Output: %s\n", createOutput)
        }

        // Create project
        if err := createProject(name, createTemplate, createFeatures, createOutput); err != nil {
            return fmt.Errorf("failed to create project: %w", err)
        }

        fmt.Printf("✓ Project %s created successfully\n", name)
        return nil
    },
    Example: `  mycli create myproject
  mycli create myproject --template react
  mycli create myproject --template vue --features typescript,eslint
  mycli create myproject --output ./projects`,
}

func init() {
    rootCmd.AddCommand(createCmd)

    // ローカルフラグの定義
    createCmd.Flags().StringVarP(&createTemplate, "template", "t", "default", "Project template")
    createCmd.Flags().StringSliceVarP(&createFeatures, "features", "f", []string{}, "Features to enable (comma-separated)")
    createCmd.Flags().StringVarP(&createOutput, "output", "o", ".", "Output directory")
}

func createProject(name, template string, features []string, output string) error {
    // Implementation
    fmt.Printf("Creating project %s with template %s\n", name, template)
    return nil
}
```

### listコマンドの実装

```go
// cmd/list.go
package cmd

import (
    "fmt"
    "os"
    "text/tabwriter"

    "github.com/spf13/cobra"
)

var (
    listAll    bool
    listFormat string
)

var listCmd = &cobra.Command{
    Use:   "list",
    Short: "List all projects",
    Long: `List all projects in the workspace.

By default, only active projects are shown. Use --all to show archived projects as well.`,
    Aliases: []string{"ls"},
    RunE: func(cmd *cobra.Command, args []string) error {
        projects, err := getProjects(listAll)
        if err != nil {
            return fmt.Errorf("failed to get projects: %w", err)
        }

        if len(projects) == 0 {
            fmt.Println("No projects found")
            return nil
        }

        // Format output
        switch listFormat {
        case "table":
            printTable(projects)
        case "json":
            printJSON(projects)
        case "yaml":
            printYAML(projects)
        default:
            return fmt.Errorf("unknown format: %s", listFormat)
        }

        return nil
    },
    Example: `  mycli list
  mycli list --all
  mycli list --format json
  mycli ls -a`,
}

func init() {
    rootCmd.AddCommand(listCmd)

    listCmd.Flags().BoolVarP(&listAll, "all", "a", false, "Show all projects including archived")
    listCmd.Flags().StringVar(&listFormat, "format", "table", "Output format (table, json, yaml)")
}

type Project struct {
    Name     string
    Template string
    Status   string
    Path     string
}

func getProjects(all bool) ([]Project, error) {
    // Implementation
    return []Project{
        {Name: "project-a", Template: "react", Status: "active", Path: "/path/to/project-a"},
        {Name: "project-b", Template: "vue", Status: "active", Path: "/path/to/project-b"},
    }, nil
}

func printTable(projects []Project) {
    w := tabwriter.NewWriter(os.Stdout, 0, 0, 3, ' ', 0)
    fmt.Fprintln(w, "NAME\tTEMPLATE\tSTATUS\tPATH")
    fmt.Fprintln(w, "----\t--------\t------\t----")
    for _, p := range projects {
        fmt.Fprintf(w, "%s\t%s\t%s\t%s\n", p.Name, p.Template, p.Status, p.Path)
    }
    w.Flush()
}

func printJSON(projects []Project) {
    // JSON output implementation
    fmt.Println("JSON output not implemented")
}

func printYAML(projects []Project) {
    // YAML output implementation
    fmt.Println("YAML output not implemented")
}
```

## フラグの種類と使い分け

### Persistent Flags（永続フラグ）

ルートコマンドとすべてのサブコマンドで使用できるグローバルフラグです。

```go
func init() {
    // すべてのコマンドで使用可能
    rootCmd.PersistentFlags().StringP("config", "c", "", "config file")
    rootCmd.PersistentFlags().BoolP("verbose", "v", false, "verbose output")
    rootCmd.PersistentFlags().Bool("debug", false, "debug mode")
}
```

### Local Flags（ローカルフラグ）

特定のコマンドでのみ使用できるフラグです。

```go
func init() {
    // createコマンドでのみ使用可能
    createCmd.Flags().StringVarP(&createTemplate, "template", "t", "default", "template")
    createCmd.Flags().StringSliceVarP(&createFeatures, "features", "f", []string{}, "features")

    // 必須フラグの設定
    createCmd.MarkFlagRequired("template")
}
```

### フラグの相互依存関係

```go
func init() {
    createCmd.Flags().String("template", "", "template name")
    createCmd.Flags().String("template-url", "", "template URL")

    // どちらか一方が必須
    createCmd.MarkFlagsOneRequired("template", "template-url")

    // 同時使用不可
    createCmd.MarkFlagsMutuallyExclusive("template", "template-url")
}
```

## Viperとの統合

Viperを使用して、設定ファイル、環境変数、フラグを統合的に管理します。

### Viperのセットアップ

```go
// cmd/root.go
package cmd

import (
    "fmt"
    "os"

    "github.com/spf13/cobra"
    "github.com/spf13/viper"
)

var cfgFile string

var rootCmd = &cobra.Command{
    Use:   "mycli",
    Short: "A powerful CLI tool",
}

func Execute() {
    if err := rootCmd.Execute(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}

func init() {
    cobra.OnInitialize(initConfig)

    rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file (default is $HOME/.mycli.yaml)")
    rootCmd.PersistentFlags().StringP("template", "t", "default", "default template")
    rootCmd.PersistentFlags().Bool("auto-install", true, "auto install dependencies")

    // Viperにフラグをバインド
    viper.BindPFlag("template", rootCmd.PersistentFlags().Lookup("template"))
    viper.BindPFlag("auto_install", rootCmd.PersistentFlags().Lookup("auto-install"))
}

func initConfig() {
    if cfgFile != "" {
        // 指定された設定ファイルを使用
        viper.SetConfigFile(cfgFile)
    } else {
        // ホームディレクトリを取得
        home, err := os.UserHomeDir()
        cobra.CheckErr(err)

        // 設定ファイルの検索パスを追加
        viper.AddConfigPath(home)
        viper.AddConfigPath(".")
        viper.SetConfigType("yaml")
        viper.SetConfigName(".mycli")
    }

    // 環境変数を読み込む（MYCLI_で始まる）
    viper.SetEnvPrefix("MYCLI")
    viper.AutomaticEnv()

    // 設定ファイルを読み込む
    if err := viper.ReadInConfig(); err == nil {
        fmt.Fprintln(os.Stderr, "Using config file:", viper.ConfigFileUsed())
    }
}
```

### 設定ファイルの使用

```yaml
# ~/.mycli.yaml
template: react
auto_install: true
features:
  - typescript
  - eslint
  - prettier
editor: vscode
theme: dark
```

### 設定値の取得

```go
func createProject(name string) error {
    // フラグ、環境変数、設定ファイルの順に値を取得
    template := viper.GetString("template")
    autoInstall := viper.GetBool("auto_install")
    features := viper.GetStringSlice("features")
    editor := viper.GetString("editor")

    fmt.Printf("Template: %s\n", template)
    fmt.Printf("Auto install: %v\n", autoInstall)
    fmt.Printf("Features: %v\n", features)
    fmt.Printf("Editor: %s\n", editor)

    return nil
}
```

優先順位は以下の通りです。

1. コマンドラインフラグ
2. 環境変数
3. 設定ファイル
4. デフォルト値

## プリラン・ポストランフック

コマンド実行の前後に処理を挟むことができます。

```go
var createCmd = &cobra.Command{
    Use:   "create [name]",
    Short: "Create a new project",
    Args:  cobra.ExactArgs(1),
    // バリデーション（最初に実行）
    PreRunE: func(cmd *cobra.Command, args []string) error {
        name := args[0]
        if len(name) < 3 {
            return fmt.Errorf("project name must be at least 3 characters")
        }
        fmt.Println("Validating project name...")
        return nil
    },
    // メイン処理
    RunE: func(cmd *cobra.Command, args []string) error {
        name := args[0]
        fmt.Printf("Creating project: %s\n", name)
        return nil
    },
    // クリーンアップ（最後に実行）
    PostRunE: func(cmd *cobra.Command, args []string) error {
        fmt.Println("Project creation completed")
        return nil
    },
}
```

実行順序は以下の通りです。

1. `PersistentPreRun` / `PersistentPreRunE`
2. `PreRun` / `PreRunE`
3. `Run` / `RunE`
4. `PostRun` / `PostRunE`
5. `PersistentPostRun` / `PersistentPostRunE`

## カスタムヘルプとUsage

ヘルプメッセージをカスタマイズできます。

```go
func init() {
    // カスタムヘルプテンプレート
    rootCmd.SetHelpTemplate(`{{.Long}}

Usage:
  {{.UseLine}}

Available Commands:{{range .Commands}}{{if (or .IsAvailableCommand (eq .Name "help"))}}
  {{rpad .Name .NamePadding }} {{.Short}}{{end}}{{end}}

Flags:
{{.LocalFlags.FlagUsages | trimTrailingWhitespaces}}

Use "{{.CommandPath}} [command] --help" for more information about a command.
`)

    // カスタムUsage関数
    rootCmd.SetUsageFunc(func(cmd *cobra.Command) error {
        fmt.Fprintf(cmd.OutOrStderr(), "Custom usage for %s\n", cmd.CommandPath())
        return nil
    })
}
```

## Cobraフレームワーク実装チェックリスト

Cobraを使った CLI開発では、以下を確認しましょう。

- [ ] cobra-cliでプロジェクトを初期化している
- [ ] サブコマンドが適切に階層化されている
- [ ] Persistent FlagsとLocal Flagsを使い分けている
- [ ] フラグのshorthandを提供している
- [ ] 必須フラグをMarkFlagRequiredで指定している
- [ ] Viperで設定ファイルを管理している
- [ ] 環境変数のサポートを実装している
- [ ] PreRunEでバリデーションを実装している
- [ ] エラーは*Eサフィックス関数でerrorを返している
- [ ] Exampleフィールドで使用例を提供している

## まとめ

Cobraは、GoにおけるデファクトスタンダードのCLIフレームワークです。強力なサブコマンドサポート、柔軟なフラグ管理、Viperとの統合により、プロフェッショナルなCLIツールを効率的に開発できます。多くのOSSプロジェクトで採用されており、実績も十分です。

次章では、開発したGo CLIツールのビルドと配布方法を学びます。

## 参考文献

- [Cobra公式ドキュメント](https://cobra.dev/)
- [Cobra GitHub Repository](https://github.com/spf13/cobra)
- [Viper公式ドキュメント](https://github.com/spf13/viper)
- [pflag - POSIX flags](https://github.com/spf13/pflag)
