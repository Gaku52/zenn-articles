---
title: "コレクションとイテレータ"
---

# Chapter 4: コレクションとイテレータ

> コレクションとイテレータを組み合わせることで、型安全でゼロコストなデータ処理パイプラインを構築できます。

## この章で学べること

- Vec\<T\>、HashMap\<K, V\>、HashSet\<T\>、BTreeMap の使い分け
- iter / into_iter / iter_mut と所有権の関係
- map、filter、flat_map、zip、enumerate 等のイテレータアダプタ
- collect、fold、sum、any、all、find 等の消費メソッド
- メソッドチェーンによる関数型データ変換パイプライン

**前提知識**: Chapter 01〜03 の内容 / **所要時間**: 40-60分

---

## 1. 主要コレクション

### Vec\<T\> -- 動的配列

`Vec<T>` は最も頻繁に使うコレクションです。ヒープに連続メモリを確保し、末尾追加が償却 O(1) で行えます。

```rust
fn main() {
    let mut v = vec![1, 2, 3, 4, 5]; // マクロで初期化

    v.push(6);                           // 末尾追加
    println!("{:?}", v.get(99));         // None（安全なアクセス）

    v.sort();
    v.retain(|&x| x % 2 == 0);          // 偶数だけ残す
    println!("{:?}", v);                 // [2, 4, 6]

    // サイズが事前にわかる場合は with_capacity で再割り当て回避
    let mut buf = Vec::with_capacity(1000);
    for i in 0..1000 { buf.push(i); }   // 再割り当てゼロ
}
```

### HashMap\<K, V\>

キー値ペアを格納し、平均 O(1) で検索できます。`entry` API で挿入と更新を簡潔に書けます。

```rust
use std::collections::HashMap;

fn main() {
    let mut scores: HashMap<&str, u32> = HashMap::new();
    scores.insert("田中", 85);
    scores.insert("鈴木", 92);

    // entry API: キーが存在しなければ挿入、あれば更新
    *scores.entry("田中").or_insert(0) += 10;

    // 単語カウント -- 実務頻出パターン
    let text = "hello world hello rust hello";
    let mut wc = HashMap::new();
    for word in text.split_whitespace() {
        *wc.entry(word).or_insert(0) += 1;
    }
    println!("{:?}", wc); // {"hello": 3, "world": 1, "rust": 1}
}
```

### HashSet\<T\> と BTreeMap\<K, V\>

```rust
use std::collections::{HashSet, BTreeMap};

fn main() {
    // HashSet: 重複なし集合 + 集合演算
    let a: HashSet<i32> = [1, 2, 3, 4, 5].into();
    let b: HashSet<i32> = [3, 4, 5, 6, 7].into();
    let inter: HashSet<_> = a.intersection(&b).collect(); // {3, 4, 5}
    let diff: HashSet<_> = a.difference(&b).collect();    // {1, 2}

    // BTreeMap: ソート順保持 + 範囲検索
    let mut map = BTreeMap::new();
    map.insert("Charlie", 85);
    map.insert("Alice", 92);
    map.insert("Bob", 78);
    for (name, score) in map.range("Alice"..="Bob") {
        println!("{}: {}", name, score); // Alice: 92, Bob: 78
    }
}
```

| 要件 | 推奨 | 計算量 |
|------|------|--------|
| 順序付きリスト | `Vec<T>` | 末尾追加 O(1) |
| キーで高速検索 | `HashMap<K, V>` | 平均 O(1) |
| ソート順検索 | `BTreeMap<K, V>` | O(log n) |
| 重複排除 | `HashSet<T>` | 平均 O(1) |
| 優先度キュー | `BinaryHeap<T>` | push/pop O(log n) |

---

## 2. イテレータの基本 -- 3つの iter

所有権の扱いが異なる3種類のイテレータを正しく使い分けることが重要です。

