---
title: "基本構文と型"
---

# Chapter 03: 基本構文と型

> 変数、定数、基本型 — Goのコードを読み書きする土台

## この章で学べること

- ✅ 変数宣言（var と :=）
- ✅ 定数（const）
- ✅ 基本型（int, float64, string, bool）
- ✅ 型変換
- ✅ ゼロ値の仕組み
- ✅ fmt パッケージによる入出力

---

## 変数宣言

Goには変数を宣言する方法が2つあります。

### var 宣言

```go
// 型を明示して宣言
var name string = "Go"
var age int = 15

// 型推論（右辺から型を推論）
var language = "Go"    // string と推論される
var year = 2009        // int と推論される

// 宣言だけ（ゼロ値で初期化される）
var count int          // 0
var message string     // ""（空文字列）
var active bool        // false
```

### 短縮宣言 `:=`

**関数内でのみ**使える省略形です。Goのコードで最も頻繁に使われます。

```go
func main() {
    name := "Go"        // var name string = "Go" と同じ
    age := 15            // var age int = 15 と同じ
    pi := 3.14159        // var pi float64 = 3.14159 と同じ
    active := true       // var active bool = true と同じ
}
```

### var と := の使い分け

| 場面 | 使うもの | 例 |
|------|---------|-----|
| 関数内の通常の変数 | `:=` | `name := "Go"` |
| ゼロ値で初期化したい | `var` | `var count int` |
| パッケージレベルの変数 | `var` | `var Version = "1.0"` |
| 型を明示したい | `var` | `var ratio float32 = 0.5` |

```go
package main

// パッケージレベルでは := は使えない
var AppName = "MyApp"

func main() {
    // 関数内では := が基本
    version := "1.0.0"

    // ゼロ値で初期化したい場合は var
    var errorCount int  // 0

    fmt.Println(AppName, version, errorCount)
}
```

---

## 定数

定数は `const` で宣言します。コンパイル時に値が確定し、変更できません。

```go
const Pi = 3.14159265358979
const AppName = "MyApp"
const MaxRetries = 3

// まとめて宣言
const (
    StatusOK    = 200
    StatusNotFound = 404
    StatusError = 500
)
```

### iota — 連番の自動生成

```go
type Weekday int

const (
    Sunday    Weekday = iota  // 0
    Monday                     // 1
    Tuesday                    // 2
    Wednesday                  // 3
    Thursday                   // 4
    Friday                     // 5
    Saturday                   // 6
)
```

`iota` は `const` ブロック内で0から始まり、行ごとに1ずつ増えます。列挙型の代わりに使います。

```go
// ビットフラグにも使える
const (
    ReadPermission   = 1 << iota  // 1  (001)
    WritePermission                // 2  (010)
    ExecutePermission              // 4  (100)
)
```

---

## 基本型

### 数値型

```go
// 整数型
var i int = 42          // プラットフォーム依存（32bit or 64bit）
var i8 int8 = 127       // -128 〜 127
var i16 int16 = 32767
var i32 int32 = 2147483647
var i64 int64 = 9223372036854775807

// 符号なし整数
var u uint = 42
var u8 uint8 = 255      // byte のエイリアス
var u32 uint32 = 4294967295

// 浮動小数点数
var f32 float32 = 3.14
var f64 float64 = 3.141592653589793  // 通常はこちらを使う

// 複素数（あまり使わない）
var c complex128 = 1 + 2i
```

**基本方針**: 整数は `int`、浮動小数点数は `float64` を使えばほぼ困りません。

### 文字列型

```go
name := "Hello, Go!"

// 文字列の長さ（バイト数）
fmt.Println(len(name))  // 10

// 文字列の結合
greeting := "Hello" + ", " + "World"

// 複数行文字列（バッククォート）
query := `
    SELECT *
    FROM users
    WHERE active = true
`

// 文字列は不変（immutable）
// name[0] = 'h'  // コンパイルエラー
```

### rune 型

Goの `string` はバイト列です。日本語などのマルチバイト文字を扱うには `rune`（= `int32`）を使います。

