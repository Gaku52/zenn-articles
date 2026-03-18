---
title: "クロージャとFnトレイト"
---

## クロージャとは

クロージャは、定義されたスコープの変数を**キャプチャ(捕捉)**できる匿名関数です。通常の `fn` 関数と異なり、外側の環境にある変数を直接利用できます。

```rust
fn main() {
    let multiplier = 3;
    let multiply = |x| x * multiplier; // 外側の multiplier をキャプチャ
    println!("{}", multiply(5)); // 15
}
```

## クロージャの構文と型推論

構文は `|引数| 式` です。型注釈を省略でき、コンパイラが文脈から推論します。

```rust
fn main() {
    let add = |a, b| a + b;                           // 型推論に任せる
    let add_typed = |a: i32, b: i32| -> i32 { a + b }; // 型を明示
    let process = |x: i32| {                           // 複数行
        let doubled = x * 2;
        format!("結果: {}", doubled)
    };
    println!("{} {} {}", add(3, 4), add_typed(3, 4), process(21));
}
```

関数との構文比較は次のとおりです。

```
関数:       fn add(x: i32, y: i32) -> i32 { x + y }
クロージャ: |x: i32, y: i32| -> i32 { x + y }
省略形:     |x, y| x + y
```

一度推論された型は固定されます。同じクロージャに異なる型の引数を渡すとコンパイルエラーになります。

## 環境のキャプチャ -- 参照とムーブ

コンパイラはクロージャ内での変数の使い方を解析し、最小限のキャプチャ方式を自動選択します。

```rust
fn main() {
    // (1) 不変参照でキャプチャ -- 読むだけ
    let name = String::from("Rust");
    let greet = || println!("Hello, {}!", name);
    greet();
    greet();
    println!("name はまだ使える: {}", name);

    // (2) 可変参照でキャプチャ -- 変更する
    let mut count = 0;
    let mut increment = || { count += 1; };
    increment();
    increment();
    println!("count: {}", count); // 2

    // (3) 所有権をムーブ -- 消費する
    let data = vec![1, 2, 3];
    let consume = || {
        let taken = data; // 所有権をムーブ
        println!("消費: {:?}", taken);
    };
    consume();
    // consume(); // エラー: FnOnce は1回のみ
}
```

## Fn, FnMut, FnOnce の違い

キャプチャ方式に応じて、クロージャは 3 つのトレイトのいずれかを実装します。包含関係があり、`Fn` を実装する型は `FnMut` と `FnOnce` も自動的に実装します。

```
Fn  ⊂  FnMut  ⊂  FnOnce

┌─────────────────────────────────────┐
│  FnOnce (全クロージャが実装)         │
│  ┌───────────────────────────────┐  │
│  │ FnMut                        │  │
│  │  ┌─────────────────────────┐  │  │
│  │  │ Fn  -- 不変参照のみ     │  │  │
│  │  └─────────────────────────┘  │  │
│  │ 可変参照を使用                │  │
│  └───────────────────────────────┘  │
│ 所有権を消費                        │
└─────────────────────────────────────┘
```

| トレイト | self の型 | 呼び出し回数 | キャプチャ |
|----------|-----------|-------------|-----------|
| `Fn` | `&self` | 何回でも | 不変参照 |
| `FnMut` | `&mut self` | 何回でも | 可変参照 |
| `FnOnce` | `self` | 1回のみ | 所有権取得 |

## クロージャを引数に取る関数

ジェネリクスとトレイト境界を使います。**呼び出し側の要件に合わせて最も緩い境界を選ぶ**のが原則です。

```rust
// 1回しか呼ばない → FnOnce で十分
fn run_once<F: FnOnce() -> String>(f: F) -> String { f() }

// 複数回呼ぶが変更を許容 → FnMut
fn run_multiple<F: FnMut()>(mut f: F) {
    for _ in 0..3 { f(); }
}

// 不変性が必要 → Fn
fn apply_twice<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 {
    f(x) + f(x)
}

fn main() {
    let name = String::from("World");
    println!("{}", run_once(move || format!("Hello, {}!", name)));

    let mut total = 0;
    run_multiple(|| { total += 1; });
    println!("total: {}", total); // 3

    println!("{}", apply_twice(|x| x * 2, 5)); // 20
}
```

