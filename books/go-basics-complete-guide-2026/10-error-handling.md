---
title: "エラーハンドリング"
---

# Chapter 10: エラーハンドリング

> Errors are values. — エラーは値である

## この章で学べること

- ✅ error インターフェースの仕組み
- ✅ エラーの作成（errors.New, fmt.Errorf）
- ✅ エラーのラップとアンラップ
- ✅ errors.Is と errors.As
- ✅ カスタムエラー型
- ✅ panic と recover（使うべきでない場面）

---

## Goのエラー処理の考え方

多くの言語では**例外**を投げてエラーを処理します：

```python
# Python
try:
    result = divide(10, 0)
except ZeroDivisionError:
    print("ゼロ除算エラー")
```

Goには例外がありません。代わりに**戻り値としてエラーを返します**：

```go
result, err := divide(10, 0)
if err != nil {
    fmt.Println("エラー:", err)
    return
}
fmt.Println(result)
```

「エラーは特別なものではなく、普通の値として扱う」— これが Go Proverb「**Errors are values**」の意味です。

---

## error インターフェース

```go
// 標準ライブラリで定義されている
type error interface {
    Error() string
}
```

`Error()` メソッドを持つ型は全て `error` として使えます。

### エラーの作成

```go
import "errors"

// 方法1: errors.New
err := errors.New("something went wrong")

// 方法2: fmt.Errorf（フォーマット付き）
name := "config.yaml"
err := fmt.Errorf("file not found: %s", name)
```

---

## 基本的なエラー処理パターン

```go
func readConfig(path string) ([]byte, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("failed to read config: %w", err)
    }
    return data, nil
}

func main() {
    data, err := readConfig("config.yaml")
    if err != nil {
        log.Fatal(err)  // エラーを表示して終了
    }
    fmt.Println(string(data))
}
```

### エラー処理の3つの選択肢

```go
// 1. 呼び出し元に返す（最も一般的）
if err != nil {
    return fmt.Errorf("context: %w", err)
}

// 2. ログに記録して処理を続行
if err != nil {
    log.Printf("warning: %v", err)
    // 処理を続ける
}

// 3. プログラムを終了（main関数やどうしようもない場合のみ）
if err != nil {
    log.Fatal(err)
}
```

---

## エラーのラップ（%w）

`fmt.Errorf` で `%w` を使うと、元のエラーを**ラップ**できます。

```go
func openDatabase(dsn string) (*sql.DB, error) {
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        // %w でエラーをラップ — 元のエラー情報を保持
        return nil, fmt.Errorf("failed to open database: %w", err)
    }
    return db, nil
}
```

ラップすることで、エラーに**コンテキスト（文脈情報）**を追加しながら、元のエラーの種類を後で判定できます。

---

## errors.Is — エラーの種類を判定

```go
import "errors"

var ErrNotFound = errors.New("not found")
var ErrPermission = errors.New("permission denied")

func findUser(id int) (*User, error) {
    // ...
    return nil, fmt.Errorf("user %d: %w", id, ErrNotFound)
}

func main() {
    _, err := findUser(42)

    // ラップされたエラーでも判定できる
    if errors.Is(err, ErrNotFound) {
        fmt.Println("ユーザーが見つかりません")
    } else if errors.Is(err, ErrPermission) {
        fmt.Println("権限がありません")
    }
}
```

`errors.Is` はラップの連鎖をたどって、元のエラーと一致するか調べます。`==` ではラップされたエラーを判定できません。

---

## errors.As — エラーの型を取得

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error: %s - %s", e.Field, e.Message)
}

func validateAge(age int) error {
    if age < 0 {
        return &ValidationError{
            Field:   "age",
            Message: "must be non-negative",
        }
    }
    return nil
}

func main() {
    err := validateAge(-1)

    var ve *ValidationError
    if errors.As(err, &ve) {
        fmt.Printf("フィールド: %s, メッセージ: %s\n", ve.Field, ve.Message)
        // フィールド: age, メッセージ: must be non-negative
    }
}
```

`errors.Is` は「このエラーか？」、`errors.As` は「この型のエラーか？」を判定します。

---

## カスタムエラー型

```go
type HTTPError struct {
    StatusCode int
    Message    string
}

func (e *HTTPError) Error() string {
    return fmt.Sprintf("HTTP %d: %s", e.StatusCode, e.Message)
}

func fetchData(url string) ([]byte, error) {
    resp, err := http.Get(url)
    if err != nil {
        return nil, fmt.Errorf("request failed: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != 200 {
        return nil, &HTTPError{
            StatusCode: resp.StatusCode,
            Message:    "unexpected status",
        }
    }

    return io.ReadAll(resp.Body)
}
```

---

## panic と recover

### panic — プログラムのクラッシュ

```go
func mustParseInt(s string) int {
    n, err := strconv.Atoi(s)
    if err != nil {
        panic(fmt.Sprintf("invalid integer: %s", s))
    }
    return n
}
```

Go Proverb: **Don't panic.** — panic は「プログラムのバグ」や「回復不能な状態」でのみ使います。通常のエラー処理には使いません。

### panic を使ってよい場面

- プログラムの初期化時に必須の設定が欠けている
- 「ここに来るはずがない」ロジックエラー
- `Must` プレフィックスの関数（`regexp.MustCompile` など）

### recover — panic の回復

```go
func safeDiv(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered: %v", r)
        }
    }()

    return a / b, nil  // b=0 なら panic が発生
}

func main() {
    result, err := safeDiv(10, 0)
    if err != nil {
        fmt.Println(err)  // recovered: runtime error: integer divide by zero
    }
}
```

`recover` は `defer` の中でのみ有効です。実務ではHTTPサーバーのミドルウェアなどで使われますが、自分で書く機会は少ないです。

---

## エラー処理のベストプラクティス

```go
// ✅ エラーにコンテキストを追加する
return fmt.Errorf("failed to create user %s: %w", name, err)

// ❌ エラーをそのまま返すだけ（デバッグしにくい）
return err

// ✅ センチネルエラーで判定可能にする
var ErrNotFound = errors.New("not found")

// ❌ エラーメッセージの文字列比較（壊れやすい）
if err.Error() == "not found" { ... }

// ✅ エラーは小文字で始める（他のエラーと結合されるため）
fmt.Errorf("open file: %w", err)

// ❌ 大文字で始める
fmt.Errorf("Open file failed: %w", err)
```

---

## やってみよう

1. `divide(a, b float64) (float64, error)` を書き、ゼロ除算をエラーとして返してみましょう
2. カスタムエラー型 `ValidationError` を作り、`errors.As` で取り出してみましょう
3. `%w` でエラーをラップし、`errors.Is` でセンチネルエラーを判定してみましょう

---

次の章では、パッケージとモジュールの管理を学びます。
