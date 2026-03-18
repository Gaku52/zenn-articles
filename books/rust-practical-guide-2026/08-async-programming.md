---
title: "非同期プログラミング"
---

## async/await の基本概念

Rust の非同期プログラミングは、少数のスレッドで大量の I/O 待ちタスクを効率的に処理する仕組みです。同期モデルでは 1 万の同時接続に 1 万のスレッドが必要ですが、非同期モデルでは CPU コア数程度のスレッドで数万の接続を捌けます。

```rust
// 同期: スレッドが I/O 待ちでブロックされる
fn sync_handler(stream: TcpStream) {
    let data = read_from_db();       // ~10ms ブロック
    let enriched = call_api(data);   // ~50ms ブロック
    stream.write_all(&enriched);
}

// 非同期: I/O 待ち中にスレッドが他のタスクを処理
async fn async_handler(stream: TcpStream) {
    let data = read_from_db().await;       // 中断 → 再開
    let enriched = call_api(data).await;   // 中断 → 再開
    stream.write_all(&enriched).await;
}
```

`async fn` は呼び出しただけでは実行されません。`Future` を返すだけであり、`.await` するか `spawn` して初めて実行が開始されます。

## Future トレイトの仕組み

`async fn` はコンパイラによって `Future` トレイトを実装する状態マシンに変換されます。

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),   // 完了。値 T を返す
    Pending,    // 未完了。Waker で再通知される
}
```

ランタイム（Executor）が `poll()` を呼び、`Ready(value)` なら完了、`Pending` ならランタイムは他のタスクに切り替えます。I/O が完了すると `Waker` 経由で再び `poll()` が呼ばれます。各 `.await` ポイントが状態遷移点となり、ヒープアロケーションなしで非同期処理が実現されます。これが Rust の「ゼロコスト抽象化」です。

## Tokio ランタイムの導入と設定

Rust の非同期ランタイムは言語に組み込まれておらず、ユーザーが選択します。実務では Tokio が事実上のスタンダードです。

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

Tokio は **Executor**（タスク実行）、**Reactor**（I/O 監視）、**Waker**（完了通知）の 3 層構造です。

```rust
// マルチスレッド（デフォルト）
#[tokio::main]
async fn main() { /* ... */ }

// シングルスレッド（テスト・軽量用途）
#[tokio::main(flavor = "current_thread")]
async fn main() { /* ... */ }

// 手動ビルド（詳細制御）
fn main() {
    let runtime = tokio::runtime::Builder::new_multi_thread()
        .worker_threads(4)
        .enable_all()
        .build()
        .unwrap();
    runtime.block_on(async { /* ... */ });
}
```

## `#[tokio::main]` と `#[tokio::test]`

`#[tokio::main]` はランタイムを起動して `block_on` を呼ぶマクロです。テストでは `#[tokio::test]` を使います。

```rust
// #[tokio::main] は以下と等価
fn main() {
    tokio::runtime::Builder::new_multi_thread()
        .enable_all().build().unwrap()
        .block_on(async { /* ... */ })
}

// テスト（デフォルトはシングルスレッド）
#[tokio::test]
async fn test_basic() {
    assert_eq!(my_async_fn().await, 42);
}

#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
async fn test_multi() {
    let (tx, mut rx) = tokio::sync::mpsc::channel(1);
    tokio::spawn(async move { tx.send(42).await.unwrap() });
    assert_eq!(rx.recv().await, Some(42));
}
```

## spawn によるタスク生成

`tokio::spawn` は新しい非同期タスクを生成します。`Send + 'static` 制約があるため、参照を含む Future は spawn できません。所有権を `move` で移します。

```rust
let handle = tokio::spawn(async {
    tokio::time::sleep(Duration::from_millis(100)).await;
    42
});
let result = handle.await.unwrap(); // 結果: 42

// CPU 集中処理は spawn_blocking でブロッキングプールに逃がす
let sum = tokio::task::spawn_blocking(|| {
    (0..10_000_000u64).sum::<u64>()
}).await.unwrap();
```