```go
s := "こんにちは"
fmt.Println(len(s))           // 15（バイト数）
fmt.Println(len([]rune(s)))   // 5（文字数）

// 1文字ずつ処理
for i, r := range s {
    fmt.Printf("index=%d, rune=%c\n", i, r)
}
```

### bool 型

```go
active := true
found := false

// 論理演算
fmt.Println(active && found)   // false（AND）
fmt.Println(active || found)   // true（OR）
fmt.Println(!active)           // false（NOT）
```

---

## ゼロ値

Goでは、変数を宣言するだけで**ゼロ値**が自動的に設定されます。`null` や `undefined` で悩む必要がありません。

```go
var i int       // 0
var f float64   // 0.0
var s string    // ""（空文字列）
var b bool      // false
var p *int      // nil（ポインタのゼロ値）
```

これはGo Proverb「**Make the zero value useful（ゼロ値を有用にせよ）**」の実践です。

```go
// bytes.Buffer はゼロ値のまま使える
var buf bytes.Buffer
buf.WriteString("Hello")
buf.WriteString(", World")
fmt.Println(buf.String())  // "Hello, World"
// new(bytes.Buffer) や初期化関数は不要
```

---

## 型変換

Goには**暗黙の型変換がありません**。すべて明示的に変換する必要があります。

```go
i := 42
f := float64(i)     // int → float64
u := uint(i)        // int → uint

f2 := 3.14
i2 := int(f2)       // float64 → int（小数点以下は切り捨て: 3）

// 文字列との変換
s := strconv.Itoa(42)        // int → string: "42"
n, err := strconv.Atoi("42") // string → int: 42

// 型の不一致はコンパイルエラー
// sum := i + f  // エラー: int と float64 は直接演算できない
sum := float64(i) + f  // OK: 両方 float64 にそろえる
```

---

## fmt パッケージ — 入出力の基本

### 出力

```go
name := "Go"
version := 1.23

// 基本出力
fmt.Println("Hello, World!")                  // 改行あり
fmt.Print("Hello ")                           // 改行なし
fmt.Printf("Language: %s v%.2f\n", name, version)  // フォーマット指定

// よく使うフォーマット指定子
fmt.Printf("%s\n", "文字列")     // 文字列
fmt.Printf("%d\n", 42)          // 整数
fmt.Printf("%f\n", 3.14)        // 浮動小数点数
fmt.Printf("%t\n", true)        // 真偽値
fmt.Printf("%v\n", anything)    // 任意の値（汎用）
fmt.Printf("%T\n", anything)    // 型名を表示
fmt.Printf("%+v\n", myStruct)   // 構造体のフィールド名付き表示
```

### 文字列のフォーマット

`Sprintf` は画面に出力せず、文字列として返します。

```go
msg := fmt.Sprintf("User: %s, Age: %d", "Alice", 30)
// msg = "User: Alice, Age: 30"
```

---

## 複数の変数をまとめて宣言

```go
// var ブロック
var (
    name    string = "Go"
    version int    = 23
    stable  bool   = true
)

// 複数同時代入
x, y := 10, 20

// 値の交換（一時変数不要）
x, y = y, x
fmt.Println(x, y)  // 20, 10
```

---

## 未使用の変数はコンパイルエラー

```go
func main() {
    x := 42
    // x を使わないとコンパイルエラー
    // "x declared and not used"

    _ = x  // ブランク識別子 _ で「意図的に無視」を明示
}
```

`_`（ブランク識別子）は「この値は使わない」ことを明示するGoの仕組みです。後の章で頻繁に登場します。

---

## やってみよう

1. 自分の名前と年齢を変数に入れて `fmt.Printf` で表示してみましょう
2. `iota` を使って季節（Spring=0, Summer=1, ...）の定数を定義してみましょう
3. 文字列 `"Go言語"` の `len()` と `len([]rune())` の結果を比較してみましょう
4. `int` と `float64` の型変換を試してみましょう

---

次の章では、制御フロー（if, for, switch）を学びます。
