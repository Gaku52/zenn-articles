---
title: "型システムとトレイト"
---

## この章で学ぶこと

Rust の型システムは、コンパイル時にバグを検出し、安全で高速なプログラムを書くための基盤です。基本型から構造体・列挙型・トレイト・ジェネリクスまでを段階的に学びます。

## 基本型

### 整数型・浮動小数点・bool・char

```rust
// 整数型: 符号付き(i) / 符号なし(u) × ビット幅
let a: i32 = -42;         // デフォルトの整数型
let b: u64 = 1_000_000;   // _ で桁区切り可能
let c: usize = 10;        // ポインタサイズ（インデックスに使用）
let hex = 0xff;            // 16進数リテラル

// 浮動小数点
let pi: f64 = 3.14159;    // デフォルト
let e: f32 = 2.71828;

// bool と char
let flag: bool = true;
let ch: char = '🦀';      // Unicode スカラ値（4バイト）
```

### 文字列型：String と &str

Rust には文字列型が2種類あります。`String` はヒープ上の所有型、`&str` は借用型（読み取り専用）です。

```rust
let greeting: &str = "こんにちは";      // 文字列リテラルは &'static str
let mut owned = String::from("Hello");  // ヒープ上の所有型
owned.push_str(", world!");

// 相互変換
let s: &str = &owned;                    // String → &str（自動変換）
let s2: String = greeting.to_string();   // &str → String
```

関数の引数には `&str` を、構造体のフィールドには `String` を使うのが基本です。

## 構造体（struct）とメソッド定義（impl）

```rust
struct User {
    name: String,
    email: String,
    age: u32,
    active: bool,
}

impl User {
    // 関連関数（コンストラクタ）
    fn new(name: &str, email: &str, age: u32) -> Self {
        Self {
            name: name.to_string(),
            email: email.to_string(),
            age,
            active: true,
        }
    }

    fn display_name(&self) -> &str { &self.name }    // &self: 不変借用
    fn deactivate(&mut self) { self.active = false; } // &mut self: 可変借用
    fn into_name(self) -> String { self.name }        // self: 所有権を消費
}
```

`&self`・`&mut self`・`self` の3つのレシーバは、前章の所有権の概念と直結しています。

### タプル構造体（ニュータイプパターン）

```rust
struct Meters(f64);
struct Seconds(f64);
// Meters と Seconds を間違えて混ぜるとコンパイルエラー
```

数値の単位を型で区別することで、単位の取り違えを防げます。

## 列挙型（enum）とパターンマッチ

Rust の `enum` は各バリアントが異なるデータを保持できます。

```rust
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Triangle { base: f64, height: f64 },
}

impl Shape {
    fn area(&self) -> f64 {
        match self {
            Shape::Circle { radius } => std::f64::consts::PI * radius * radius,
            Shape::Rectangle { width, height } => width * height,
            Shape::Triangle { base, height } => 0.5 * base * height,
        }
    }
}
```

`match` では全バリアントを網羅しないとコンパイルエラーになります。バリアント追加時の対応漏れを防ぐ強力な仕組みです。

### Option と Result

Rust には `null` がありません。代わりに `Option<T>`（値の有無）と `Result<T, E>`（成否）を使います。

```rust
fn find_user(id: u32) -> Option<String> {
    if id == 1 { Some("田中".to_string()) } else { None }
}

fn main() {
    match find_user(1) {
        Some(name) => println!("ユーザー: {}", name),
        None => println!("見つかりません"),
    }
    // if let で簡潔に書くこともできます
    if let Some(name) = find_user(2) {
        println!("{}", name);
    }
}
```

## トレイトの定義と実装

トレイトは型が持つべき振る舞いを定義するインターフェースです。

```rust
trait Summary {
    fn summarize_author(&self) -> String;         // 必須メソッド

    fn summarize(&self) -> String {               // デフォルト実装
        format!("({}の記事をもっと読む...)", self.summarize_author())
    }
}

struct Article { title: String, author: String }

impl Summary for Article {
    fn summarize_author(&self) -> String { self.author.clone() }
    fn summarize(&self) -> String {  // デフォルト実装をオーバーライド
        format!("{} (by {})", self.title, self.author)
    }
}

struct Tweet { username: String, text: String }

impl Summary for Tweet {
    fn summarize_author(&self) -> String { format!("@{}", self.username) }
    // summarize() はデフォルト実装をそのまま使用
}
```

## ジェネリクスとトレイト境界（where 句）

ジェネリクスで汎用的なコードを書き、トレイト境界で型を制約します。

```rust
// トレイト境界構文
fn largest<T: PartialOrd>(list: &[T]) -> Option<&T> {
    let mut max = list.first()?;
    for item in &list[1..] {
        if item > max { max = item; }
    }
    Some(max)
}

// where 句（複雑な制約を読みやすく書く）
fn process<T, U>(t: &T, u: &U) -> String
where
    T: std::fmt::Display + Clone,
    U: std::fmt::Debug + Default,
{
    format!("T={}, U={:?}", t, u)
}

// impl Trait 構文（引数・戻り値で使える簡潔な書き方）
fn notify(item: &impl Summary) {
    println!("速報: {}", item.summarize());
}
```

