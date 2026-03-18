---
title: "スマートポインタ"
---

## スマートポインタとは

Rust のスマートポインタとは、通常のポインタのように振る舞いつつ、追加のメタデータや機能を持つデータ構造です。`Deref` トレイトと `Drop` トレイトを実装することで、通常の参照と同じように使いながら自動的なリソース管理を実現します。

本章では標準ライブラリが提供する主要なスマートポインタの役割と使い分けを、実践的なコード例とともに解説します。

## Box\<T\> --- ヒープ割り当てと再帰的データ構造

`Box<T>` は最もシンプルなスマートポインタです。データをヒープ上に確保し、スタック上にはポインタ（8 バイト）だけを置きます。

### 再帰型での利用

再帰的なデータ構造はコンパイル時にサイズが決まりません。`Box` で包むことでサイズを固定できます。

```rust
#[derive(Debug)]
enum List {
    Cons(i32, Box<List>),
    Nil,
}

fn main() {
    let list = List::Cons(1,
        Box::new(List::Cons(2,
            Box::new(List::Cons(3,
                Box::new(List::Nil))))));
    println!("{:?}", list);
}
```

### トレイトオブジェクト

`Box<dyn Trait>` を使うと、異なる型を同一のコレクションに格納できます。

```rust
trait Animal {
    fn sound(&self) -> &str;
}
struct Dog;
impl Animal for Dog { fn sound(&self) -> &str { "ワン" } }
struct Cat;
impl Animal for Cat { fn sound(&self) -> &str { "ニャー" } }

fn main() {
    let animals: Vec<Box<dyn Animal>> = vec![Box::new(Dog), Box::new(Cat)];
    for a in &animals {
        println!("{}", a.sound());
    }
}
```

`Box` は再帰型、大きなデータのムーブ最適化、`dyn Trait` による動的ディスパッチに使います。逆に `i32` のような小さな値を `Box` で包むのは不要なオーバーヘッドです。

## Rc\<T\> --- 参照カウントによる共有所有権

`Rc<T>`（Reference Counted）は、同一データに対して複数の所有者を許可します。最後の所有者がドロップされたときにデータが解放されます。

```rust
use std::rc::Rc;

fn main() {
    let data = Rc::new(String::from("共有データ"));
    println!("参照カウント: {}", Rc::strong_count(&data)); // 1

    let data2 = Rc::clone(&data); // カウント増加、データはコピーしない
    println!("参照カウント: {}", Rc::strong_count(&data)); // 2

    {
        let _data3 = Rc::clone(&data);
        println!("参照カウント: {}", Rc::strong_count(&data)); // 3
    } // _data3 ドロップ → カウント減少
    println!("参照カウント: {}", Rc::strong_count(&data)); // 2
}
```

### Weak\<T\> で循環参照を防ぐ

`Rc` 同士が互いを参照すると循環参照が発生し、メモリリークを起こします。`Weak<T>` は参照カウントに加算されない弱い参照で、循環を防止します。

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,      // 弱い参照
    children: RefCell<Vec<Rc<Node>>>,  // 強い参照
}
```

**注意**: `Rc<T>` は `Send` を実装しないため、スレッド間では使えません。

## Arc\<T\> --- スレッド安全な共有所有権

`Arc<T>`（Atomically Reference Counted）は `Rc<T>` のマルチスレッド版です。参照カウントの操作にアトミック命令を使います。

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let data = Arc::new(vec![1, 2, 3, 4, 5]);
    let mut handles = vec![];

    for i in 0..3 {
        let data = Arc::clone(&data);
        handles.push(thread::spawn(move || {
            let sum: i32 = data.iter().sum();
            println!("スレッド{}: 合計 = {}", i, sum);
        }));
    }
    for h in handles { h.join().unwrap(); }
}
```

| 特性 | Rc\<T\> | Arc\<T\> |
|:-----|:--------|:---------|
| スレッド安全 | No | Yes |
| カウント操作 | 通常の加減算 | アトミック操作 |
| オーバーヘッド | 小さい | やや大きい |
| 内部可変性の組み合わせ | + RefCell | + Mutex / RwLock |

