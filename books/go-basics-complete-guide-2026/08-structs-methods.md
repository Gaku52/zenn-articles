---
title: "構造体とメソッド"
---

# Chapter 08: 構造体とメソッド

> struct、メソッド、レシーバ — Goのオブジェクト指向

## この章で学べること

- ✅ 構造体（struct）の定義と初期化
- ✅ メソッドとレシーバ
- ✅ 値レシーバとポインタレシーバの使い分け
- ✅ 構造体の埋め込み（コンポジション）
- ✅ コンストラクタパターン

---

## Goにクラスはない

Goにはクラス、継承、コンストラクタという概念がありません。代わりに：

- **構造体（struct）** でデータを定義
- **メソッド** で振る舞いを定義
- **インターフェース**（次章）で抽象化
- **埋め込み** で再利用

このシンプルな組み合わせが、Goのオブジェクト指向です。

---

## 構造体の定義と初期化

```go
// 構造体の定義
type User struct {
    Name  string
    Email string
    Age   int
}

func main() {
    // 初期化方法1: フィールド名を指定（推奨）
    alice := User{
        Name:  "Alice",
        Email: "alice@example.com",
        Age:   30,
    }

    // 初期化方法2: 順番で指定（非推奨 — フィールド追加時に壊れる）
    bob := User{"Bob", "bob@example.com", 25}

    // 初期化方法3: ゼロ値（全フィールドがゼロ値）
    var empty User  // Name: "", Email: "", Age: 0

    // フィールドへのアクセス
    fmt.Println(alice.Name)   // Alice
    alice.Age = 31            // 変更も可能
}
```

### ポインタで構造体を作成

```go
// & をつけるとポインタが返る
user := &User{
    Name: "Alice",
    Age:  30,
}
// user の型は *User

// ポインタでもドットでアクセスできる（自動間接参照）
fmt.Println(user.Name)  // Alice（(*user).Name と書く必要なし）
```

---

## メソッド

メソッドは**レシーバ**を持つ関数です。構造体に振る舞いを追加します。

```go
type Rectangle struct {
    Width  float64
    Height float64
}

// メソッドの定義
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

func (r Rectangle) Perimeter() float64 {
    return 2 * (r.Width + r.Height)
}

func main() {
    rect := Rectangle{Width: 10, Height: 5}
    fmt.Println(rect.Area())       // 50
    fmt.Println(rect.Perimeter())  // 30
}
```

`(r Rectangle)` の部分が**レシーバ**です。このメソッドは `Rectangle` 型に属することを示します。

---

## 値レシーバとポインタレシーバ

レシーバの扱いは、前章で学んだ「値渡し」の原則そのものです。

- **値レシーバ** = 構造体のコピーを受け取る（値渡し）
- **ポインタレシーバ** = 構造体のアドレスを受け取る（ポインタ渡し）

### 値レシーバ — 読み取り専用

```go
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
    // r はコピー。変更しても元には影響しない
}
```

### ポインタレシーバ — 変更可能

```go
func (r *Rectangle) Scale(factor float64) {
    r.Width *= factor
    r.Height *= factor
    // r はポインタ。元の構造体が変更される
}

func main() {
    rect := Rectangle{Width: 10, Height: 5}
    rect.Scale(2)
    fmt.Println(rect.Width)   // 20
    fmt.Println(rect.Height)  // 10
}
```

> **自動変換**: `rect.Scale(2)` は Goが自動で `(&rect).Scale(2)` に変換します。明示的に `&` を書く必要はありません。

> **他言語との比較**: Java のメソッドレシーバは常にポインタ（参照）ですが、その性質が隠蔽されています。Go では値レシーバとポインタレシーバを明示的に選択できるため、動作が予測しやすくなっています。

### 使い分けの基準

| 条件 | 値レシーバ | ポインタレシーバ |
|------|-----------|----------------|
| フィールドを変更する | - | ✅ |
| 構造体が大きい（コピーコストを避けたい） | - | ✅ |
| 読み取りだけ & 構造体が小さい | ✅ | - |
| 一貫性（他のメソッドがポインタ） | - | ✅ |