`FnOnce` を境界にすれば `Fn` や `FnMut` のクロージャも受け入れられます。必要以上に制限的な境界(`Fn`)を選ぶと、所有権を消費するクロージャを渡せなくなるので注意してください。

## クロージャを返す関数 -- impl Fn

クロージャの型は匿名型なので、戻り値には `impl Fn` を使います。

```rust
// 静的ディスパッチ: 1種類のクロージャを返す
fn make_adder(x: i32) -> impl Fn(i32) -> i32 {
    move |y| x + y
}

// 動的ディスパッチ: match で異なるクロージャを返す場合は Box<dyn Fn>
fn make_operation(op: &str) -> Box<dyn Fn(i32, i32) -> i32> {
    match op {
        "add" => Box::new(|a, b| a + b),
        "mul" => Box::new(|a, b| a * b),
        _     => Box::new(|a, _| a),
    }
}

// FnMut を返す例: カウンター
fn make_counter(start: i32) -> impl FnMut() -> i32 {
    let mut count = start;
    move || { count += 1; count }
}

fn main() {
    let add5 = make_adder(5);
    println!("{}, {}", add5(3), add5(7)); // 8, 12

    let mul = make_operation("mul");
    println!("{}", mul(4, 5)); // 20

    let mut counter = make_counter(0);
    println!("{}, {}", counter(), counter()); // 1, 2
}
```

`impl Fn` は 1 種類のクロージャしか返せません。`match` の各アームで異なるクロージャを返す場合は `Box<dyn Fn>` を使ってください。

## move クロージャ

`move` を付けると、キャプチャする変数の**所有権を強制的にクロージャへ移転**します。スレッドに渡す場合や関数からクロージャを返す場合に必須です。

```rust
use std::thread;

fn main() {
    let message = String::from("Hello from thread!");

    let handle = thread::spawn(move || {
        println!("{}", message); // move で所有権を転送
    });
    // println!("{}", message); // エラー: ムーブ済み
    handle.join().unwrap();

    // Copy 型は move してもコピーされる
    let x = 42;
    let closure = move || println!("{}", x);
    closure();
    println!("x はまだ使える: {}", x); // OK: i32 は Copy
}
```

## イテレータとクロージャの組み合わせ

イテレータアダプタはクロージャを引数に取ります。`filter`, `map`, `fold` などを連鎖させて宣言的なパイプラインを構築できます。

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // filter → map → collect パイプライン
    let result: Vec<String> = numbers.iter()
        .filter(|&&x| x % 2 == 0)
        .map(|&x| x * x)
        .filter(|&x| x > 10)
        .map(|x| format!("値: {}", x))
        .collect();
    println!("{:?}", result); // ["値: 16", "値: 36", "値: 64", "値: 100"]

    // fold で集約
    let sum: i32 = numbers.iter().fold(0, |acc, &x| acc + x);
    println!("合計: {}", sum); // 55

    // flat_map でネストを平坦化
    let sentences = vec!["hello world", "rust is great"];
    let words: Vec<&str> = sentences.iter()
        .flat_map(|s| s.split_whitespace())
        .collect();
    println!("{:?}", words); // ["hello", "world", "rust", "is", "great"]

    // scan で累積和(状態を保持した変換)
    let running: Vec<i32> = vec![1, 2, 3, 4, 5].iter()
        .scan(0, |state, &x| { *state += x; Some(*state) })
        .collect();
    println!("{:?}", running); // [1, 3, 6, 10, 15]
}
```

## 実務パターン: コールバック

イベント駆動設計では、クロージャをコールバックとして登録するパターンが頻出します。異なるクロージャを同じコレクションに格納するため `Box<dyn Fn>` を使います。

```rust
use std::collections::HashMap;

