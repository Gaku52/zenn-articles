---
title: "インターフェース"
---

# Chapter 09: インターフェース

> 暗黙的実装と構造的型付け — Go最大の設計上の発明

## この章で学べること

- ✅ インターフェースの定義と実装
- ✅ 暗黙的実装（implements 宣言が不要）
- ✅ 標準ライブラリの重要なインターフェース
- ✅ 空インターフェース（any）
- ✅ 型アサーションと型スイッチ
- ✅ インターフェースの設計原則
- ✅ **nil インターフェースの落とし穴（重要）**

---

## インターフェースとは

インターフェースは**メソッドの集合**を定義します。そのメソッドを全て持つ型は、自動的にそのインターフェースを満たします。

```go
// インターフェースの定義
type Shape interface {
    Area() float64
}

// Rectangle は Area() を持つので、Shape を満たす
type Rectangle struct {
    Width, Height float64
}

func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

// Circle も Area() を持つので、Shape を満たす
type Circle struct {
    Radius float64
}

func (c Circle) Area() float64 {
    return math.Pi * c.Radius * c.Radius
}
```

### 使ってみる

```go
func printArea(s Shape) {
    fmt.Printf("面積: %.2f\n", s.Area())
}

func main() {
    rect := Rectangle{Width: 10, Height: 5}
    circle := Circle{Radius: 3}

    printArea(rect)    // 面積: 50.00
    printArea(circle)  // 面積: 28.27
}
```

`printArea` は `Shape` インターフェースを受け取るので、`Area()` メソッドを持つ**任意の型**を渡せます。

---

## 暗黙的実装 — Goの最大の特徴

Java や TypeScript では `implements` を明示的に宣言します：

```java
// Java
class Rectangle implements Shape { ... }
```

```typescript
// TypeScript
class Rectangle implements Shape { ... }
```

Goでは **`implements` の宣言は不要**です。メソッドが揃っていれば、自動的にインターフェースを満たします。

```go
// "implements" と書く場所はない
// Rectangle が Area() を持っている → Shape を満たす
// それだけ
```

この設計の利点：

1. **後からインターフェースを定義できる** — 既存の型を変更せずにインターフェースで抽象化できる
2. **パッケージ間の依存が減る** — インターフェースの定義元を知らなくても実装できる
3. **テストが簡単** — モックを作りやすい

---

## 標準ライブラリの重要なインターフェース

Goの標準ライブラリには、小さくて強力なインターフェースが多数あります。

### fmt.Stringer — 文字列表現

```go
type Stringer interface {
    String() string
}

type User struct {
    Name string
    Age  int
}

func (u User) String() string {
    return fmt.Sprintf("%s (%d歳)", u.Name, u.Age)
}

func main() {
    user := User{Name: "Alice", Age: 30}
    fmt.Println(user)  // Alice (30歳)
    // fmt.Println が自動で String() を呼ぶ
}
```

### io.Reader / io.Writer — 入出力の抽象化

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

ファイル、ネットワーク接続、バッファ、HTTP レスポンス — 全てが同じ `Reader` / `Writer` インターフェースを実装しています。

```go
// どんな Reader からでも読み取れる関数
func countLines(r io.Reader) (int, error) {
    scanner := bufio.NewScanner(r)
    count := 0
    for scanner.Scan() {
        count++
    }
    return count, scanner.Err()
}

func main() {
    // ファイルから
    f, _ := os.Open("data.txt")
    defer f.Close()
    n, _ := countLines(f)

    // 文字列から（同じ関数が使える）
    r := strings.NewReader("line1\nline2\nline3")
    n, _ = countLines(r)
}
```

### error — エラーもインターフェース

```go
type error interface {
    Error() string
}
```

`Error()` メソッドを持つ型は全て `error` として使えます。次の章で詳しく学びます。

---

## 空インターフェースと any

```go
// 空インターフェース = メソッドが0個 = すべての型が満たす
var anything interface{}
anything = 42
anything = "hello"
anything = true

// Go 1.18 以降は any と書ける（interface{} のエイリアス）
var value any = 42
```

`any` は「どんな型でも受け取れる」ことを意味します。`fmt.Println` が任意の値を受け取れるのはこの仕組みです。

ただし、`any` を多用するとGoの型安全性が失われます。**できるだけ具体的な型やインターフェースを使いましょう**。

---

## 型アサーション

インターフェースから具体的な型を取り出す方法です。

