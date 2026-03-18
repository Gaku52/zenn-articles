---
title: "並行プログラミング"
---

Rust の並行プログラミングは「恐れのない並行性（Fearless Concurrency）」と呼ばれています。所有権システムとコンパイラが、データ競合をコンパイル時に検出してくれるため、C/C++ のようなデバッグ困難なマルチスレッドバグに悩まされることがありません。この章では、スレッドの基礎から実践的な並行パターンまでを段階的に学びます。

## スレッドの生成

`std::thread::spawn` にクロージャを渡すと、新しい OS スレッドが生成されます。戻り値の `JoinHandle` を使って、スレッドの完了を待機できます。

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..=5 {
            println!("[子スレッド] カウント: {}", i);
        }
        42 // スレッドの戻り値
    });

    // join() で子スレッドの完了を待ち、戻り値を取得します
    let result = handle.join().unwrap();
    println!("子スレッドの戻り値: {}", result);
}
```

`thread::spawn` に渡すクロージャは `'static` ライフタイムを要求するため、ローカル変数への参照を直接渡すとコンパイルエラーになります。`move` キーワードで所有権をクロージャに移動させるか、後述のスコープドスレッドを使いましょう。

### スコープドスレッド

Rust 1.63 で安定化された `thread::scope` を使うと、ローカル変数への参照を子スレッドに安全に渡せます。スコープの終了時に全スレッドが自動で `join` されるため、ライフタイムの安全性が保証されます。

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3, 4, 5];
    let mut results = vec![0; 5];

    thread::scope(|s| {
        s.spawn(|| {
            let sum: i32 = data.iter().sum();
            println!("合計: {}", sum);
        });

        for (i, slot) in results.iter_mut().enumerate() {
            s.spawn(move || {
                *slot = data[i] * data[i];
            });
        }
    }); // 全スレッドの完了を自動で待機します

    println!("二乗: {:?}", results); // [1, 4, 9, 16, 25]
}
```

## Send トレイトと Sync トレイト

Rust の並行安全性を支える 2 つのマーカートレイトがあります。

- **`Send`** -- 型 `T` を別スレッドに所有権ごと移動できることを示します
- **`Sync`** -- `&T`（共有参照）を複数スレッドから同時に使えることを示します

重要な関係として、`T: Sync` ならば `&T: Send` が成り立ちます。つまり「共有参照を別スレッドに送れる」ことと「複数スレッドで同時に参照できる」ことは同義です。

```rust
use std::sync::Arc;

fn main() {
    // Rc<T> は !Send -- 参照カウントが非 Atomic のためスレッド間移動は不可
    // let rc = std::rc::Rc::new(42);
    // thread::spawn(move || println!("{}", rc)); // コンパイルエラー!

    // Arc<T> は Send + Sync -- Atomic な参照カウントで安全に共有可能
    let arc = Arc::new(42);
    let arc_clone = Arc::clone(&arc);

    let handle = std::thread::spawn(move || {
        println!("別スレッド: {}", arc_clone);
    });
    handle.join().unwrap();
    println!("メイン: {}", arc);
}
```

代表的な型の Send/Sync 対応を整理します。

| 型 | Send | Sync | 理由 |
|---|---|---|---|
| `i32`, `String` | o | o | 基本型はすべて対応 |
| `Vec<T>` | T が Send なら | T が Sync なら | 内包する型に依存 |
| `Arc<T>` | T が Send+Sync なら | T が Send+Sync なら | Atomic 参照カウント |
| `Rc<T>` | x | x | 非 Atomic な参照カウント |
| `Cell<T>` | o | x | 内部可変性がスレッド安全でない |
| `Mutex<T>` | T が Send なら | o | ロックで排他制御 |

## Mutex\<T> と RwLock\<T>

### Mutex（排他ロック）

`Mutex<T>` は、同時に 1 つのスレッドだけがデータにアクセスできるようにする排他ロックです。`.lock()` でロックを取得し、返される `MutexGuard` がスコープを抜ける（drop される）と自動的にロックが解放されます。

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0u64));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            for _ in 0..1_000 {
                let mut num = counter.lock().unwrap();
                *num += 1;
            } // MutexGuard が drop → ロック自動解放
        }));
    }

    for h in handles {
        h.join().unwrap();
    }
    println!("カウンタ: {}", *counter.lock().unwrap()); // 10000
}
```

### RwLock（読み書きロック）

`RwLock<T>` は、複数の読み取りを同時に許可しつつ、書き込みは排他的に制御します。読み取りが多く書き込みが少ない場面で `Mutex` よりも高いスループットを実現できます。

