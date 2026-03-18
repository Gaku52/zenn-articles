---
title: "ライフタイム"
---

## ライフタイムとは何か

Rustにはガベージコレクタがありません。その代わりに、**ライフタイム**という仕組みでメモリ安全性をコンパイル時に保証します。ライフタイムとは、参照が有効である期間をコンパイラが追跡する仕組みのことです。

### ダングリング参照の防止

C や C++ では、既に解放されたメモリへの参照（ダングリング参照）が実行時のクラッシュや未定義動作を引き起こします。Rust ではこれをコンパイル時に検出します。

```rust
// このコードはコンパイルエラーになります
fn dangle() -> &String {
    let s = String::from("hello");
    &s  // s はこの関数の終了時に drop される → ダングリング参照！
}

// 正しい方法: 所有権ごと返します
fn no_dangle() -> String {
    String::from("hello")
}
```

借用チェッカーは、参照が指す先のデータよりも参照自体が長生きしないことを検証します。以下の図でライフタイムの長さの関係を確認してみましょう。

```
fn main() {
    let r;                // ---------+-- 'a（r のライフタイム）
    {                     //          |
        let x = 5;       // -+-- 'b  |  （x のライフタイム）
        r = &x;          //  |       |
    }                     // -+       |  ← x が drop される
    // println!("{}", r); //          |  ← 無効な参照 → コンパイルエラー
}                         // ---------+
```

`'b` は `'a` より短いため、`r = &x` は不正です。コンパイラがこの関係を自動的に検出してくれます。

## ライフタイム注釈の構文

ライフタイム注釈はアポストロフィ `'` に続く小文字のアルファベットで記述します。慣例では `'a`、`'b`、`'c` のように短い名前を使います。

```rust
&'a T        // ライフタイム 'a を持つ不変参照
&'a mut T    // ライフタイム 'a を持つ可変参照
```

ライフタイム注釈は「参照がどれくらい生きるか」を変えるものではありません。**複数の参照間の関係**をコンパイラに伝えるものです。実行時のコストは一切ありません。

## 関数のライフタイムパラメータ

関数が参照を受け取って参照を返す場合、戻り値のライフタイムがどの引数に由来するかをコンパイラに伝える必要があります。

```rust
// 2つの文字列スライスのうち長い方を返す関数
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let string1 = String::from("long string");
    {
        let string2 = String::from("xyz");
        let result = longest(string1.as_str(), string2.as_str());
        println!("長い方: {}", result); // OK: string2 はまだ有効
    }
}
```

`longest<'a>` の `'a` は「引数 `x` と `y` のライフタイムの短い方」を表しています。戻り値はその短い方の期間だけ有効です。

戻り値が一方の引数にしか依存しない場合は、ライフタイムを分けられます。

```rust
fn first<'a, 'b>(x: &'a str, _y: &'b str) -> &'a str {
    x  // 戻り値は x のライフタイムにのみ依存
}
```

戻り値が新しい所有値であれば、ライフタイム注釈は不要です。

```rust
fn combine(x: &str, y: &str) -> String {
    format!("{}{}", x, y)  // 新しい String を返す → 注釈不要
}
```

## 構造体のライフタイム

構造体が参照をフィールドに持つ場合、ライフタイムパラメータが必要です。「構造体のインスタンスが、参照先のデータよりも長生きしないこと」が保証されます。

```rust
#[derive(Debug)]
struct Excerpt<'a> {
    part: &'a str,
}

impl<'a> Excerpt<'a> {
    fn announce_and_return(&self, announcement: &str) -> &str {
        println!("お知らせ: {}", announcement);
        self.part  // 省略規則3により &self のライフタイムが適用
    }
}

fn main() {
    let novel = String::from("むかしむかし。ある所に...");
    let excerpt = Excerpt {
        part: novel.split('。').next().unwrap(),
    };
    println!("{:?}", excerpt);
}
```

複数の参照フィールドがある場合は、複数のライフタイムパラメータを使います。

```rust
struct Pair<'a, 'b> {
    first: &'a str,
    second: &'b str,
}
```