単一スレッドなら `Rc`、マルチスレッドなら `Arc` を選びましょう。

## RefCell\<T\> --- 内部可変性パターン

`RefCell<T>` は、不変参照（`&self`）しか持てない場面でもデータを変更可能にします。借用ルールの検証をコンパイル時ではなく**実行時**に行う点が特徴です。

```rust
use std::cell::RefCell;

struct Logger {
    messages: RefCell<Vec<String>>,
}

impl Logger {
    fn new() -> Self {
        Logger { messages: RefCell::new(Vec::new()) }
    }
    fn log(&self, msg: &str) {
        self.messages.borrow_mut().push(msg.to_string());
    }
    fn dump(&self) {
        for msg in self.messages.borrow().iter() {
            println!("[LOG] {}", msg);
        }
    }
}

fn main() {
    let logger = Logger::new();
    logger.log("初期化完了");
    logger.log("処理開始");
    logger.dump();
}
```

### 実行時パニックに注意

`borrow()` と `borrow_mut()` を同時に呼ぶとパニックします。借用のスコープを分離して回避してください。

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(vec![1, 2, 3]);
    // OK: 借用のスコープを分離する
    {
        let r = data.borrow();
        println!("{:?}", r);
    } // r がドロップされる
    data.borrow_mut().push(4); // 安全
}
```

パニックを避けたい場合は `try_borrow()` / `try_borrow_mut()` で `Result` を返せます。

## Cell\<T\> と RefCell\<T\> の違い

`Cell<T>` も内部可変性を提供しますが、参照を取らず `get()` / `set()` で値をコピーする点が異なります。

```rust
use std::cell::Cell;

struct Counter {
    count: Cell<u32>,
}
impl Counter {
    fn new() -> Self { Counter { count: Cell::new(0) } }
    fn increment(&self) { self.count.set(self.count.get() + 1); }
}

fn main() {
    let c = Counter::new();
    c.increment();
    c.increment();
    println!("カウント: {}", c.count.get()); // 2
}
```

| 特性 | Cell\<T\> | RefCell\<T\> |
|:-----|:----------|:-------------|
| 型制約 | `T: Copy` が必要 | 任意の `T` |
| 操作 | `get()` / `set()` | `borrow()` / `borrow_mut()` |
| 参照の取得 | 不可（値コピー） | 可能 |
| パニック | なし | 借用ルール違反時 |
| オーバーヘッド | 最小 | 実行時チェックあり |

`T` が `Copy` なら `Cell`、参照が必要なら `RefCell` です。

## Cow\<\'a, T\> --- Clone on Write

`Cow<'a, T>` は読み取り専用なら借用を維持し、変更時にだけクローンします。不要なアロケーションを回避できます。

```rust
use std::borrow::Cow;

fn normalize(input: &str) -> Cow<'_, str> {
    if input.contains(' ') {
        Cow::Owned(input.replace(' ', "_")) // 変更が必要 → 新しい String
    } else {
        Cow::Borrowed(input) // 変更不要 → 借用のまま
    }
}

fn main() {
    let s1 = normalize("hello_world"); // Borrowed: アロケーションなし
    let s2 = normalize("hello world"); // Owned: 新しい String を生成
    println!("{}, {}", s1, s2);
}
```

大半の入力が変更不要な場合、`Cow` によりアロケーションをほぼゼロに抑えられます。HTML エスケープやパス正規化など、「変更が例外的なケース」で特に効果を発揮します。

## Deref トレイトと自動参照解決

`Deref` トレイトを実装すると、**Deref coercion**（自動参照解決）が働きます。

```rust
fn takes_str(s: &str) { println!("{}", s); }

fn main() {
    let boxed = Box::new(String::from("hello"));
    // Box<String> → &String → &str と自動変換される
    takes_str(&boxed);
}
```

コンパイラは以下の変換を自動で行います。

