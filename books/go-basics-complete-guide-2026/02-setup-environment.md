---
title: "環境構築"
---

# Chapter 02: 環境構築

> Goをインストールして最初のプログラムを動かす

## この章で学べること

- ✅ Goのインストール方法（macOS / Windows / Linux）
- ✅ VS Code + Go拡張の設定
- ✅ go mod によるプロジェクト作成
- ✅ Hello World の実行

---

## Goのインストール

### macOS（Homebrew）

```bash
# Homebrewでインストール（推奨）
$ brew install go

# バージョン確認
$ go version
go version go1.24.0 darwin/arm64
```

### Windows

```bash
# wingetでインストール
$ winget install GoLang.Go

# または公式サイトからインストーラをダウンロード
# https://go.dev/dl/
```

### Linux（Ubuntu / Debian）

```bash
# 公式のtar.gzからインストール
$ wget https://go.dev/dl/go1.24.0.linux-amd64.tar.gz
$ sudo rm -rf /usr/local/go
$ sudo tar -C /usr/local -xzf go1.24.0.linux-amd64.tar.gz

# PATHに追加（~/.bashrc または ~/.zshrc）
$ echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
$ source ~/.bashrc

$ go version
```

### インストール確認

```bash
$ go version
go version go1.24.0 darwin/arm64

$ go env GOPATH
/Users/yourname/go

$ go env GOROOT
/usr/local/go
```

---

## VS Code の設定

### 1. Go拡張のインストール

VS Code を開き、拡張機能から **Go**（Go Team at Google）をインストールします。

### 2. Go ツールのインストール

VS Code を開いた状態で `Cmd+Shift+P`（Windows: `Ctrl+Shift+P`）→ `Go: Install/Update Tools` を実行し、すべてのツールにチェックを入れてインストールします。

主要ツール：
- **gopls** — 言語サーバー（補完、定義ジャンプ、リファクタリング）
- **dlv** — デバッガ
- **staticcheck** — 静的解析
- **goimports** — インポート自動整理

### 3. 保存時の自動フォーマット

VS Code の `settings.json` に以下を追加：

```json
{
  "[go]": {
    "editor.formatOnSave": true,
    "editor.defaultFormatter": "golang.go",
    "editor.codeActionsOnSave": {
      "source.organizeImports": "explicit"
    }
  }
}
```

これで保存するたびに `gofmt` と `goimports` が自動実行されます。コードスタイルの議論は不要 — Goでは全員が同じフォーマットを使います。

---

## 最初のプロジェクトを作る

### go mod init

```bash
# プロジェクトディレクトリを作成
$ mkdir hello-go
$ cd hello-go

# モジュールの初期化
$ go mod init hello-go
```

`go.mod` ファイルが生成されます：

```
module hello-go

go 1.24
```

### Hello World

`main.go` を作成します：

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, Go!")
}
```

### 実行

```bash
# そのまま実行
$ go run main.go
Hello, Go!

# ビルドしてからバイナリを実行
$ go build -o hello
$ ./hello
Hello, Go!
```

`go run` は内部的にコンパイル→実行を行います。`go build` はバイナリファイルを生成します。

---

## プロジェクト構成の基本

```
hello-go/
├── go.mod       # モジュール定義（依存管理）
├── go.sum       # 依存のチェックサム（自動生成）
└── main.go      # エントリーポイント
```

### package main とは

```go
package main  // このパッケージは実行可能プログラムであることを示す

func main() {  // プログラムのエントリーポイント
    // ここから実行が始まる
}
```

- `package main` — 実行可能なプログラムの宣言
- `func main()` — プログラムの開始点（引数なし、戻り値なし）
- 1つのプログラムに `main` パッケージと `main` 関数は1つだけ

---

## よく使う go コマンド

| コマンド | 用途 |
|---------|------|
| `go run main.go` | コンパイル＆実行 |
| `go build` | バイナリをビルド |
| `go mod init <名前>` | 新しいモジュールを初期化 |
| `go mod tidy` | 不要な依存を削除、必要な依存を追加 |
| `go fmt ./...` | コード全体をフォーマット |
| `go vet ./...` | コードの問題を静的解析 |
| `go test ./...` | テストを実行 |
| `go doc fmt.Println` | ドキュメントを表示 |

---

## Go Playground

ブラウザ上でGoを試せる公式ツールがあります：

**https://go.dev/play/**

インストール不要で、コードの共有もできます。ちょっとした動作確認に便利です。

---

## やってみよう

1. `go version` でGoのバージョンを確認してみましょう
2. `hello-go` プロジェクトを作成し、`Hello, Go!` を表示させてみましょう
3. 表示するメッセージを変えて `go run` してみましょう
4. `go build` でバイナリを作成し、直接実行してみましょう

---

次の章では、Goの基本構文と型システムを学びます。