なお、構造体が自分自身のフィールドを参照する「自己参照構造体」は Rust では直接作れません。代わりにインデックスで間接参照するか、所有型と参照型を分離する設計が一般的です。

## ライフタイム省略規則（elision rules）

毎回ライフタイム注釈を書くのは大変です。コンパイラには3つの**省略規則**があり、多くの場合は注釈を省略できます。

| 規則 | 内容 |
|------|------|
| 規則1（入力） | 各参照パラメータに個別のライフタイムを割り当てる |
| 規則2（出力） | 入力ライフタイムが1つなら、出力にも同じものを適用 |
| 規則3（メソッド） | `&self` のライフタイムを出力に適用 |

3つの規則を適用しても出力のライフタイムが決まらない場合、明示的な注釈が必要です。

```rust
// 省略可能: 入力が1つ → 規則1+2で推論できる
fn first_word(s: &str) -> &str {
    s.split_whitespace().next().unwrap_or("")
}
// 展開すると: fn first_word<'a>(s: &'a str) -> &'a str

// 省略不可: 入力が2つ → どちらの参照が返るかわからない
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

// 省略可能: メソッド → 規則3で &self のライフタイムが適用
struct MyString { data: String }
impl MyString {
    fn as_str(&self) -> &str { &self.data }
    // 展開すると: fn as_str<'a>(&'a self) -> &'a str
}
```

## 'static ライフタイム

`'static` はプログラム全体の実行期間を表す特別なライフタイムです。2つの異なる意味があります。

| 記法 | 意味 | 例 |
|------|------|-----|
| `&'static T` | プログラム全期間有効な参照 | 文字列リテラル `"hello"` |
| `T: 'static` | 型 T が参照を含まないか、'static 参照のみ含む | `String`, `i32` など所有型すべて |

`T: 'static` は「永遠にメモリに残る」ではなく「永遠に生きることが**可能**」という意味です。

```rust
// 文字列リテラルは 'static
let s: &'static str = "バイナリに埋め込まれる";

// スレッドに値を渡すには T: Send + 'static が必要
fn spawn_task<T: Send + 'static>(value: T) {
    std::thread::spawn(move || {
        println!("処理中");
        drop(value);
    });
}

fn main() {
    let owned = String::from("hello");
    spawn_task(owned); // OK: String は所有型なので 'static を満たす
}
```

### 'static の誤用に注意

```rust
// BAD: 不必要に 'static を要求 → String の参照を渡せなくなる
fn process_bad(data: &'static str) { println!("{}", data); }

// GOOD: 任意のライフタイムを受け入れる
fn process_good(data: &str) { println!("{}", data); }
```

本当に必要な場合（スレッド間共有など）以外は `&'static` を引数に要求するのは避けましょう。

## ライフタイムとジェネリクスの組み合わせ

ライフタイムパラメータはジェネリクスパラメータの一種です。慣例として型パラメータの前に書きます。

```rust
use std::fmt::Display;

struct Annotated<'a, T> {
    label: &'a str,
    value: T,
}

impl<'a, T: Display> Annotated<'a, T> {
    fn new(label: &'a str, value: T) -> Self {
        Annotated { label, value }
    }

    fn display(&self) {
        println!("{}: {}", self.label, self.value);
    }
}

fn main() {
    let label = String::from("温度");
    let annotated = Annotated::new(&label, 36.5_f64);
    annotated.display(); // 温度: 36.5
}
```

ライフタイム境界 `'a: 'b` は「`'a` は少なくとも `'b` と同じだけ長く生きる」という制約を表現します。

```rust
fn select<'a, 'b: 'a>(first: &'a str, second: &'b str) -> &'a str {
    if first.len() > second.len() { first } else { second }
    // 'b: 'a なので 'b の参照を 'a として返せる
}
```

## 実務での典型パターン: ライフタイムエラーが出たらどうする？

### パターン1: ローカル変数への参照を返そうとした

```
error[E0515]: cannot return reference to local variable `s`
```

**対策**: 所有権ごと返しましょう。