```go
var s Shape = Circle{Radius: 5}

// 型アサーション
c, ok := s.(Circle)
if ok {
    fmt.Println("半径:", c.Radius)  // 半径: 5
}

// 失敗する場合
r, ok := s.(Rectangle)
if !ok {
    fmt.Println("Rectangle ではありません")
}
```

### 型スイッチ

```go
func describe(s Shape) string {
    switch v := s.(type) {
    case Circle:
        return fmt.Sprintf("半径%fの円", v.Radius)
    case Rectangle:
        return fmt.Sprintf("%fx%fの四角形", v.Width, v.Height)
    default:
        return "不明な図形"
    }
}
```

---

## インターフェースの設計原則

### 小さく保つ

Go Proverb: **The bigger the interface, the weaker the abstraction.**（インターフェースが大きいほど、抽象化は弱くなる）

```go
// ✅ 良い例: 小さなインターフェース
type Reader interface {
    Read(p []byte) (n int, err error)
}

// ❌ 悪い例: 大きすぎるインターフェース
type FileSystem interface {
    Open(name string) (File, error)
    Create(name string) (File, error)
    Remove(name string) error
    Rename(oldpath, newpath string) error
    Stat(name string) (FileInfo, error)
    ReadDir(name string) ([]DirEntry, error)
    // ... 多すぎる
}
```

標準ライブラリのインターフェースは1-2メソッドが主流です。

### 消費側で定義する

インターフェースは**使う側のパッケージ**で定義するのがGoの慣習です。

```go
// ❌ provider パッケージでインターフェースを定義
package provider
type UserRepository interface { ... }

// ✅ consumer パッケージで必要なメソッドだけ定義
package handler
type UserGetter interface {
    GetUser(id int) (*User, error)
}
```

---

## 【重要】nil インターフェース vs nil を持つインターフェース

Goで最も有名な落とし穴の一つです。**他の言語にはない概念**なので注意してください。

### インターフェースの内部構造

Goのインターフェース変数は内部に2つの情報を持っています：

```
インターフェース変数
┌──────────────┐
│ 型情報 (T)   │  ← どの型の値が入っているか
│ 値 (V)       │  ← 実際の値（またはポインタ）
└──────────────┘

インターフェースが nil になるのは、T も V も両方 nil のときだけ
```

### 具体例で理解する

```go
// ケース1: 完全な nil（インターフェース自体が nil）
var s Shape          // T=nil, V=nil → s == nil は true
fmt.Println(s == nil)  // true ✅

// ケース2: nil ポインタをインターフェースに代入
var c *Circle = nil  // nil ポインタ
s = c                // T=*Circle, V=nil → s == nil は false！
fmt.Println(s == nil)  // false ⚠️
```

**図解:**

```
ケース1: var s Shape
  s: [ T=nil | V=nil ]  → nil と判定される ✅

ケース2: var c *Circle = nil; s = c
  s: [ T=*Circle | V=nil ]  → nil ではないと判定される ⚠️
      ↑
      型情報が入っているため nil ではない
```

### エラー処理での落とし穴

この問題が最も頻繁に現れるのはエラーを返す関数です：

```go
type MyError struct {
    Message string
}

func (e *MyError) Error() string {
    return e.Message
}

// ❌ 危険なパターン
func riskyFunc(bad bool) error {
    var err *MyError  // nil ポインタ（型は *MyError）
    if bad {
        err = &MyError{"something went wrong"}
    }
    return err  // bad=false でも nil ではない error が返る！
                // 内部: T=*MyError, V=nil → error として nil ではない
}

func main() {
    err := riskyFunc(false)
    if err != nil {
        fmt.Println("エラー:", err)  // ← ここに入ってしまう！
    }
}
```

```go
// ✅ 安全なパターン
func safeFunc(bad bool) error {
    if bad {
        return &MyError{"something went wrong"}
    }
    return nil  // 明示的に nil を返す（T=nil, V=nil）
}
```

### ルール

> `error` を返す関数では、必ず**明示的に `nil` を返す**こと。
> `var err *MyError` のような型付きの nil 変数を `return err` で返してはいけない。

---

## やってみよう

1. `Stringer` インターフェースを実装して、自分の構造体を `fmt.Println` で表示してみましょう
2. `Shape` インターフェースに `Perimeter() float64` を追加し、Circle と Rectangle の両方で実装してみましょう
3. 型スイッチを使って、`any` 型の値の型を判定する関数を書いてみましょう
4. nil インターフェースの落とし穴を実際に確認してみましょう（`riskyFunc` を試してみましょう）

---

次の章では、Goのエラーハンドリングを学びます。error インターフェースが中心です。
