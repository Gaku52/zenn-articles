---
title: "goroutineとchannel"
---

# Chapter 01: goroutineとchannel

> Goの並行処理の基盤 -- 軽量スレッドと型安全な通信路

Goの並行処理は **goroutine**（軽量な実行単位）と **channel**（goroutine間の通信路）の2つで成り立っています。この章ではこの2つの基本と、実践で使うパターンを身につけます。

---

## goroutineの基本

`go` キーワードを関数呼び出しの前に付けるだけで、新しいgoroutineが起動する。

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
)

func main() {
	fmt.Printf("論理CPU数: %d\n", runtime.NumCPU())

	var wg sync.WaitGroup
	const N = 100_000

	// 10万のgoroutineを起動しても問題なく動作する
	for i := 0; i < N; i++ {
		wg.Add(1) // 必ずgo文の前でAddする
		go func(id int) {
			defer wg.Done()
			_ = id // 各goroutineは約2KBのスタックで開始
		}(i)
	}

	wg.Wait()
	fmt.Printf("全 %d goroutine 完了\n", N)
}
```

**押さえておくポイント:**

- goroutineはOSスレッドではない。GoランタイムがM:Nスケジューリングで多重化する
- 初期スタックは約2KB（OSスレッドは~1MB）なので、数十万単位で起動できる
- `sync.WaitGroup` の `Add` は **必ず `go` 文の前** で呼ぶ。goroutine内で呼ぶと `Wait` が先に実行される可能性がある
- `time.Sleep` で完了を待つのはアンチパターン。`WaitGroup` かchannelで同期する

---

## channelの基本

channelはgoroutine間でデータを安全にやり取りするための型付きパイプ。

### バッファなしchannel（同期channel）

送信と受信が **同時に** 行われる。これを「ランデブー」と呼ぶ。

```go
package main

import "fmt"

func main() {
	ch := make(chan int) // バッファなし

	go func() {
		ch <- 42 // 受信されるまでブロック
	}()

	value := <-ch // 送信されるまでブロック
	fmt.Println(value) // 42
}
```

完了通知だけが目的なら、ゼロバイトの `chan struct{}` がイディオム。

```go
done := make(chan struct{})
go func() {
	defer close(done) // closeで全受信者に通知できる
	heavyWork()
}()
<-done
```

### バッファ付きchannel

バッファに空きがあれば送信はブロックしない。

```go
ch := make(chan string, 3) // バッファサイズ3
ch <- "a" // ブロックしない
ch <- "b"
ch <- "c"
// ch <- "d" // ここでブロック（バッファ満杯）
fmt.Println(<-ch) // "a" (FIFO)
```

**バッファサイズの目安:**

| サイズ | 用途 | 例 |
|--------|------|-----|
| 0 | 同期が必要な場合 | `done := make(chan struct{})` |
| 1 | シグナリング | `signal.Notify(sigCh, ...)` |
| N | 生産者/消費者の速度差吸収 | `jobs := make(chan Job, 100)` |

大きすぎるバッファはメモリを浪費し、問題の発見を遅らせる。

---

## チャネルの方向制約

関数のシグネチャでチャネルの方向を制約すると、誤用をコンパイル時に防げる。

```go
package main

import "fmt"

// chan<- int: 送信専用。受信するとコンパイルエラー
func producer(out chan<- int) {
	for i := 0; i < 5; i++ {
		out <- i * i
	}
	close(out) // 送信側がcloseする
}

// <-chan int: 受信専用。送信するとコンパイルエラー
func consumer(in <-chan int) {
	for v := range in { // closeされるまでループ
		fmt.Println(v)
	}
}

func main() {
	ch := make(chan int, 5)
	go producer(ch) // chan int → chan<- int に暗黙変換
	consumer(ch)    // chan int → <-chan int に暗黙変換
}
```

チャネル操作の結果を表にまとめておく。panicになるケースは確実に覚えておこう。

| 操作 | nilチャネル | クローズ済み | 正常 |
|------|-----------|------------|------|
| 送信 `ch <-` | 永久ブロック | **panic** | 送信 |
| 受信 `<-ch` | 永久ブロック | ゼロ値, false | 受信 |
| `close` | **panic** | **panic** | クローズ |

---

## select文

selectは複数のchannel操作を同時に待ち受ける。複数が同時に準備完了ならランダムに1つを選ぶ。

```go
ch1 := make(chan string)
ch2 := make(chan string)

go func() { time.Sleep(100 * time.Millisecond); ch1 <- "one" }()
go func() { time.Sleep(200 * time.Millisecond); ch2 <- "two" }()

for i := 0; i < 2; i++ {
	select {
	case msg := <-ch1:
		fmt.Println("ch1:", msg)
	case msg := <-ch2:
		fmt.Println("ch2:", msg)
	}
}
```

selectの3つの使いどころを押さえよう。

```go
// 1. タイムアウト/キャンセル — ctx.Done()と組み合わせる
select {
case r := <-result:
	return r, nil
case <-ctx.Done():
	return "", ctx.Err()
}

// 2. ノンブロッキング — defaultで即座にリターン
select {
case v := <-ch:
	return v, true
default:
	return 0, false
}