struct EventEmitter {
    listeners: HashMap<String, Vec<Box<dyn Fn(&str)>>>,
}

impl EventEmitter {
    fn new() -> Self {
        EventEmitter { listeners: HashMap::new() }
    }
    fn on(&mut self, event: &str, cb: impl Fn(&str) + 'static) {
        self.listeners.entry(event.to_string())
            .or_insert_with(Vec::new)
            .push(Box::new(cb));
    }
    fn emit(&self, event: &str, data: &str) {
        if let Some(cbs) = self.listeners.get(event) {
            for cb in cbs { cb(data); }
        }
    }
}

fn main() {
    let mut emitter = EventEmitter::new();
    emitter.on("click", |d| println!("クリック: {}", d));
    emitter.on("click", |d| println!("ログ: {}", d));
    emitter.emit("click", "ボタンA");
}
```

`'static` 境界は、`Box<dyn Fn>` に格納するためにクロージャがローカル変数への参照を持たないことを要求しています。

## 実務パターン: ビルダーパターン

クロージャを蓄積するビルダーで、柔軟な処理パイプラインを構築できます。

```rust
struct Pipeline {
    steps: Vec<Box<dyn Fn(String) -> String>>,
}

impl Pipeline {
    fn new() -> Self { Pipeline { steps: Vec::new() } }
    fn add(mut self, step: impl Fn(String) -> String + 'static) -> Self {
        self.steps.push(Box::new(step));
        self
    }
    fn execute(&self, input: &str) -> String {
        self.steps.iter().fold(input.to_string(), |acc, step| step(acc))
    }
}

fn main() {
    let output = Pipeline::new()
        .add(|s| s.trim().to_string())
        .add(|s| s.to_uppercase())
        .add(|s| format!("[DONE] {}", s))
        .execute("  hello world  ");
    println!("{}", output); // [DONE] HELLO WORLD
}
```

メソッドチェーンで `self` を返しながらクロージャを蓄積する設計は、設定やクエリビルダーなど幅広い場面で応用できます。

## まとめ

| 概念 | 要点 |
|------|------|
| クロージャ | 環境をキャプチャできる匿名関数。`\|引数\| 式` で定義する |
| 型推論 | 引数・戻り値の型注釈は省略可能。一度推論されると固定される |
| 不変キャプチャ | 読み取りのみ → `&T` で借用。`Fn` を実装する |
| 可変キャプチャ | 変更あり → `&mut T` で借用。`FnMut` を実装する |
| 所有権消費 | ムーブ → `T`。`FnOnce` を実装し 1 回のみ呼べる |
| トレイト境界 | 最も緩い境界を選ぶ。`FnOnce` > `FnMut` > `Fn` の順 |
| `move` | 所有権をクロージャへ強制移転。スレッドや戻り値で必須 |
| `impl Fn` | 静的ディスパッチでクロージャを返す(1 種類のみ) |
| `Box<dyn Fn>` | 動的ディスパッチ。異なるクロージャを統一的に格納できる |
| イテレータ連携 | `filter`, `map`, `fold` 等にクロージャを渡して宣言的処理 |

## やってみよう!

**演習 1 -- 変換パイプライン:** `Vec<i32>` を受け取り、(1) 奇数抽出 (2) 3倍 (3) 100以下のみ (4) 降順ソートして返す関数 `transform` をイテレータチェーンで書いてください。期待値: `vec![27, 21, 15, 9, 3]`

**演習 2 -- 関数ファクトリ:** `"upper"` なら大文字変換、`"repeat"` なら 2 回繰り返し、それ以外ならそのまま返すクロージャを返す `make_formatter(style: &str) -> Box<dyn Fn(&str) -> String>` を実装してください。

**演習 3 -- イベントシステム:** 本章のコールバックパターンを拡張し、`on` / `emit` / `off`(全コールバック解除)を持つ `EventSystem` を実装してください。ヒント: `HashMap<String, Vec<Box<dyn Fn(&str)>>>` を内部に持つ構造体にします。
