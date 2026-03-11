---
title: "パッケージとモジュール"
---

# Chapter 11: パッケージとモジュール

> コードの整理、依存管理、プロジェクト構成

## この章で学べること

- ✅ パッケージの仕組みとインポート
- ✅ エクスポート規則（大文字 / 小文字）
- ✅ go mod による依存管理
- ✅ プロジェクトのディレクトリ構成
- ✅ internal パッケージ
- ✅ テストの書き方（go test）

---

## パッケージとは

Goのコードは**パッケージ**単位で整理されます。1つのディレクトリ = 1つのパッケージです。

```
myapp/
├── go.mod
├── main.go          # package main
└── math/
    └── calc.go      # package math
```

```go
// math/calc.go
package math

func Add(a, b int) int {
    return a + b
}

func subtract(a, b int) int {  // 小文字 = 非公開
    return a - b
}
```

```go
// main.go
package main

import (
    "fmt"
    "myapp/math"
)

func main() {
    fmt.Println(math.Add(3, 5))  // 8
    // math.subtract(3, 5)       // エラー: 非公開関数
}
```

### エクスポート規則

| 先頭文字 | アクセス | 例 |
|---------|--------|-----|
| **大文字** | どこからでもアクセス可能 | `Add`, `User`, `MaxRetries` |
| **小文字** | 同じパッケージ内のみ | `add`, `user`, `maxRetries` |

これは関数、型、変数、定数、構造体のフィールド、メソッド — 全てに適用されます。

---

## import の書き方

```go
import (
    // 標準ライブラリ
    "fmt"
    "os"
    "strings"

    // 外部パッケージ（空行で区切る）
    "github.com/gin-gonic/gin"

    // プロジェクト内パッケージ（空行で区切る）
    "myapp/internal/config"
)
```

### エイリアス

```go
import (
    "fmt"
    mrand "math/rand"    // エイリアスをつけてインポート
    crand "crypto/rand"  // 名前の衝突を回避
)

func main() {
    fmt.Println(mrand.Intn(100))
}
```

### ブランクインポート

```go
import (
    "database/sql"
    _ "github.com/lib/pq"  // init() を実行するためだけのインポート
)
```

`_` でインポートすると、パッケージの `init()` 関数だけが実行されます。データベースドライバの登録などで使われます。

---

## go mod — モジュール管理

### モジュールの初期化

```bash
$ mkdir myapp
$ cd myapp
$ go mod init github.com/yourname/myapp
```

`go.mod` が生成されます：

```
module github.com/yourname/myapp

go 1.23.0
```

### 外部パッケージの追加

```bash
# コードで import して go mod tidy を実行
$ go mod tidy

# または直接追加
$ go get github.com/gin-gonic/gin@latest
```

`go.mod` に依存が追加され、`go.sum` にチェックサムが記録されます。

### よく使う go mod コマンド

| コマンド | 用途 |
|---------|------|
| `go mod init <名前>` | モジュールを初期化 |
| `go mod tidy` | 不要な依存を削除、必要な依存を追加 |
| `go mod download` | 依存をダウンロード |
| `go get <パッケージ>@<バージョン>` | パッケージを追加・更新 |
| `go list -m all` | 全依存を一覧表示 |

---

## プロジェクト構成

### シンプルなプロジェクト

```
myapp/
├── go.mod
├── go.sum
├── main.go
├── handler.go
└── handler_test.go
```

小さなプロジェクトは、ディレクトリを分けずにフラットに置くのがGoらしいです。

### 中規模プロジェクト

```
myapp/
├── go.mod
├── go.sum
├── main.go
├── internal/          # このモジュール内でのみ使える
│   ├── config/
│   │   └── config.go
│   ├── handler/
│   │   ├── handler.go
│   │   └── handler_test.go
│   └── model/
│       └── user.go
├── pkg/               # 外部から利用可能なパッケージ
│   └── validator/
│       └── validator.go
└── cmd/               # 複数の実行可能バイナリ
    ├── server/
    │   └── main.go
    └── cli/
        └── main.go
```

### internal パッケージ

`internal` ディレクトリ内のパッケージは、そのモジュール（またはその親ディレクトリ）の外部からインポートできません。Goのコンパイラが強制するアクセス制御です。

```go
// ✅ myapp 内からは使える
import "myapp/internal/config"

// ❌ 他のモジュールからは使えない（コンパイルエラー）
import "github.com/yourname/myapp/internal/config"
```

---

## テストの書き方

Goにはテストフレームワークが**標準搭載**されています。

### テストファイルの命名規則

- テストファイルは `_test.go` で終わる
- テスト対象と同じディレクトリに置く
- テスト関数は `Test` で始まる

```
math/
├── calc.go        # 本体
└── calc_test.go   # テスト
```

### テストの書き方

```go
// math/calc_test.go
package math

import "testing"

func TestAdd(t *testing.T) {
    result := Add(3, 5)
    if result != 8 {
        t.Errorf("Add(3, 5) = %d, want 8", result)
    }
}

func TestAddNegative(t *testing.T) {
    result := Add(-1, -2)
    if result != -3 {
        t.Errorf("Add(-1, -2) = %d, want -3", result)
    }
}
```

### テーブル駆動テスト

Goで最も一般的なテストパターンです。

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive", 3, 5, 8},
        {"negative", -1, -2, -3},
        {"zero", 0, 0, 0},
        {"mixed", -5, 3, -2},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Add(tt.a, tt.b)
            if result != tt.expected {
                t.Errorf("Add(%d, %d) = %d, want %d",
                    tt.a, tt.b, result, tt.expected)
            }
        })
    }
}
```

### テストの実行

```bash
# カレントパッケージのテスト
$ go test

# 全パッケージのテスト
$ go test ./...

# 詳細出力
$ go test -v ./...

# 特定のテストだけ実行
$ go test -run TestAdd

# カバレッジ
$ go test -cover ./...
```

---

## やってみよう

1. `calculator` パッケージを作り、`Add`, `Subtract`, `Multiply`, `Divide` を実装してみましょう
2. 各関数のテーブル駆動テストを書いて `go test -v` で実行してみましょう
3. `internal` パッケージを使って、外部からアクセスできないヘルパー関数を作ってみましょう
4. `go mod tidy` で依存を整理してみましょう

---

次の章では、この本のまとめと次のステップを紹介します。
