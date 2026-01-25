---
title: "Go CLIのセットアップ"
---

# Go CLIのセットアップ

Goは、コンパイル型の静的型付け言語で、高速な実行速度と単一バイナリへのコンパイルが特徴です。本章では、Goを使った高パフォーマンスなCLIツール開発のためのセットアップと基本的な実装方法を学びます。

## Goの特徴とCLI開発における利点

### 高速な実行速度

Goはコンパイル型言語であり、ネイティブコードにコンパイルされるため、Python などのインタープリタ型言語と比較して高速に動作します。大量のファイル処理やデータ変換を行うCLIツールに適しています。

### 単一バイナリへのコンパイル

依存関係を含めて単一の実行可能ファイルにコンパイルできるため、配布が容易です。ユーザーはGoのランタイムをインストールする必要がありません。

### クロスコンパイルの容易さ

異なるプラットフォーム向けのバイナリを、単一の開発環境から簡単にビルドできます。

### 並行処理のサポート

Goroutineとchannelを使った並行処理が言語レベルでサポートされており、並列処理が必要なCLIツールに最適です。

## Go環境のセットアップ

### Goのインストール

```bash
# macOS（Homebrewを使用）
brew install go

# Linux（公式インストーラー）
wget https://go.dev/dl/go1.21.5.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.21.5.linux-amd64.tar.gz

# 環境変数の設定
export PATH=$PATH:/usr/local/go/bin
export GOPATH=$HOME/go
export PATH=$PATH:$GOPATH/bin
```

バージョン確認:

```bash
go version
# go version go1.21.5 darwin/arm64
```

### 開発環境の設定

```bash
# エディタ設定（VS Code）
code --install-extension golang.go

# 開発ツールのインストール
go install golang.org/x/tools/gopls@latest
go install github.com/go-delve/delve/cmd/dlv@latest
go install honnef.co/go/tools/cmd/staticcheck@latest
```

## プロジェクトの初期化

### プロジェクト構造の作成

推奨されるGoプロジェクト構造は以下の通りです。

```bash
mkdir my-cli-tool
cd my-cli-tool

# Goモジュールの初期化
go mod init github.com/username/my-cli-tool
```

### 標準的なディレクトリ構造

```
my-cli-tool/
├── cmd/
│   └── mycli/
│       └── main.go          # エントリーポイント
├── internal/
│   ├── project/
│   │   ├── project.go       # プロジェクト管理ロジック
│   │   └── project_test.go  # テスト
│   ├── template/
│   │   └── template.go      # テンプレート処理
│   └── config/
│       └── config.go        # 設定管理
├── pkg/
│   └── utils/
│       └── file.go          # 再利用可能なユーティリティ
├── testdata/                # テストデータ
├── .gitignore
├── go.mod                   # モジュール定義
├── go.sum                   # 依存関係のチェックサム
├── Makefile                 # ビルドスクリプト
└── README.md
```

## flagパッケージによる基本的なCLI

Goの標準ライブラリ`flag`パッケージを使った基本的なCLI実装を見ていきましょう。

### シンプルなCLI実装

```go
// cmd/mycli/main.go
package main

import (
    "flag"
    "fmt"
    "os"
)

var (
    name    string
    verbose bool
    output  string
)

func init() {
    flag.StringVar(&name, "name", "", "Project name (required)")
    flag.StringVar(&name, "n", "", "Project name (shorthand)")
    flag.BoolVar(&verbose, "verbose", false, "Enable verbose output")
    flag.BoolVar(&verbose, "v", false, "Enable verbose output (shorthand)")
    flag.StringVar(&output, "output", ".", "Output directory")
    flag.StringVar(&output, "o", ".", "Output directory (shorthand)")
}

func main() {
    flag.Parse()

    // Validate required flags
    if name == "" {
        fmt.Println("Error: --name is required")
        flag.Usage()
        os.Exit(1)
    }

    // Main logic
    if verbose {
        fmt.Printf("Creating project: %s\n", name)
        fmt.Printf("Output directory: %s\n", output)
    }

    if err := createProject(name, output); err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }

    fmt.Printf("✓ Project %s created successfully\n", name)
}

func createProject(name, output string) error {
    // Implementation
    fmt.Printf("Creating project %s in %s\n", name, output)
    return nil
}
```

使用例:

```bash
# ビルド
go build -o mycli cmd/mycli/main.go

# 実行
./mycli --name myproject --verbose
./mycli -n myproject -v -o ./projects
```

## サブコマンドの実装

flagパッケージを使ったサブコマンドの実装例です。