## `join!` と `select!` マクロ

```rust
async fn task_a() -> String { sleep(Duration::from_millis(100)).await; "A".into() }
async fn task_b() -> String { sleep(Duration::from_millis(200)).await; "B".into() }
async fn task_c() -> String { sleep(Duration::from_millis(50)).await;  "C".into() }

#[tokio::main]
async fn main() {
    // join!: 全 Future を並行実行し全完了を待つ（200ms で完了）
    let (a, b, c) = tokio::join!(task_a(), task_b(), task_c());

    // try_join!: いずれかが Err を返した時点で即座に返る
    let (user, profile) = tokio::try_join!(fetch_user(1), fetch_profile(1))?;

    // select!: 最初に完了した Future の結果を取得（他は drop）
    tokio::select! {
        val = task_a() => println!("A が先: {}", val),
        val = task_c() => println!("C が先: {}", val), // 50ms で最速
    }
}
```

動的な数のタスクには `JoinSet` を使います。

```rust
let mut set = tokio::task::JoinSet::new();
for id in 1..=10 {
    set.spawn(async move { format!("Item#{}", id) });
}
while let Some(result) = set.join_next().await {
    println!("{}", result.unwrap()); // 完了順に取得
}
```

## `tokio::sync` の同期プリミティブ

### Mutex / RwLock

```rust
use std::sync::Arc;
use tokio::sync::{Mutex, RwLock};

// Mutex: .await 中もロック保持可能（.await を含まないなら std 版が軽量）
let data = Arc::new(Mutex::new(vec![]));
let d = data.clone();
tokio::spawn(async move {
    let mut lock = d.lock().await;
    lock.push(42);
    tokio::time::sleep(Duration::from_millis(10)).await; // 安全
    lock.push(43);
});

// RwLock: 読み取りは並行、書き込みは排他
let config = Arc::new(RwLock::new("initial".to_string()));
let guard = config.read().await;
println!("{}", *guard);
```

### チャネル（mpsc, oneshot, broadcast, watch）

```rust
// mpsc: 多対一（バックプレッシャー付き）
let (tx, mut rx) = mpsc::channel::<String>(32);
tx.send("hello".into()).await.unwrap();

// oneshot: 一対一・一回限り（リクエスト-レスポンスに最適）
let (reply_tx, reply_rx) = oneshot::channel();
reply_tx.send(42).unwrap();
let answer = reply_rx.await.unwrap();

// broadcast: 多対多（全受信者にクローン送信）
let (btx, _) = broadcast::channel::<String>(16);
let mut brx = btx.subscribe();
btx.send("通知".into()).unwrap();

// watch: 最新値のみ保持（設定変更通知に最適）
let (wtx, mut wrx) = watch::channel("初期値".to_string());
wtx.send("更新値".into()).unwrap();
wrx.changed().await.unwrap();
```

## 非同期のキャンセルとタイムアウト

```rust
// タイムアウト
let result = tokio::time::timeout(Duration::from_secs(2), slow_op()).await;
match result {
    Ok(value) => println!("成功: {}", value),
    Err(_) => println!("タイムアウト"),
}

// タスクキャンセル（次の .await で停止。Drop は正しく実行される）
let handle = tokio::spawn(async { /* ... */ });
handle.abort();
```

複数タスクの協調的なキャンセルには `CancellationToken` が推奨されます。

```rust
use tokio_util::sync::CancellationToken;

let token = CancellationToken::new();
let child = token.child_token(); // 親キャンセルで自動キャンセル

tokio::spawn(async move {
    loop {
        tokio::select! {
            _ = child.cancelled() => { println!("停止"); break; }
            _ = sleep(Duration::from_secs(1)) => { println!("作業中..."); }
        }
    }
});

token.cancel(); // 全タスクにキャンセルを通知
```

## JS/Python の async/await との違い

