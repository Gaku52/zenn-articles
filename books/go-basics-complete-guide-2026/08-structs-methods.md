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

> **補足**: `rect.Scale(2)` は自動で `(&rect).Scale(2)` に変換されます。明示的に `&` を書く必要はありません。

### 使い分けの基準

| 条件 | 値レシーバ | ポインタレシーバ |
|------|-----------|----------------|
| フィールドを変更する | - | ✅ |
| 構造体が大きい | - | ✅ |
| 読み取りだけ & 構造体が小さい | ✅ | - |
| 一貫性（他のメソッドがポインタ） | - | ✅ |

**実務のルール**: 迷ったら**ポインタレシーバ**を使ってください。1つでもポインタレシーバがある型は、**全てのメソッドをポインタレシーバに統一する**のが一般的です。

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
2. ポインタレシーバを使って `Resize(factor float64)` メソッドを追加してみましょう
3. `NewCircle(radius float64) (*Circle, error)` コンストラクタを作り、負の半径をエラーにしてみましょう
4. 構造体にJSONタグを付けて、JSONとの変換を試してみましょう

---

次の章では、インターフェースを学びます。構造体とメソッドの知識が前提になります。