ジェネリクスはコンパイル時に「単態化」され型ごとの専用コードが生成されるため、実行時オーバーヘッドはゼロです。

## デリブマクロ（#[derive(...)]）

`#[derive]` で標準トレイトの実装を自動生成できます。

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, Default)]
struct Config {
    host: String,
    port: u16,
    debug: bool,
}

fn main() {
    let c1 = Config { host: "localhost".into(), port: 8080, debug: true };
    let c2 = c1.clone();          // Clone
    println!("{:?}", c1);         // Debug
    println!("同じ? {}", c1 == c2); // PartialEq
    let d = Config::default();    // Default → host: "", port: 0, debug: false
}
```

derive 可能な標準トレイトの一覧です。

| derive 対象 | 用途 |
|---|---|
| `Debug` | `{:?}` によるデバッグ出力 |
| `Clone` / `Copy` | 明示的コピー / 暗黙のビットコピー |
| `PartialEq` / `Eq` | `==` `!=` による比較 |
| `PartialOrd` / `Ord` | `<` `>` による順序比較・ソート |
| `Hash` | `HashMap` のキーとして使用 |
| `Default` | デフォルト値の生成 |

## 標準トレイト（Display, Debug, Clone, PartialEq 等）

### Display -- ユーザー向けの表示

`Display` は derive できないため、手動で実装します。

```rust
use std::fmt;

struct Temperature(f64);

impl fmt::Display for Temperature {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{:.1}°C", self.0)
    }
}
```

### From / Into -- 型変換

`From<T>` を実装すると `Into` が自動的に使えます。

```rust
impl From<i32> for Temperature {
    fn from(val: i32) -> Self { Temperature(val as f64) }
}

fn show(t: impl Into<Temperature>) {
    let temp: Temperature = t.into();
    println!("{}", temp);
}
fn main() { show(100); }  // "100.0°C"
```

### Ord -- カスタムソート

```rust
#[derive(Debug, Eq, PartialEq)]
struct Student { name: String, score: u32 }

impl Ord for Student {
    fn cmp(&self, other: &Self) -> std::cmp::Ordering {
        other.score.cmp(&self.score) // スコア降順
    }
}
impl PartialOrd for Student {
    fn partial_cmp(&self, other: &Self) -> Option<std::cmp::Ordering> {
        Some(self.cmp(other))
    }
}
```

## TypeScript / Go のインターフェースとの比較

| 特徴 | Rust (trait) | TypeScript (interface) | Go (interface) |
|---|---|---|---|
| 実装方法 | `impl Trait for Type` で明示的 | `implements` で明示的 | メソッドが揃えば暗黙的 |
| デフォルト実装 | あり | なし（abstract class で代替） | なし |
| 静的ディスパッチ | あり（ゼロコスト） | なし | なし |
| 動的ディスパッチ | `dyn Trait` で明示的に選択 | 常に動的 | 常に動的 |
| 演算子オーバーロード | トレイトで実現 | なし | なし |
| 孤児ルール | あり（実装の衝突を防止） | なし | なし |

Rust のトレイトは「明示的な実装」と「ゼロコスト抽象化」を両立しています。Go の暗黙的インターフェースと異なり、`impl Trait for Type` と明示することで意図しない型の一致を防ぎ、TypeScript と異なり静的ディスパッチにより実行時コストがかかりません。

## まとめ

| 概念 | 要点 |
|---|---|
| 基本型 | 整数（`i32`等）、浮動小数点（`f64`）、`bool`、`char`、`String`/`&str` |
| struct | データをまとめる直積型。名前付き・タプル・ユニットの3種 |
| impl | メソッド（`&self`/`&mut self`/`self`）と関連関数を定義 |
| enum | バリアントごとにデータを持てる直和型。`match` で網羅的に分岐 |
| Option / Result | null や例外の代わりに使う安全な型 |
| トレイト | 振る舞いのインターフェース。デフォルト実装も可能 |
| ジェネリクス | 型パラメータで汎用コードを記述。実行時コストゼロ |
| トレイト境界 / where | ジェネリクスの制約（`T: Clone + Debug`） |
| #[derive] | 標準トレイトの自動実装 |
| From / Into | 型変換トレイト。`From` 実装で `Into` が自動提供 |
| 他言語比較 | 明示的実装 + ゼロコスト抽象化が Rust の特徴 |

## やってみよう！

**演習1: 構造体とメソッド**
`Book` 構造体（`title: String`、`price: u32`、`pages: u32`）を定義し、`new()`・`price_per_page() -> f64`（1ページ単価）・`is_long() -> bool`（300ページ以上で `true`）を実装してください。

**演習2: enum とパターンマッチ**
`PaymentMethod` 列挙型を `CreditCard { number: String }`・`BankTransfer { bank_name: String }`・`Cash` の3バリアントで定義し、`describe() -> String` で支払い方法の説明を返してください。

**演習3: トレイトと動的ディスパッチ**
`Printable` トレイト（`fn to_line(&self) -> String`）を定義し、演習1の `Book` と演習2の `PaymentMethod` に実装してください。`Vec<Box<dyn Printable>>` に格納してループで出力しましょう。

**演習4: ジェネリクス**
`fn find_min<T: PartialOrd>(items: &[T]) -> Option<&T>` を実装し、整数・文字列スライスの両方で動作確認してください。
