---
title: "次のステップ"
---

# Chapter 12: 次のステップ

> 基礎を固めた次に進むべき道

## この本で学んだこと

お疲れさまでした。ここまでで、Go言語の基礎を体系的に学びました。

```
✅ Goの設計哲学 — シンプルさ・明示性・実用性
✅ 基本構文 — 変数、定数、型、:=
✅ 制御フロー — if, for, switch, defer
✅ 関数 — 複数戻り値、クロージャ、可変長引数
✅ データ構造 — 配列、スライス、マップ
✅ ポインタ — 値渡し、参照、メモリモデル
✅ 構造体とメソッド — struct、レシーバ、コンポジション
✅ インターフェース — 暗黙的実装、構造的部分型
✅ エラーハンドリング — error、errors.Is/As、panic/recover
✅ パッケージとモジュール — go mod、テスト
```

これだけの知識があれば、Goのコードを**読んで理解**し、**基本的なプログラムを書く**ことができます。

---

## 次に学ぶべきトピック

### 1. 並行処理（最重要）

Goの真の強みは並行処理です。基礎を終えた今が学ぶタイミングです。

- **goroutine** — 軽量スレッド
- **channel** — goroutine間の通信
- **sync パッケージ** — Mutex, WaitGroup, Once
- **context** — キャンセル、タイムアウト、リクエストスコープ
- **並行処理パターン** — Worker Pool, Fan-out/Fan-in, Pipeline

```go
// 予告: 並行処理の世界
func main() {
    ch := make(chan string)

    go func() {
        ch <- "Hello from goroutine!"
    }()

    msg := <-ch
    fmt.Println(msg)
}
```

### 2. Web開発

GoはWebサーバー開発に最も使われている言語の一つです。

- **net/http** — 標準ライブラリだけでWebサーバーが作れる
- **Gin / Echo** — 人気のWebフレームワーク
- **database/sql** — データベース接続
- **gRPC** — マイクロサービス間通信

```go
// 予告: 標準ライブラリだけでWebサーバー（Go 1.22+ のルーティング構文）
func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("GET /hello", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "Hello, Web!")
    })
    http.ListenAndServe(":8080", mux)
}
```

### 3. CLIツール開発

Goはコマンドラインツールの開発に最適です。

- **cobra** — CLI フレームワーク（kubectl, gh で使われている）
- **viper** — 設定管理
- **クロスコンパイル** — 1コマンドで全OS向けバイナリを生成

```bash
# macOS で Windows 向けバイナリをビルド
$ GOOS=windows GOARCH=amd64 go build -o myapp.exe
```

### 4. テストの深掘り

- **ベンチマークテスト** — `func BenchmarkXxx(b *testing.B)`
- **テストヘルパー** — `t.Helper()`, `t.Cleanup()`
- **ゴールデンファイルテスト** — 期待出力をファイルで管理
- **httptest** — HTTPハンドラのテスト

### 5. ジェネリクス（Go 1.18+）

Go 1.21+ では `min` / `max` がビルトイン関数として追加されました。

```go
// ビルトイン関数（Go 1.21+）
fmt.Println(min(3, 5))       // 3
fmt.Println(max(3.14, 2.71)) // 3.14
```

自分でジェネリック関数を定義する場合は `cmp.Ordered`（標準ライブラリ）を使います。

```go
import "cmp"

// 任意の順序付き型に対応する関数
func Clamp[T cmp.Ordered](v, lo, hi T) T {
    if v < lo {
        return lo
    }
    if v > hi {
        return hi
    }
    return v
}

fmt.Println(Clamp(15, 0, 10))  // 10
fmt.Println(Clamp(-5, 0, 10))  // 0
```

---

## 推奨リソース

### 公式

- **A Tour of Go** — https://go.dev/tour/ — 対話型チュートリアル
- **Effective Go** — https://go.dev/doc/effective_go — Goらしいコードの書き方
- **Go Blog** — https://go.dev/blog/ — 公式ブログ
- **Go Playground** — https://go.dev/play/ — ブラウザでGoを試す

### 書籍

- **「Go言語による並行処理」** — 並行処理のバイブル
- **「プログラミング言語Go」** — 言語仕様の網羅的解説

### 実践プロジェクトのアイデア

| プロジェクト | 学べること |
|-------------|----------|
| TODOリストCLI | cobra, ファイルI/O, JSON |
| URLショートナー | net/http, データベース |
| チャットサーバー | goroutine, channel, WebSocket |
| ファイル同期ツール | concurrency, ファイル操作 |
| REST API サーバー | Gin/Echo, ミドルウェア, テスト |

---

## Go実践開発ガイド（姉妹本）

本書で学んだ基礎の上に、並行処理・Web開発・CLI開発・本番運用までをカバーする**実践編**を別途用意しています。

基礎を固めた方は、ぜひ次のステップとしてご活用ください。

---

## 最後に

Goの設計思想は「**シンプルさは複雑さの先にある**」です。

Goを学び始めると「なぜこの機能がないの？」と感じることがあるかもしれません。しかし使い込むうちに、その「ないこと」こそがGoの強みだと分かるようになります。

**Clear is better than clever.** — 賢いコードではなく、明快なコードを。

Goの世界を楽しんでください。

---

ご質問やフィードバックがあれば、Zennのコメント欄でお気軽にどうぞ。