```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let config = Arc::new(RwLock::new(String::from("初期設定")));

    let config_r = Arc::clone(&config);
    let reader = thread::spawn(move || {
        let cfg = config_r.read().unwrap(); // 読み取りロック
        println!("設定値: {}", *cfg);
    });

    let config_w = Arc::clone(&config);
    let writer = thread::spawn(move || {
        let mut cfg = config_w.write().unwrap(); // 書き込みロック（排他）
        *cfg = "更新された設定".to_string();
    });

    reader.join().unwrap();
    writer.join().unwrap();
}
```

使い分けの目安は次の通りです。

| 観点 | Mutex | RwLock |
|---|---|---|
| 読み取りの並行性 | 不可（排他） | 可能 |
| 書き込みの排他性 | 排他 | 排他 |
| オーバーヘッド | 低い | やや高い |
| 向いている場面 | 書き込みが多い、ロック保持時間が短い | 読み取りが多く書き込みが少ない |

## チャネル（mpsc::channel）

チャネルはスレッド間でデータをメッセージとして送受信する仕組みです。標準ライブラリの `mpsc` は Multiple Producer, Single Consumer（複数送信・単一受信）のチャネルを提供します。

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();

    // 複数のプロデューサーを生成します
    for id in 0..3 {
        let tx = tx.clone();
        thread::spawn(move || {
            for i in 0..3 {
                let msg = format!("Producer {} - Message {}", id, i);
                tx.send(msg).unwrap();
                thread::sleep(Duration::from_millis(50));
            }
        });
    }
    drop(tx); // 元の送信側を drop してチャネルが閉じる条件を整えます

    // 全メッセージを受信します
    while let Ok(msg) = rx.recv() {
        println!("受信: {}", msg);
    }
    println!("全メッセージ処理完了");
}
```

バッファ付きチャネルが必要な場合は `mpsc::sync_channel(n)` を使います。バッファが満杯のとき、`send` は受信側が消費するまでブロックします。

## Arc\<Mutex\<T>> パターン

スレッド間でデータを共有しつつ変更する最も基本的なパターンが `Arc<Mutex<T>>` です。`Arc` がスレッド安全な参照カウントを提供し、`Mutex` が排他アクセスを保証します。

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let shared_map = Arc::new(Mutex::new(HashMap::new()));
    let mut handles = vec![];

    for i in 0..5 {
        let map = Arc::clone(&shared_map);
        handles.push(thread::spawn(move || {
            let mut m = map.lock().unwrap();
            m.insert(format!("key_{}", i), i * 10);
        }));
    }

    for h in handles {
        h.join().unwrap();
    }

    let map = shared_map.lock().unwrap();
    println!("共有マップ: {:?}", *map);
}
```

このパターンを使う際のポイントです。

1. **ロック保持時間は最小限にする** -- ロック中にネットワーク呼び出しなどの長時間処理を行わないでください
2. **クローンは `Arc::clone(&arc)` で** -- `arc.clone()` でも動きますが、`Arc::clone` と明示した方が参照カウントの増加であることが読み手に伝わります
3. **ロック取得後の `unwrap()` について** -- `lock()` が `Err` を返すのは、ロック保持中に別スレッドがパニックした場合（ポイズニング）のみです

## Rayon によるデータ並列処理