**実務のルール**: 1つでもポインタレシーバがある型は、**全てのメソッドをポインタレシーバに統一**するのが一般的です。迷ったらポインタレシーバを選びましょう。

---

## コンストラクタパターン

Goにはコンストラクタ構文がありません。代わりに `New` で始まる関数を慣習的に使います。

```go
type Server struct {
    host    string
    port    int
    timeout time.Duration
}

// コンストラクタ関数
func NewServer(host string, port int) *Server {
    return &Server{
        host:    host,
        port:    port,
        timeout: 30 * time.Second,  // デフォルト値
    }
}

func main() {
    s := NewServer("localhost", 8080)
    fmt.Println(s.host)  // localhost
}
```

### バリデーション付きコンストラクタ

```go
func NewUser(name, email string) (*User, error) {
    if name == "" {
        return nil, fmt.Errorf("name is required")
    }
    if !strings.Contains(email, "@") {
        return nil, fmt.Errorf("invalid email: %s", email)
    }
    return &User{Name: name, Email: email}, nil
}
```

---

## 構造体の埋め込み（コンポジション）

Goでは継承の代わりに**埋め込み**を使います。

```go
// 基本の構造体
type Animal struct {
    Name string
}

func (a Animal) Speak() string {
    return a.Name + " says ..."
}

// 埋め込み（Animal のフィールドとメソッドを「持つ」）
type Dog struct {
    Animal          // 型名だけ書く = 埋め込み
    Breed  string
}

func main() {
    d := Dog{
        Animal: Animal{Name: "Pochi"},
        Breed:  "Shiba",
    }

    // Animal のフィールドに直接アクセスできる
    fmt.Println(d.Name)     // Pochi（d.Animal.Name と同じ）
    fmt.Println(d.Speak())  // Pochi says ...
    fmt.Println(d.Breed)    // Shiba
}
```

### メソッドのオーバーライド

```go
// Dog 独自の Speak を定義すると上書きされる
func (d Dog) Speak() string {
    return d.Name + " says Woof!"
}

func main() {
    d := Dog{Animal: Animal{Name: "Pochi"}}
    fmt.Println(d.Speak())         // Pochi says Woof!
    fmt.Println(d.Animal.Speak())  // Pochi says ...（元のメソッドも呼べる）
}
```

埋め込みは「継承」ではなく「**has-a**（持つ）」関係です。Dog は Animal を**持っている**のであって、Animal **である**わけではありません。

---

## エクスポートとアクセス制御

Goでは**先頭が大文字ならエクスポート（公開）、小文字なら非公開**です。

```go
type User struct {
    Name  string  // 大文字 = 他のパッケージからアクセス可能
    Email string  // 大文字 = 公開
    age   int     // 小文字 = このパッケージ内のみ
}
```

クラスの `public` / `private` に相当しますが、修飾子ではなく**命名規則**で制御するのがGoの特徴です。

---

## タグ

構造体フィールドには**タグ**をつけることができます。JSONのシリアライズ/デシリアライズでよく使います。

```go
type User struct {
    Name  string `json:"name"`
    Email string `json:"email"`
    Age   int    `json:"age,omitempty"`
}

func main() {
    user := User{Name: "Alice", Email: "alice@example.com"}

    // 構造体 → JSON
    data, _ := json.Marshal(user)
    fmt.Println(string(data))
    // {"name":"Alice","email":"alice@example.com"}
    // Age は omitempty なのでゼロ値（0）の場合は省略される

    // JSON → 構造体
    var u User
    json.Unmarshal([]byte(`{"name":"Bob","age":25}`), &u)
    fmt.Println(u.Name)  // Bob
}
```

---

## やってみよう

1. `Circle` 構造体（半径を持つ）を作り、`Area()` と `Circumference()` メソッドを実装してみましょう

:::details 解答例を見る