```rust
fn main() {
    let v = vec!["hello".to_string(), "world".to_string()];

    // iter(): &T -- コレクションは残る
    for s in v.iter() { println!("{}", s); }
    println!("v はまだ使える: {:?}", v);

    // iter_mut(): &mut T -- 要素を直接変更できる
    let mut nums = vec![1, 2, 3];
    for n in nums.iter_mut() { *n *= 2; }
    println!("{:?}", nums); // [2, 4, 6]

    // into_iter(): T -- コレクションを消費する
    let upper: Vec<String> = v.into_iter()
        .map(|s| s.to_uppercase())
        .collect();
    // v はもう使えない（ムーブ済み）
    println!("{:?}", upper);
}
```

| メソッド | 要素の型 | コレクション |
|----------|---------|-------------|
| `.iter()` | `&T` | そのまま残る |
| `.iter_mut()` | `&mut T` | そのまま残る |
| `.into_iter()` | `T` | 消費される |

後でコレクションを使うなら `iter()`、もう使わないなら `into_iter()` を選びます。

---

## 3. イテレータアダプタ

アダプタは**遅延評価**されます。消費メソッドが呼ばれて初めて処理が実行されます。

### map / filter

```rust
fn main() {
    let nums = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    let doubled: Vec<i32> = nums.iter().map(|x| x * 2).collect();
    let evens: Vec<&i32> = nums.iter().filter(|x| *x % 2 == 0).collect();
    println!("{:?}", doubled); // [2, 4, ..., 20]
    println!("{:?}", evens);   // [2, 4, 6, 8, 10]
}
```

### flat_map / zip / enumerate

```rust
fn main() {
    // flat_map: ネスト構造をフラットに
    let sentences = vec!["hello world", "foo bar"];
    let words: Vec<&str> = sentences.iter()
        .flat_map(|s| s.split_whitespace()).collect();
    println!("{:?}", words); // ["hello", "world", "foo", "bar"]

    // zip: 2つのイテレータをペアに
    let names = vec!["Alice", "Bob"];
    let ages = vec![30, 25];
    let people: Vec<_> = names.iter().zip(ages.iter()).collect();
    println!("{:?}", people); // [("Alice", 30), ("Bob", 25)]

    // enumerate: インデックス付き
    for (i, name) in names.iter().enumerate() {
        println!("[{}] {}", i, name);
    }
}
```

### その他の便利なアダプタ

```rust
fn main() {
    let data = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    let first3: Vec<_> = data.iter().take(3).collect();            // [1, 2, 3]
    let after3: Vec<_> = data.iter().skip(3).collect();            // [4, ..., 10]
    let small: Vec<_> = data.iter().take_while(|&&x| x < 5).collect(); // [1, 2, 3, 4]

    // chain: 2つのイテレータを連結
    let combined: Vec<_> = [1, 2].iter().chain([3, 4].iter()).collect();

    // scan: 累積和（状態付き変換）
    let running: Vec<i32> = data.iter()
        .scan(0, |s, &x| { *s += x; Some(*s) }).collect();
    println!("累積和: {:?}", running); // [1, 3, 6, 10, ...]
}
```

---

## 4. イテレータの消費

消費メソッドがイテレーションを駆動し、最終結果を生成します。

### collect -- 柔軟なコレクション変換

```rust
use std::collections::{HashMap, HashSet};

fn main() {
    let v: Vec<i32> = (1..=5).collect();
    let set: HashSet<i32> = vec![1, 2, 2, 3].into_iter().collect(); // 重複排除
    let map: HashMap<&str, i32> = vec![("a", 1), ("b", 2)].into_iter().collect();

    // Result<Vec<T>, E> への collect -- エラーがあれば即 Err
    let result: Result<Vec<i32>, _> = vec!["1", "abc", "3"].iter()
        .map(|s| s.parse::<i32>()).collect();
    println!("{:?}", result); // Err(ParseIntError)
}
```

### fold / sum / any / all / find

```rust
fn main() {
    let nums = vec![1, 2, 3, 4, 5];

    let sum: i32 = nums.iter().sum();                          // 15
    let product: i32 = nums.iter().product();                  // 120
    let factorial = (1..=10).fold(1u64, |acc, x| acc * x);    // 3628800

    let has_even = nums.iter().any(|&x| x % 2 == 0);          // true
    let all_pos = nums.iter().all(|&x| x > 0);                // true
    let first_even = nums.iter().find(|&&x| x % 2 == 0);      // Some(&2)
    let pos = nums.iter().position(|&x| x == 3);               // Some(2)
}
```