- `&T` → `&U`（`T: Deref<Target=U>` のとき）
- `&mut T` → `&mut U`（`T: DerefMut<Target=U>` のとき）
- `&mut T` → `&U`（`T: Deref<Target=U>` のとき）

この仕組みのおかげで、`&Box<String>` を `&str` を受け取る関数にそのまま渡せます。

## Drop トレイトとリソース管理

`Drop` トレイトは値がスコープを抜けたときに呼ばれるデストラクタです。RAII パターンを実現します。

```rust
struct FileGuard { name: String }

impl FileGuard {
    fn new(name: &str) -> Self {
        println!("[OPEN] {}", name);
        FileGuard { name: name.to_string() }
    }
}

impl Drop for FileGuard {
    fn drop(&mut self) { println!("[CLOSE] {}", self.name); }
}

fn main() {
    let _f1 = FileGuard::new("data.txt");
    let _f2 = FileGuard::new("log.txt");
    println!("処理中...");
} // f2 → f1 の順でドロップ（宣言の逆順）
```

- ドロップ順序は**宣言の逆順**です
- 早期解放には `std::mem::drop()` を使います
- `Drop` を実装した型は `Copy` を実装できません

## 使い分けフローチャート

```text
所有者は1つだけ？
├─ Yes → ヒープに置く必要がある？
│   ├─ Yes → Box<T>
│   └─ No  → 通常の変数で十分
└─ No（複数の所有者）
    └─ スレッドをまたぐ？
        ├─ No → Rc<T>（変更するなら Rc<RefCell<T>>）
        └─ Yes → Arc<T>
            ├─ 読み書き同程度 → Arc<Mutex<T>>
            └─ 読み取り多い   → Arc<RwLock<T>>

内部可変性だけが必要な場合:
  T: Copy → Cell<T> / それ以外 → RefCell<T>

変更時のみクローン → Cow<'a, T>
```

## まとめ

| スマートポインタ | 所有権モデル | スレッド安全 | 主な用途 |
|:----------------|:------------|:------------|:---------|
| `Box<T>` | 単一所有 | Yes | ヒープ確保、再帰型、dyn Trait |
| `Rc<T>` | 共有（単一スレッド） | No | グラフ構造、複数所有者 |
| `Arc<T>` | 共有（マルチスレッド） | Yes | スレッド間データ共有 |
| `RefCell<T>` | 内部可変性 | No | 不変参照越しの変更 |
| `Cell<T>` | 内部可変性（Copy 型） | No | 軽量な内部可変性 |
| `Cow<'a, T>` | 遅延クローン | Yes | 変更時のみアロケーション |
| `Weak<T>` | 弱参照 | Rc/Arc に準ずる | 循環参照の防止 |
| `Deref` / `Drop` | --- | --- | スマートポインタの基盤トレイト |

## やってみよう!

### 演習 1: Box で二分探索木を実装する

```rust
enum BST {
    Node { value: i32, left: Box<BST>, right: Box<BST> },
    Empty,
}
// insert(value) / contains(value) / to_sorted_vec() を実装してください
// ヒント: insert では値の大小で左右に再帰的に挿入します
```

### 演習 2: Rc\<RefCell\<T\>\> で共有カウンター

```rust
use std::rc::Rc;
use std::cell::RefCell;

struct SharedCounter { count: Rc<RefCell<i32>> }
// new() / clone_counter() / increment() / get() を実装してください
// 複数インスタンスで increment() の結果が共有されることを確認しましょう
```

### 演習 3: Cow で URL パス正規化

```rust
use std::borrow::Cow;

fn normalize_path(path: &str) -> Cow<'_, str> {
    // 1. 末尾の "/" を除去（ルート "/" は除く）
    // 2. 連続する "//" を "/" に置換
    // 3. 変更不要なら Borrowed を返す
    todo!()
}
// normalize_path("/api/users")   → Borrowed
// normalize_path("/api/users/")  → Owned("/api/users")
// normalize_path("/api//users")  → Owned("/api/users")
```
