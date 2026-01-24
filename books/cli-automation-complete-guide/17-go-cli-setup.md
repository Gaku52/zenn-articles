# Go CLIセットアップ

本章では、Goでパフォーマンスの高いCLIツールを開発するためのセットアップを学びます。

## プロジェクト初期化

```bash
mkdir my-cli-tool
cd my-cli-tool
go mod init github.com/username/my-cli-tool
```

## 依存関係

```bash
# Cobraインストール
go get -u github.com/spf13/cobra@latest

# Viperインストール（設定管理）
go get -u github.com/spf13/viper@latest
```

## ディレクトリ構造

```
my-cli-tool/
├── cmd/
│   ├── root.go
│   ├── create.go
│   ├── list.go
│   └── delete.go
├── internal/
│   ├── project/
│   └── template/
├── main.go
├── go.mod
└── go.sum
```

## エントリーポイント

```go
// main.go
package main

import "github.com/username/my-cli-tool/cmd"

func main() {
    cmd.Execute()
}
```

## まとめ

Goの高速性とシンプルさにより、パフォーマンスの高いCLIツールを開発できます。