```rust
// NG: ローカル変数への参照は返せない
// fn make_greeting(name: &str) -> &str {
//     let s = format!("Hello, {}!", name);
//     &s
// }

// OK: String を返す
fn make_greeting(name: &str) -> String {
    format!("Hello, {}!", name)
}
```

### パターン2: ライフタイムが合わない

```
error[E0597]: `x` does not live long enough
```

**対策**: データのスコープを広げるか、`.clone()` で所有値を作ります。

### パターン3: 不変借用と可変借用の衝突

```
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
```

**対策**: NLL（Non-Lexical Lifetimes）を活用し、不変参照の最終使用後に可変操作を行います。

```rust
let mut v = vec![1, 2, 3];
let first = &v[0];
println!("{}", first); // first の最後の使用 → ライフタイム終了
v.push(4);             // OK: first はもう使われていない
```

### パターン4: 構造体のライフタイムが複雑になりすぎた

**対策**: まず所有型（`String`、`Vec<T>`）で設計し、パフォーマンスが問題になったら参照に切り替えます。

```rust
// 最初はシンプルに所有型で
struct Config { name: String, values: Vec<String> }

// パフォーマンスが必要な箇所だけ参照に
struct ConfigView<'a> { name: &'a str, values: &'a [String] }
```

### 判断フローチャート

ライフタイムエラーに遭遇したら、以下の順に検討してください。

1. **所有値を返せるか？** → `String` や `Vec<T>` を返す
2. **`.clone()` で解決するか？** → まず clone で動かし、後で最適化
3. **スコープを調整できるか？** → データの定義位置を外側に移動
4. **ライフタイム注釈で解決するか？** → 関数シグネチャに `'a` を追加
5. **設計を見直すべきか？** → 参照ではなく所有型ベースの設計に変更

## まとめ

| 概念 | 要点 |
|------|------|
| ライフタイム | 参照の有効期間をコンパイル時に追跡し、ダングリング参照を防ぐ仕組み |
| `'a` 注釈 | 複数の参照間の関係をコンパイラに伝える。実行時コストはゼロ |
| 関数のライフタイム | 戻り値がどの引数の参照に由来するかを明示する |
| 構造体のライフタイム | 参照フィールドを持つ構造体には `'a` パラメータが必要 |
| 省略規則 | 3つの規則（入力・出力・メソッド）で多くの場合は注釈を省略できる |
| `'static` | プログラム全体の期間を表す。所有型はすべて `T: 'static` を満たす |
| ジェネリクスとの組み合わせ | `'a: 'b` で outlives 関係を表現。型パラメータの前に書く |
| エラー対処 | まず所有型で解決 → clone → スコープ調整 → 注釈追加の順に検討 |

## やってみよう！

### 演習1: longest 関数を書いてみよう

2つの `&str` を受け取り、長い方を返す関数 `longest` を実装してください。ライフタイム注釈を正しく付ける必要があります。

```rust
// TODO: ライフタイム注釈を付けて実装してください
fn longest(x: &str, y: &str) -> &str {
    // ヒント: <'a> を追加し、引数・戻り値に 'a を付けます
    todo!()
}

fn main() {
    let s1 = String::from("Rust");
    let s2 = String::from("Programming");
    let result = longest(s1.as_str(), s2.as_str());
    println!("長い方: {}", result);
}
```

### 演習2: 構造体にライフタイムを付けよう

以下の構造体 `Article` にライフタイムパラメータを付けてコンパイルを通してください。

```rust
// TODO: ライフタイムを付けてください
struct Article {
    title: &str,
    body: &str,
}

// ヒント: struct Article<'a> とし、impl にも <'a> が必要です
impl Article {
    fn summary(&self) -> String {
        format!("{}...", &self.body[..20.min(self.body.len())])
    }
}
```

### 演習3: ライフタイムエラーを修正しよう

以下のコードはコンパイルエラーになります。修正してください。

```rust
fn first_word(s: &str) -> &str {
    s.split_whitespace().next().unwrap_or("")
}

fn main() {
    let word;
    {
        let sentence = String::from("hello world");
        word = first_word(&sentence);
    }
    println!("{}", word); // sentence が drop 済み → エラー！
    // ヒント: sentence のスコープを word より長くしましょう
}
```