[Rayon](https://docs.rs/rayon) は、既存のイテレータコードを最小限の変更で並列化できるライブラリです。`.iter()` を `.par_iter()` に変えるだけで、ワークスティーリング方式のスレッドプールが自動的にデータを分割・並列処理します。

```toml
# Cargo.toml
[dependencies]
rayon = "1.10"
```

```rust
use rayon::prelude::*;

fn main() {
    let data: Vec<u64> = (0..10_000_000).collect();

    // 並列 filter + map + sum
    let sum: u64 = data
        .par_iter()
        .filter(|&&x| x % 2 == 0)
        .map(|&x| x * x)
        .sum();
    println!("偶数の二乗和: {}", sum);

    // 並列ソート
    let mut nums: Vec<i32> = (0..1_000_000).rev().collect();
    nums.par_sort_unstable();
    assert!(nums.windows(2).all(|w| w[0] <= w[1]));
    println!("ソート完了");
}
```

2 つの独立した処理を並列実行するには `rayon::join` が便利です。

```rust
let (result_a, result_b) = rayon::join(
    || heavy_computation_a(),
    || heavy_computation_b(),
);
```

Rayon は CPU バウンドな処理に最適です。I/O バウンドな処理（ネットワーク、ファイルアクセス等）には `tokio` などの非同期ランタイムを使いましょう。

## 並行処理のアンチパターンとデッドロック回避

### アンチパターン 1: デッドロック

複数のロックを異なる順序で取得すると、デッドロックが発生します。Rust はデータ競合をコンパイル時に防ぎますが、デッドロックは論理エラーであり、コンパイラでは検出できません。

```rust
// NG: ロック順序が逆 → デッドロック
// スレッド1: A → B の順 / スレッド2: B → A の順

// OK: 常に同じ順序（A → B）でロックを取得します
let (a1, b1) = (Arc::clone(&a), Arc::clone(&b));
let t1 = thread::spawn(move || {
    let _a = a1.lock().unwrap();
    let _b = b1.lock().unwrap(); // 常に A → B
});
let (a2, b2) = (Arc::clone(&a), Arc::clone(&b));
let t2 = thread::spawn(move || {
    let _a = a2.lock().unwrap(); // 同じ順序
    let _b = b2.lock().unwrap();
});
```

### アンチパターン 2: ロック保持中の長時間処理

ロック中にネットワーク呼び出しなどを行うと、他のスレッドが長時間ブロックされます。ロック外で処理を済ませてからロックを取得し、保持時間を最小限に抑えましょう。

### アンチパターン 3: ロック粒度が大きすぎる

構造体全体を 1 つの `Mutex` で保護すると、独立したフィールドへのアクセスでもロック競合が発生します。独立したフィールドには個別のロックを使いましょう。ただし、複数ロックの取得順序には注意が必要です。

**デッドロック回避の 3 原則**を覚えておきましょう。

1. **ロック順序を統一する** -- プロジェクト全体で「A の前に B をロックしない」というルールを決めます
2. **ロック保持時間を最小化する** -- 必要な処理だけをロック内で行います
3. **可能ならチャネルを使う** -- ロック自体を避けられるなら、メッセージパッシングの方が安全です

## Go の goroutine/channel との比較

Go と Rust はどちらも並行プログラミングを重視していますが、アプローチが異なります。

| 観点 | Rust | Go |
|---|---|---|
| 並行単位 | OS スレッド（`thread::spawn`） | goroutine（グリーンスレッド） |
| 生成コスト | 高い（MB 単位のスタック） | 低い（数 KB のスタック） |
| 数万並行 | スレッドプール（rayon等）が必要 | goroutine で自然に実現 |
| データ競合 | コンパイル時に防止 | 実行時の race detector で検出 |
| チャネル | `mpsc`（複数送信・単一受信） | 双方向、select 文で多重待ち |
| 共有メモリ | `Arc<Mutex<T>>`（型安全） | `sync.Mutex`（実行時チェック） |
| async/await | `tokio` 等の外部ランタイムが必要 | goroutine が言語組み込み |

Go の最大の強みは goroutine の軽量さです。数十万の並行タスクを低コストで実行できます。一方、Rust の強みはコンパイル時の安全性です。Go では `go vet` や race detector で実行時に検出するバグを、Rust はコンパイル段階で完全に防ぎます。Go は「共有するな、通信せよ」という思想を推奨しており、Rust でもまずはチャネルベースの設計を検討するのがよいでしょう。

## まとめ

| 項目 | 要点 |
|---|---|
| `thread::spawn` | OS スレッドを生成。`join()` で完了待ち・戻り値を取得 |
| `thread::scope` | ローカル変数の参照を安全に渡せるスコープドスレッド |
| `Send` / `Sync` | コンパイル時にデータ競合を防ぐマーカートレイト |
| `Mutex<T>` | 排他ロック。同時に 1 スレッドだけアクセス可能 |
| `RwLock<T>` | 読み取り並行・書き込み排他。読み取り優位な場面に最適 |
| `mpsc::channel` | 複数送信・単一受信のチャネル。メッセージパッシングの基本 |
| `Arc<Mutex<T>>` | スレッド間でデータを共有・変更する最も基本的なパターン |
| Rayon | `.par_iter()` で簡単にデータ並列化。CPU バウンド処理向け |
| デッドロック回避 | ロック順序の統一、保持時間の最小化、チャネルの活用 |
| Go との違い | Rust はコンパイル時安全性、Go は goroutine の軽量さが強み |

## やってみよう！

**演習 1: スレッドで並列集計**
1~100 の整数を 4 つのスレッドに均等に分割し、各スレッドで部分和を計算してください。最後にメインスレッドで合算して合計値（5050）を表示しましょう。`thread::scope` を使って実装してみてください。

**演習 2: チャネルで Producer-Consumer**
3 つの Producer スレッドがそれぞれランダムな数値を `mpsc::channel` に送信し、1 つの Consumer がすべて受信して合計を表示するプログラムを書いてください。全 Producer の送信完了後にチャネルが正しく閉じられることを確認しましょう。

**演習 3: Rayon による高速化**
100 万個の乱数ベクタに対して、通常の `iter()` と Rayon の `par_iter()` でそれぞれフィルタリング・集計を行い、処理時間を比較してください。`std::time::Instant` で計測し、何倍の高速化が得られるか確認しましょう。

**演習 4: デッドロックの体験と修正**
意図的にデッドロックを起こすコード（2 つの `Mutex` を逆順で取得）を書き、プログラムが停止することを確認してください。その後、ロック順序を統一して修正しましょう。
