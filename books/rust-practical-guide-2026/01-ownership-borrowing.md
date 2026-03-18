---
title: "所有権と借用"
---

# 所有権と借用

Rustを学ぶとき、最初に出会う壁が「所有権」と「借用」です。この仕組みこそが、ガベージコレクタなしでメモリ安全性を保証するRust最大の武器です。本章では「なぜこの仕組みが必要なのか」を意識しながら、実践的なコード例とともに解説します。

## 所有権の3つのルール

Rustのメモリ管理は、たった3つのルールで成り立っています。

1. **各値には「所有者」と呼ばれる変数が1つ存在します**
2. **所有者は同時に1つだけです**
3. **所有者がスコープを抜けると、値は自動的に破棄されます**

```rust
fn main() {
    {
        let s = String::from("hello"); // s が所有者になる
        println!("{}", s);
    } // s がスコープを抜ける → メモリが自動解放
    // println!("{}", s); // コンパイルエラー: s はもう存在しない
}
```

実務では、ファイルハンドルやDB接続もスコープを抜けた時点で確実にクローズされます。リソースの閉じ忘れというバグのカテゴリ自体が消えるのです。

## ムーブセマンティクス

多くの言語では `let y = x;` で値がコピーされますが、Rustではヒープ上にデータを持つ型は**ムーブ**（所有権の移転）が発生します。

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1; // ムーブ: 所有権が s1 → s2 へ
    // println!("{}", s1); // コンパイルエラー! s1 は無効化済み
    println!("{}", s2);    // OK
}
```

もし両方が同じヒープメモリを所有していたら、スコープ終了時に同じメモリが2回解放される「ダブルフリー」が起きます。ムーブはこれを根本から排除する仕組みです。

ムーブは代入以外にも、関数の引数渡しや`Vec::push`などで発生します。

```rust
fn takes_string(s: String) {
    println!("受け取った: {}", s);
} // s はここで drop される

fn main() {
    let s = String::from("hello");
    takes_string(s);
    // s は無効。「読むだけ」なら次に説明する参照を使うべき
}
```

## 参照と借用

所有権を渡さず値を使いたい場合、**借用**（参照を作る）を使います。

### 不変参照 `&T`

```rust
fn calculate_stats(data: &[i32]) -> (i32, f64) {
    let sum: i32 = data.iter().sum();
    let avg = sum as f64 / data.len() as f64;
    (sum, avg)
}

fn main() {
    let scores = vec![85, 92, 78, 96, 88];
    let (sum, avg) = calculate_stats(&scores); // 参照を渡す
    println!("合計: {}, 平均: {:.1}", sum, avg);
    println!("データ数: {}", scores.len()); // scores はまだ使える
}
```

### 可変参照 `&mut T`

```rust
fn add_tax(prices: &mut Vec<f64>, tax_rate: f64) {
    for price in prices.iter_mut() {
        *price *= 1.0 + tax_rate;
    }
}

fn main() {
    let mut prices = vec![1000.0, 2500.0, 800.0];
    add_tax(&mut prices, 0.10);
    println!("{:?}", prices); // [1100.0, 2750.0, 880.0]
}
```

## 借用ルール

Rustの借用には3つの制約があり、これによりデータ競合をコンパイル時に防止します。

1. **不変参照 `&T` は同時に複数持てます**
2. **可変参照 `&mut T` は同時に1つだけです**
3. **不変参照と可変参照は同時に存在できません**

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;
    let r2 = &s;       // OK: 不変参照は複数持てる
    println!("{}, {}", r1, r2);
    // r1, r2 はここ以降使われない（NLL により終了）

    let r3 = &mut s;   // OK: 不変参照の使用が終わった後
    r3.push_str(", world!");
    println!("{}", r3);
}
```

実務でよく遭遇するパターンとして、イテレーション中のコレクション変更があります。

```rust
fn main() {
    let mut data = vec![1, 2, 3];
    // data.iter() で不変参照中に data.push() は不可
    // → 結果を一度集めてから追加するのが定石
    let additions: Vec<i32> = data.iter()
        .filter(|&&x| x > 2).map(|&x| x * 2).collect();
    data.extend(additions);
    println!("{:?}", data); // [1, 2, 3, 6]
}
```

## スライス

### 文字列スライス `&str`

`&str` は文字列データへの軽量な参照です。関数の引数には `&String` ではなく `&str` を使うのが慣例です。`&str` のほうが汎用的で、`String`・文字列リテラル・スライスのいずれも受け取れます。

```rust
fn first_word(s: &str) -> &str {
    match s.find(' ') {
        Some(pos) => &s[..pos],
        None => s,
    }
}

fn main() {
    let sentence = String::from("Rust is fast");
    let word = first_word(&sentence);  // String → &str は自動変換
    println!("{}", word); // "Rust"

    let word2 = first_word("hello world"); // リテラルもそのまま渡せる
    println!("{}", word2); // "hello"
}
```

### 配列スライス `&[T]`

```rust
fn average(numbers: &[f64]) -> f64 {
    if numbers.is_empty() { return 0.0; }
    numbers.iter().sum::<f64>() / numbers.len() as f64
}

fn main() {
    let data = vec![10.0, 20.0, 30.0, 40.0, 50.0];
    println!("全体: {:.1}", average(&data));       // Vec 全体
    println!("前半: {:.1}", average(&data[..3]));   // 一部だけ
}
```