```go
package main

import (
    "fmt"
    "math"
)

type Circle struct {
    Radius float64
}

// 面積: πr²（値レシーバ: フィールドを変更しない）
func (c Circle) Area() float64 {
    return math.Pi * c.Radius * c.Radius
}

// 円周: 2πr
func (c Circle) Circumference() float64 {
    return 2 * math.Pi * c.Radius
}

func main() {
    c := Circle{Radius: 5.0}
    fmt.Printf("半径: %.1f\n", c.Radius)
    fmt.Printf("面積: %.2f\n", c.Area())          // 78.54
    fmt.Printf("円周: %.2f\n", c.Circumference()) // 31.42
}
```

:::

2. ポインタレシーバを使って `Resize(factor float64)` メソッドを追加してみましょう

:::details 解答例を見る

```go
package main

import (
    "fmt"
    "math"
)

type Circle struct {
    Radius float64
}

func (c Circle) Area() float64 {
    return math.Pi * c.Radius * c.Radius
}

// ポインタレシーバ: Radius を変更するため *Circle が必要
func (c *Circle) Resize(factor float64) {
    c.Radius *= factor
}

func main() {
    c := Circle{Radius: 5.0}
    fmt.Printf("変更前: 半径=%.1f, 面積=%.2f\n", c.Radius, c.Area())
    // 変更前: 半径=5.0, 面積=78.54

    c.Resize(2.0) // 半径を2倍に
    fmt.Printf("変更後: 半径=%.1f, 面積=%.2f\n", c.Radius, c.Area())
    // 変更後: 半径=10.0, 面積=314.16
}
```

値レシーバ `(c Circle)` ではコピーが渡されるため、フィールドを変更しても元の構造体には反映されません。フィールドを変更する場合はポインタレシーバ `(c *Circle)` を使います。

:::

3. `NewCircle(radius float64) (*Circle, error)` コンストラクタを作り、負の半径をエラーにしてみましょう

:::details 解答例を見る

```go
package main

import (
    "fmt"
    "math"
)

type Circle struct {
    Radius float64
}

func (c Circle) Area() float64 {
    return math.Pi * c.Radius * c.Radius
}

// コンストラクタ関数: New で始める慣習
func NewCircle(radius float64) (*Circle, error) {
    if radius < 0 {
        return nil, fmt.Errorf("半径は0以上である必要があります")
    }
    return &Circle{Radius: radius}, nil
}

func main() {
    // 正常系
    c, err := NewCircle(5.0)
    if err != nil {
        fmt.Println("エラー:", err)
        return
    }
    fmt.Printf("円: 半径=%.1f, 面積=%.2f\n", c.Radius, c.Area())
    // 円: 半径=5.0, 面積=78.54

    // 異常系: 負の半径
    _, err = NewCircle(-3.0)
    if err != nil {
        fmt.Println("エラー:", err)
        // エラー: 半径は0以上である必要があります
    }
}
```

:::

4. 構造体にJSONタグを付けて、JSONとの変換を試してみましょう

:::details 解答例を見る

```go
package main

import (
    "encoding/json"
    "fmt"
)

type Circle struct {
    Radius float64 `json:"radius"`
    Color  string  `json:"color,omitempty"`
}

func main() {
    // 構造体 → JSON
    c := Circle{Radius: 5.0, Color: "red"}
    data, _ := json.Marshal(c)
    fmt.Println(string(data))
    // {"radius":5,"color":"red"}

    // omitempty: ゼロ値のフィールドは JSON から省略される
    c2 := Circle{Radius: 3.0}
    data2, _ := json.Marshal(c2)
    fmt.Println(string(data2))
    // {"radius":3}

    // JSON → 構造体
    var c3 Circle
    json.Unmarshal([]byte(`{"radius":10,"color":"blue"}`), &c3)
    fmt.Printf("半径: %.1f, 色: %s\n", c3.Radius, c3.Color)
    // 半径: 10.0, 色: blue
}
```

JSON タグはバッククォート `` ` `` で囲みます。`omitempty` を付けるとゼロ値（`""`, `0`, `false` など）のフィールドが JSON 出力から省略されます。

:::

---

次の章では、インターフェースを学びます。構造体とメソッドの知識が前提になります。