:::message
`any` と `find` は条件を満たした時点で即座に探索を打ち切ります。`collect` してから `is_empty` をチェックするのは無駄なので避けましょう。
:::

---

## 5. データ変換パイプラインの実践

### 売上データの集計

```rust
use std::collections::HashMap;

struct Sale { product: String, category: String, amount: f64, quantity: u32 }

fn main() {
    let sales = vec![
        Sale { product: "りんご".into(), category: "果物".into(), amount: 150.0, quantity: 3 },
        Sale { product: "バナナ".into(), category: "果物".into(), amount: 100.0, quantity: 5 },
        Sale { product: "トマト".into(), category: "野菜".into(), amount: 200.0, quantity: 4 },
    ];

    // カテゴリ別売上集計 -- fold で HashMap を構築
    let totals: HashMap<&str, f64> = sales.iter()
        .fold(HashMap::new(), |mut acc, s| {
            *acc.entry(s.category.as_str()).or_insert(0.0)
                += s.amount * s.quantity as f64;
            acc
        });
    for (cat, total) in &totals {
        println!("{}: {:.0}円", cat, total);
    }
}
```

### filter_map -- フィルタと変換の同時実行

```rust
fn main() {
    let items = vec!["42", "hello", "7", "world"];

    // BAD: parse を2回呼んでいる
    let _bad: Vec<i32> = items.iter()
        .filter(|s| s.parse::<i32>().is_ok())
        .map(|s| s.parse::<i32>().unwrap()).collect();

    // GOOD: filter_map で1回にまとめる
    let good: Vec<i32> = items.iter()
        .filter_map(|s| s.parse::<i32>().ok()).collect();
    println!("{:?}", good); // [42, 7]
}
```

### ゼロコスト抽象化

イテレータチェーンでは中間コレクションが一切生成されません。各要素がパイプライン全体を1つずつ通過し、コンパイラは手書きの for ループと**同一の機械語**に最適化します。

```
iter() -> map(x*2) -> filter(>5) -> collect()

要素1 -> 2 -> skip
要素3 -> 6 -> pass -> 結果に追加
要素4 -> 8 -> pass -> 結果に追加
```

`--release` ビルドでは、イテレータチェーンと手書きループの性能差はゼロです。

---

## まとめ

| 概念 | 要点 |
|------|------|
| `Vec<T>` | 最も基本的。連続メモリ、末尾操作 O(1) |
| `HashMap<K, V>` | キー値ペア。O(1) 検索。entry API で挿入/更新 |
| `HashSet<T>` | 重複なし集合。和・積・差の集合演算 |
| `BTreeMap<K, V>` | ソート順保持。範囲検索が可能 |
| `iter()` / `into_iter()` | 借用 vs 所有権ムーブ。用途で使い分ける |
| アダプタ | map / filter / flat_map 等。遅延評価で連鎖 |
| 消費メソッド | collect / fold / sum 等。実行を駆動する |
| `filter_map` | フィルタ+変換を1回で。実務頻出 |
| ゼロコスト | チェーンは手書きループと同等の性能 |

---

## やってみよう！

**演習 1: 単語頻度カウンタ**
文字列 `"the quick brown fox jumps over the lazy dog the fox"` から各単語の出現回数を `HashMap<&str, usize>` でカウントし、出現回数の降順で表示する関数を書いてください。

**演習 2: イテレータチェーン**
1〜100 の整数から「3の倍数だけ抽出 → 2乗 → 合計」を1つのイテレータチェーンで実行してください（期待値: 112455）。

**演習 3: 成績フィルタ**
`struct Student { name: String, score: u32 }` のベクタから、80点以上の学生名をスコア降順で `Vec<String>` として返す関数を、`iter` 版と `into_iter` 版の両方で実装してください。

**演習 4: 集合演算**
2つの `Vec<i32>` から積集合をソート済み `Vec<i32>` で返す関数を `HashSet` を使って書いてください。