// 3. 定期処理 — ticker.Cとctx.Done()の組み合わせ
for {
	select {
	case <-ticker.C:
		doWork()
	case <-ctx.Done():
		return
	}
}
```

---

## 実践パターン

### パターン1: Fan-out / Fan-in

1つのソースから複数のワーカーに分配し、結果を1つにマージする。

```
            ┌──> Worker1 ──┐
  Source ───┼──> Worker2 ──┼──> Merged Output
            ├──> Worker3 ──┤
            └──> Worker4 ──┘
```

```go
func fanIn(channels ...<-chan int) <-chan int {
	var wg sync.WaitGroup
	merged := make(chan int)
	for _, ch := range channels {
		wg.Add(1)
		go func(c <-chan int) {
			defer wg.Done()
			for v := range c { merged <- v }
		}(ch)
	}
	go func() { wg.Wait(); close(merged) }()
	return merged
}

func worker(jobs <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for v := range jobs { out <- v * v }
	}()
	return out
}

func main() {
	jobs := make(chan int, 10)
	go func() {
		defer close(jobs)
		for i := 1; i <= 20; i++ { jobs <- i }
	}()

	// Fan-out: 3ワーカーに分配 → Fan-in: 結果マージ
	for r := range fanIn(worker(jobs), worker(jobs), worker(jobs)) {
		fmt.Println(r)
	}
}
```

### パターン2: errgroupによる並行処理

`golang.org/x/sync/errgroup` はWaitGroup + エラー集約 + context連携のオールインワン。実務で最も使う。

```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, userID string) (*Profile, []Order, error) {
	g, ctx := errgroup.WithContext(ctx)
	g.SetLimit(5) // 同時実行数を制限

	var (
		profile *Profile
		orders  []Order
	)
	g.Go(func() error {
		var err error
		profile, err = fetchProfile(ctx, userID)
		return err
	})
	g.Go(func() error {
		var err error
		orders, err = fetchOrders(ctx, userID)
		return err
	})

	if err := g.Wait(); err != nil {
		return nil, nil, err // 最初のエラーを返す
	}
	return profile, orders, nil
}
```

### 最重要アンチパターン: goroutineリーク

goroutineが終了せずに溜まり続ける問題。本番障害の原因になりやすい。

```go
// BAD: 2つ起動して1つだけ受信。もう1つは永遠にブロック
func leakySearch(query string) string {
	ch := make(chan string)
	go func() { ch <- searchAPI1(query) }()
	go func() { ch <- searchAPI2(query) }()
	return <-ch // 1つだけ受信 → もう1つのgoroutineがリーク
}

// GOOD: バッファ付きchannelで全goroutineが送信可能に
func safeSearch(query string) string {
	ch := make(chan string, 2) // 全goroutineが送信できるサイズ
	go func() { ch <- searchAPI1(query) }()
	go func() { ch <- searchAPI2(query) }()
	return <-ch
}
```

**リーク防止の鉄則:**

1. goroutineを起動したら「どうやって終了するか」を必ず設計する
2. 長時間動くgoroutineには `context.Context` によるキャンセルを組み込む
3. テストでは `go.uber.org/goleak` でリーク検出する

---

## まとめ

| 概念 | 要点 |
|------|------|
| goroutine | `go` キーワードで起動する軽量実行単位。初期スタック約2KBで数十万単位の起動が可能 |
| sync.WaitGroup | goroutineの完了待ちに使う。`Add` は必ず `go` 文の前で呼ぶこと |
| バッファなしchannel | 送信と受信が同時に行われる（ランデブー）。完了通知には `chan struct{}` が定番 |
| バッファ付きchannel | バッファに空きがあれば送信はブロックしない。サイズは用途に応じて最小限に |
| チャネル方向制約 | `chan<-`（送信専用）`<-chan`（受信専用）で誤用をコンパイル時に防止 |
| select文 | 複数のchannel操作を同時に待ち受ける。タイムアウト・ノンブロッキング・定期処理の3用途 |
| Fan-out / Fan-in | 1つのソースから複数ワーカーに分配し、結果を1つにマージするパターン |
| errgroup | WaitGroup + エラー集約 + context連携のオールインワン。実務で最頻出 |
| goroutineリーク | 終了しないgoroutineが溜まる問題。起動時に「どう終了するか」を必ず設計する |

---

## やってみよう！

### 練習1: 並行カウンター

3つのgoroutineがそれぞれ1000回ずつカウンターをインクリメントする処理を実装してください。`sync.WaitGroup` で全goroutineの完了を待ち、最終的なカウンター値が3000になることを確認してください。

**ヒント:** カウンターへのアクセスを `sync.Mutex` または `atomic.Int64` で保護する必要がある。

### 練習2: パイプライン

以下の3ステージのパイプラインをchannelで構築してください。

1. **generate**: 1から20までの整数をchannelに送信
2. **square**: 受信した値を二乗してchannelに送信
3. **filter**: 偶数だけをchannelに送信

各ステージはgoroutineで動作し、channelの方向制約（`<-chan`, `chan<-`）を使うこと。`context.Context` によるキャンセルにも対応してください。

### 練習3: タイムアウト付きHTTPクライアント

`select` と `time.After` を使い、3つのURL（任意）に同時にHTTPリクエストを投げ、**最初に返ってきたレスポンス**のステータスコードを表示するプログラムを書いてください。全体のタイムアウトは5秒とします。

**ヒント:** `context.WithTimeout` を使う方法もある。両方試して違いを比較してみよう。