```go
// cmd/mycli/main.go
package main

import (
    "flag"
    "fmt"
    "os"
)

func main() {
    if len(os.Args) < 2 {
        printUsage()
        os.Exit(1)
    }

    // サブコマンドの定義
    createCmd := flag.NewFlagSet("create", flag.ExitOnError)
    createName := createCmd.String("name", "", "Project name")
    createTemplate := createCmd.String("template", "default", "Project template")

    listCmd := flag.NewFlagSet("list", flag.ExitOnError)
    listAll := listCmd.Bool("all", false, "Show all projects")

    deleteCmd := flag.NewFlagSet("delete", flag.ExitOnError)
    deleteName := deleteCmd.String("name", "", "Project name to delete")
    deleteForce := deleteCmd.Bool("force", false, "Force deletion")

    // サブコマンドの解析
    switch os.Args[1] {
    case "create":
        createCmd.Parse(os.Args[2:])
        if *createName == "" {
            fmt.Println("Error: --name is required")
            createCmd.Usage()
            os.Exit(1)
        }
        handleCreate(*createName, *createTemplate)

    case "list":
        listCmd.Parse(os.Args[2:])
        handleList(*listAll)

    case "delete":
        deleteCmd.Parse(os.Args[2:])
        if *deleteName == "" {
            fmt.Println("Error: --name is required")
            deleteCmd.Usage()
            os.Exit(1)
        }
        handleDelete(*deleteName, *deleteForce)

    default:
        fmt.Printf("Unknown command: %s\n", os.Args[1])
        printUsage()
        os.Exit(1)
    }
}

func printUsage() {
    fmt.Println("Usage: mycli <command> [options]")
    fmt.Println("\nCommands:")
    fmt.Println("  create    Create a new project")
    fmt.Println("  list      List all projects")
    fmt.Println("  delete    Delete a project")
    fmt.Println("\nUse 'mycli <command> -h' for more information about a command")
}

func handleCreate(name, template string) {
    fmt.Printf("Creating project: %s (template: %s)\n", name, template)
}

func handleList(all bool) {
    fmt.Printf("Listing projects (all: %v)\n", all)
}

func handleDelete(name string, force bool) {
    fmt.Printf("Deleting project: %s (force: %v)\n", name, force)
}
```

## 構造化されたプロジェクト実装

より実践的な構造化プロジェクトの例です。

### パッケージ構造

```go
// internal/project/project.go
package project

import (
    "fmt"
    "os"
    "path/filepath"
)

// Project represents a project
type Project struct {
    Name     string
    Template string
    Path     string
}

// Manager handles project operations
type Manager struct {
    baseDir string
}

// NewManager creates a new project manager
func NewManager(baseDir string) *Manager {
    return &Manager{baseDir: baseDir}
}

// Create creates a new project
func (m *Manager) Create(name, template string) (*Project, error) {
    projectPath := filepath.Join(m.baseDir, name)

    // Check if project already exists
    if _, err := os.Stat(projectPath); !os.IsNotExist(err) {
        return nil, fmt.Errorf("project already exists: %s", name)
    }

    // Create project directory
    if err := os.MkdirAll(projectPath, 0755); err != nil {
        return nil, fmt.Errorf("failed to create directory: %w", err)
    }

    project := &Project{
        Name:     name,
        Template: template,
        Path:     projectPath,
    }

    return project, nil
}

// List lists all projects
func (m *Manager) List() ([]*Project, error) {
    entries, err := os.ReadDir(m.baseDir)
    if err != nil {
        return nil, fmt.Errorf("failed to read directory: %w", err)
    }

    var projects []*Project
    for _, entry := range entries {
        if entry.IsDir() {
            projects = append(projects, &Project{
                Name: entry.Name(),
                Path: filepath.Join(m.baseDir, entry.Name()),
            })
        }
    }

    return projects, nil
}

// Delete deletes a project
func (m *Manager) Delete(name string) error {
    projectPath := filepath.Join(m.baseDir, name)

    if _, err := os.Stat(projectPath); os.IsNotExist(err) {
        return fmt.Errorf("project not found: %s", name)
    }

    if err := os.RemoveAll(projectPath); err != nil {
        return fmt.Errorf("failed to delete project: %w", err)
    }

    return nil
}
```

### メイン実装