| 特徴 | Rust | JavaScript | Python |
|---|---|---|---|
| 実行モデル | ゼロコスト Future | Promise（ヒープ） | コルーチン（ヒープ） |
| ランタイム | ユーザー選択 | V8 組み込み | asyncio 標準 |
| スレッドモデル | マルチスレッド | シングルスレッド | シングルスレッド（GIL） |
| 遅延評価 | `.await` まで実行されない | Promise は即座に実行 | `.await` まで実行されない |

重要な違いは 3 点です。(1) Rust の Future は `.await` まで何も起きない遅延評価。(2) ランタイムが言語に組み込まれていない。(3) `spawn` に `Send + 'static` 制約がある。

## 実務パターン: 複数の非同期タスクの協調

### アクターモデル（mpsc + oneshot）

```rust
enum Command {
    Get { key: String, reply: oneshot::Sender<Option<String>> },
    Set { key: String, value: String, reply: oneshot::Sender<bool> },
}

async fn db_actor(mut rx: mpsc::Receiver<Command>) {
    let mut store = std::collections::HashMap::new();
    while let Some(cmd) = rx.recv().await {
        match cmd {
            Command::Get { key, reply } => { let _ = reply.send(store.get(&key).cloned()); }
            Command::Set { key, value, reply } => { store.insert(key, value); let _ = reply.send(true); }
        }
    }
}
```

### グレースフルシャットダウン（watch + select!）

```rust
let (shutdown_tx, _) = watch::channel(false);
let mut rx = shutdown_tx.subscribe();

tokio::spawn(async move {
    loop {
        tokio::select! {
            _ = rx.changed() => { if *rx.borrow() { return; } }
            _ = sleep(Duration::from_secs(1)) => { println!("処理中..."); }
        }
    }
});

// シャットダウンシグナル送信
let _ = shutdown_tx.send(true);
```

## まとめ

| 項目 | 要点 |
|---|---|
| `async/await` | Future を生成・待機する構文。呼んだだけでは実行されない |
| `Future` トレイト | `poll()` が `Ready` か `Pending` を返す遅延評価モデル |
| Tokio | Executor + Reactor + Waker の 3 層構造 |
| `#[tokio::main]` / `#[tokio::test]` | ランタイム起動マクロ |
| `tokio::spawn` | 非同期タスク生成。`Send + 'static` が必要 |
| `spawn_blocking` | CPU 集中処理をブロッキングプールに逃がす |
| `join!` / `try_join!` | 全 Future を並行実行し全完了を待つ |
| `select!` | 最初に完了した Future の結果を取得 |
| `JoinSet` | 動的な数のタスクを管理 |
| `tokio::sync::Mutex` | `.await` 中もロックを安全に保持 |
| `mpsc` / `oneshot` / `broadcast` / `watch` | 用途別チャネル |
| `timeout` / `CancellationToken` | タイムアウトと協調的キャンセル |

## やってみよう!

**演習 1: 並行 API コール**
3 つの URL に `reqwest::get` を `join!` で並行リクエストし、全レスポンスのバイト数を合計する関数を書いてください。`try_join!` 版も実装しましょう。

**演習 2: タイムアウト付きリトライ**
最大 3 回リトライし、各試行に 2 秒のタイムアウトを設定する `retry_with_timeout` 関数を `tokio::time::timeout` で実装してください。

**演習 3: アクターモデルのカウンター**
`mpsc` + `oneshot` で `increment`・`get_value` を受け付けるアクターを実装し、10 タスクから並行に 100 回ずつ increment して最終値が 1000 であることを確認しましょう。

**演習 4: グレースフルシャットダウン**
`watch` + `select!` で 3 つのワーカーがシャットダウンシグナルを受けて安全に停止するプログラムを書いてください。`tokio::signal::ctrl_c` をトリガーにしましょう。

**演習 5: 並行度制限クローラー**
URL リストを最大 5 並行で取得する関数を `JoinSet` で実装してください。各ページの `<title>` タグの内容を抽出して返す仕様です。