スライスはデータをコピーせずに「ビュー」を提供するため、大きなデータを扱う実務コードで頻繁に使われます。

## Copy トレイトと Clone トレイト

### Copy -- 暗黙のコピー

スタック上の小さな型（`i32`, `f64`, `bool`, `char` など）は `Copy` を実装しており、代入時にムーブではなくコピーが行われます。

```rust
fn main() {
    let x: i32 = 42;
    let y = x;  // コピー（ムーブではない）
    println!("x={}, y={}", x, y); // 両方有効
}
```

### Clone -- 明示的な深いコピー

ヒープデータを持つ型は `.clone()` で明示的にコピーします。

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone(); // ヒープデータごとコピー
    println!("s1={}, s2={}", s1, s2); // 両方有効
}
```

### 自作の型での使い分け

```rust
#[derive(Debug, Clone, Copy)] // 全フィールドが Copy なら derive 可能
struct Point { x: f64, y: f64 }

#[derive(Debug, Clone)]       // String があるので Copy は不可
struct User { name: String, age: u32 }

fn main() {
    let p1 = Point { x: 1.0, y: 2.0 };
    let p2 = p1;        // Copy（暗黙）
    println!("{:?}", p1); // OK

    let u1 = User { name: "太郎".into(), age: 30 };
    let u2 = u1.clone(); // Clone（明示）
    println!("{:?}", u2);
}
```

## 実務パターン: 関数に値を渡すベストプラクティス

### パターン1: 読み取りのみ → `&str` / `&[T]`

```rust
fn validate_email(email: &str) -> bool {
    email.contains('@') && email.contains('.')
}

fn find_max(values: &[i32]) -> Option<&i32> {
    values.iter().max()
}
```

### パターン2: 値を変更する → `&mut T`

```rust
fn normalize_path(path: &mut String) {
    if !path.ends_with('/') { path.push('/'); }
}
```

### パターン3: 所有権が必要 → `T`（値渡し）

```rust
fn into_csv_row(fields: Vec<String>) -> String {
    fields.join(",") // fields を消費する
}
```

### パターン4: 条件付きの所有 → `Cow`

```rust
use std::borrow::Cow;

fn sanitize(input: &str) -> Cow<'_, str> {
    if input.contains('<') {
        Cow::Owned(input.replace('<', "&lt;").replace('>', "&gt;"))
    } else {
        Cow::Borrowed(input) // 変更不要ならゼロコスト
    }
}
```

**判断の順序:** 読むだけ? → `&T`。変更する? → `&mut T`。消費する? → `T`。不要な `.clone()` を避けるのがRustらしいコードの第一歩です。

## まとめ

| 概念 | 要点 |
|------|------|
| 所有権の3ルール | 各値の所有者は1つ。スコープ終了で自動解放 |
| ムーブ | 代入や関数呼び出しで所有権が移転。元の変数は無効化 |
| Copy | `i32`, `bool` 等は暗黙にコピー。ムーブしない |
| Clone | `String`, `Vec` 等は `.clone()` で明示的に深いコピー |
| 不変参照 `&T` | 読み取り専用。同時に複数持てる |
| 可変参照 `&mut T` | 読み書き可能。同時に1つだけ |
| 借用ルール | `&T` と `&mut T` は同時に存在できない |
| `&str` / `&[T]` | 軽量な参照。関数引数にはスライスを推奨 |
| 引数設計 | 読む → `&T`、変更 → `&mut T`、消費 → `T` |

## やってみよう!

### 演習1: ムーブエラーを修正する

以下のコードのコンパイルエラーを修正してください。

```rust
fn main() {
    let name = String::from("Alice");
    let greeting = create_greeting(name);
    println!("名前: {}", name);     // ここでエラー
    println!("挨拶: {}", greeting);
}

fn create_greeting(name: String) -> String {
    format!("こんにちは、{}さん!", name)
}
```

**ヒント:** `create_greeting` の引数を参照に変えてみましょう。

### 演習2: 借用で最長文字列を返す

`Vec<String>` を受け取り、最も長い文字列への参照を返す関数を完成させてください。

```rust
fn longest_string(list: &[String]) -> &str {
    // ここを実装
    // ヒント: iter(), max_by_key() を使います
    todo!()
}

fn main() {
    let words = vec!["Rust".into(), "Ownership".into(), "Borrowing".into()];
    let longest = longest_string(&words);
    println!("最長: {}", longest);     // "Ownership"
    println!("単語数: {}", words.len()); // words はまだ使える
}
```

### 演習3: 構造体で所有権と借用を使い分ける

以下の仕様を満たす `UserRepository` を実装してください。

- `User` 構造体（名前・メールアドレスを所有）
- `add_user(&mut self, name: String, email: String)` -- ユーザー追加
- `find_by_email(&self, email: &str) -> Option<&User>` -- 検索
- `all_emails(&self) -> Vec<&str>` -- 全メール一覧

各メソッドの引数・戻り値で `&self` / `&mut self` / `&str` / `String` をどう使い分けるかを考えながら設計してみましょう。