```go
// cmd/mycli/main.go
package main

import (
    "flag"
    "fmt"
    "os"
    "path/filepath"

    "github.com/username/my-cli-tool/internal/project"
)

func main() {
    homeDir, _ := os.UserHomeDir()
    defaultBaseDir := filepath.Join(homeDir, "projects")

    if len(os.Args) < 2 {
        printUsage()
        os.Exit(1)
    }

    manager := project.NewManager(defaultBaseDir)

    switch os.Args[1] {
    case "create":
        handleCreate(manager, os.Args[2:])
    case "list":
        handleList(manager, os.Args[2:])
    case "delete":
        handleDelete(manager, os.Args[2:])
    default:
        fmt.Printf("Unknown command: %s\n", os.Args[1])
        printUsage()
        os.Exit(1)
    }
}

func handleCreate(manager *project.Manager, args []string) {
    cmd := flag.NewFlagSet("create", flag.ExitOnError)
    name := cmd.String("name", "", "Project name (required)")
    template := cmd.String("template", "default", "Project template")

    cmd.Parse(args)

    if *name == "" {
        fmt.Println("Error: --name is required")
        cmd.Usage()
        os.Exit(1)
    }

    proj, err := manager.Create(*name, *template)
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }

    fmt.Printf("✓ Project created: %s\n", proj.Name)
    fmt.Printf("  Path: %s\n", proj.Path)
    fmt.Printf("  Template: %s\n", proj.Template)
}

func handleList(manager *project.Manager, args []string) {
    projects, err := manager.List()
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }

    if len(projects) == 0 {
        fmt.Println("No projects found")
        return
    }

    fmt.Println("Projects:")
    for _, proj := range projects {
        fmt.Printf("  - %s (%s)\n", proj.Name, proj.Path)
    }
}

func handleDelete(manager *project.Manager, args []string) {
    cmd := flag.NewFlagSet("delete", flag.ExitOnError)
    name := cmd.String("name", "", "Project name (required)")
    force := cmd.Bool("force", false, "Force deletion without confirmation")

    cmd.Parse(args)

    if *name == "" {
        fmt.Println("Error: --name is required")
        cmd.Usage()
        os.Exit(1)
    }

    if !*force {
        fmt.Printf("Delete project '%s'? (y/N): ", *name)
        var response string
        fmt.Scanln(&response)
        if response != "y" && response != "Y" {
            fmt.Println("Cancelled")
            return
        }
    }

    if err := manager.Delete(*name); err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }

    fmt.Printf("✓ Project deleted: %s\n", *name)
}

func printUsage() {
    fmt.Println("Usage: mycli <command> [options]")
    fmt.Println("\nCommands:")
    fmt.Println("  create    Create a new project")
    fmt.Println("  list      List all projects")
    fmt.Println("  delete    Delete a project")
}
```

## Makefileによるビルド管理

```makefile
# Makefile
.PHONY: build install test clean

BINARY_NAME=mycli
GO=go
GOFLAGS=-ldflags="-s -w"

# ビルド
build:
	$(GO) build $(GOFLAGS) -o $(BINARY_NAME) cmd/mycli/main.go

# インストール
install: build
	cp $(BINARY_NAME) $(GOPATH)/bin/

# テスト
test:
	$(GO) test -v ./...

# カバレッジ
coverage:
	$(GO) test -coverprofile=coverage.out ./...
	$(GO) tool cover -html=coverage.out

# リント
lint:
	staticcheck ./...
	go vet ./...

# クリーン
clean:
	rm -f $(BINARY_NAME)
	rm -f coverage.out

# クロスコンパイル
build-all:
	GOOS=linux GOARCH=amd64 $(GO) build $(GOFLAGS) -o dist/$(BINARY_NAME)-linux-amd64 cmd/mycli/main.go
	GOOS=darwin GOARCH=amd64 $(GO) build $(GOFLAGS) -o dist/$(BINARY_NAME)-darwin-amd64 cmd/mycli/main.go
	GOOS=darwin GOARCH=arm64 $(GO) build $(GOFLAGS) -o dist/$(BINARY_NAME)-darwin-arm64 cmd/mycli/main.go
	GOOS=windows GOARCH=amd64 $(GO) build $(GOFLAGS) -o dist/$(BINARY_NAME)-windows-amd64.exe cmd/mycli/main.go
```

使用例:

```bash
# ビルド
make build

# インストール
make install

# テスト
make test

# すべてのプラットフォーム向けビルド
make build-all
```

## Go CLI開発セットアップチェックリスト

Go CLI開発を始める前に、以下を確認しましょう。

- [ ] Goがインストールされている（go version確認）
- [ ] GOPATH、GOBINが適切に設定されている
- [ ] プロジェクト構造が標準に従っている
- [ ] go.modファイルが作成されている
- [ ] Makefileでビルドタスクを定義している
- [ ] .gitignoreでバイナリを除外している
- [ ] internal/パッケージで内部ロジックを分離している
- [ ] pkg/パッケージで再利用可能なコードを分離している
- [ ] エラーハンドリングが適切に実装されている
- [ ] テストファイルが_test.goで作成されている

## まとめ

Goは、高速な実行速度と単一バイナリへのコンパイルという特徴から、CLIツール開発に最適な言語です。標準ライブラリのflagパッケージでも基本的なCLIを実装できますが、次章で学ぶCobraフレームワークを使用することで、より高機能なCLIツールを効率的に開発できます。

次章では、Go CLI開発における事実上の標準フレームワークであるCobraを学びます。

## 参考文献

- [Go公式ドキュメント](https://go.dev/doc/)
- [Effective Go](https://go.dev/doc/effective_go)
- [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)
- [Standard Go Project Layout](https://github.com/golang-standards/project-layout)
- [flag package - 公式ドキュメント](https://pkg.go.dev/flag)
