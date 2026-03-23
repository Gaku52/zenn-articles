---
title: "並行パターン"
---

# Chapter 03: 並行パターン

> Worker Pool, Fan-out/Fan-in, Pipeline -- 実践的な並行処理設計

goroutineとchannelを組み合わせた実用的な並行パターンを学びます。本章では5つの定番パターンを、動くコードとともに解説します。

## Pipeline パターン

データ処理を複数のステージに分割し、各ステージをチャネルで連結する。

```
[generate] ──ch──> [filter] ──ch──> [square] ──ch──> [出力]
```

設計原則: **単一責任**（1ステージ1処理）、**チャネル所有権**（作った側がclose）、**キャンセル対応**（context必須）。

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func generate(ctx context.Context, nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for _, n := range nums {
			select {
			case <-ctx.Done():
				return
			case out <- n:
			}
		}
	}()
	return out
}

func square(ctx context.Context, in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for {
			select {
			case <-ctx.Done():
				return
			case n, ok := <-in:
				if !ok {
					return
				}
				select {
				case <-ctx.Done():
					return
				case out <- n * n:
				}
			}
		}
	}()
	return out
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	for v := range square(ctx, generate(ctx, 1, 2, 3, 4, 5)) {
		fmt.Println(v) // 1, 4, 9, 16, 25
	}
}
```

すべてのステージで `select` + `ctx.Done()` を組み合わせるのがポイント。キャンセル時にgoroutineが確実に終了する。

## Fan-out / Fan-in

1つの入力を複数ワーカーに分散（Fan-out）し、複数の出力を1つに統合（Fan-in）するパターン。

```
Input ──┬──> [Worker 1] ──┐
        ├──> [Worker 2] ──┼──> Merged Output
        └──> [Worker 3] ──┘
```

Fan-inの核心は `sync.WaitGroup` で全ワーカーの完了を待ち、完了後にmergedチャネルをcloseすること。

```go
func fanIn(ctx context.Context, channels ...<-chan int) <-chan int {
	var wg sync.WaitGroup
	merged := make(chan int)

	for _, ch := range channels {
		wg.Add(1)
		go func(c <-chan int) {
			defer wg.Done()
			for v := range c {
				select {
				case <-ctx.Done():
					return
				case merged <- v:
				}
			}
		}(ch)
	}
	go func() {
		wg.Wait()
		close(merged)
	}()
	return merged
}
```

入力チャネルを複数ワーカーが共有するとFan-out、各ワーカーの出力を `fanIn` に渡せばFan-in。結果の順序は保証されない。順序が必要なら、インデックス付き構造体を使って最後にソートする。

## Worker Pool

固定数のgoroutineでジョブキューを処理し、リソース使用量を制御する。ワーカー数の目安は、CPUバウンドなら `runtime.NumCPU()`、I/Oバウンドなら外部リソースの制約に合わせる。

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

type Job struct{ ID int }
type Result struct{ JobID int; Output string; Err error }

func workerPool(ctx context.Context, workers int, jobs <-chan Job) <-chan Result {
	results := make(chan Result)
	var wg sync.WaitGroup

	for i := 0; i < workers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for {
				select {
				case <-ctx.Done():
					return
				case job, ok := <-jobs:
					if !ok {
						return
					}
					time.Sleep(50 * time.Millisecond) // 処理
					results <- Result{JobID: job.ID, Output: fmt.Sprintf("done-%d", job.ID)}
				}
			}
		}()
	}
	go func() { wg.Wait(); close(results) }()
	return results
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	jobs := make(chan Job, 100)
	go func() {
		defer close(jobs)
		for i := 0; i < 50; i++ {
			jobs <- Job{ID: i}
		}
	}()

	for r := range workerPool(ctx, 5, jobs) {
		if r.Err != nil {
			fmt.Printf("Error job %d: %v\n", r.JobID, r.Err)
		}
	}
	fmt.Println("All jobs completed")
}
```

## errgroup によるエラーハンドリング

`golang.org/x/sync/errgroup` は、並行処理のエラー伝搬とContextキャンセルを自動化する。`sync.WaitGroup` の上位互換として使える。

```go
package main

import (
	"context"
	"fmt"
	"time"

	"golang.org/x/sync/errgroup"
)

func fetchData(ctx context.Context, name string, d time.Duration) (string, error) {
	select {
	case <-ctx.Done():
		return "", ctx.Err()
	case <-time.After(d):
		return fmt.Sprintf("%s: OK", name), nil
	}
}

func main() {
	ctx := context.Background()
	g, ctx := errgroup.WithContext(ctx)
	g.SetLimit(3) // 同時実行数を3に制限

	names := []string{"users", "orders", "reviews", "points"}
	results := make([]string, len(names))
	for i, name := range names {
		// Go 1.22+ ではループ変数のキャプチャ問題が解消されたため、
		// 「i, name := i, name」のようなコピーは不要です。
		g.Go(func() error {
			data, err := fetchData(ctx, name, 50*time.Millisecond)
			if err != nil {
				return fmt.Errorf("%s: %w", name, err)
			}
			results[i] = data
			return nil
		})
	}

	if err := g.Wait(); err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}
	for _, r := range results {
		fmt.Println(r)
	}
}
```

`errgroup` の主要API:

| メソッド | 動作 |
|---------|------|
| `g.Go(func)` | goroutineを起動。エラーが返ると他をキャンセル |
| `g.SetLimit(n)` | 同時実行数を制限 |
| `g.TryGo(func)` | リミット到達時はfalseを返す（ノンブロッキング） |

## Rate Limiter

API呼び出し頻度を制限するパターン。`golang.org/x/time/rate` が標準的な選択肢。

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"golang.org/x/time/rate"
)

func main() {
	ctx := context.Background()
	limiter := rate.NewLimiter(rate.Limit(10), 3) // 毎秒10件、バースト3

	for i := 0; i < 20; i++ {
		if err := limiter.Wait(ctx); err != nil {
			log.Fatal(err)
		}
		fmt.Printf("[%s] Request %d\n", time.Now().Format("15:04:05.000"), i)
	}
}
```

`rate.Limiter` の3つの使い方:

- **`Wait(ctx)`** -- トークン取得までブロック（最も一般的）
- **`Allow()`** -- 即座にtrue/falseを返す（ノンブロッキング）
- **`Reserve()`** -- トークンを予約し、待機時間を取得

## アンチパターン: goroutine爆発とリーク

本番で最も多い事故は **無制限goroutine起動** と **goroutineリーク** の2つ。

```go
// BAD: goroutine爆発 --> OOM
for _, item := range getItems() { // 10万件
    go process(item)
}

// GOOD: セマフォで制限
sem := make(chan struct{}, 100)
var wg sync.WaitGroup
for _, item := range items {
    wg.Add(1)
    sem <- struct{}{}
    go func(it Item) {
        defer wg.Done()
        defer func() { <-sem }()
        process(it)
    }(item)
}
wg.Wait()
```

```go
// BAD: 受信者なし --> goroutineリーク
func leaky() <-chan int {
    ch := make(chan int)
    go func() { ch <- heavyComputation() }() // 永遠にブロック
    return ch
}

// GOOD: context + バッファ1
func safe(ctx context.Context) <-chan int {
    ch := make(chan int, 1)
    go func() {
        select {
        case <-ctx.Done(): return
        case ch <- heavyComputation():
        }
    }()
    return ch
}
```

## パターン選択ガイド

| やりたいこと | 推奨パターン |
|------------|------------|
| データを段階的に変換 | Pipeline |
| 同じ処理を並列化 | Fan-out / Fan-in |
| goroutine数を制限 | Worker Pool / セマフォ |
| エラーで全体を止めたい | errgroup |
| API呼び出し頻度を制限 | Rate Limiter (`x/time/rate`) |

---

## まとめ

| 概念 | 要点 |
|------|------|
| Pipeline | データ処理を複数ステージに分割しチャネルで連結。各ステージは単一責任、作成側が `close`、`select` + `ctx.Done()` で安全停止 |
| Fan-out / Fan-in | 1入力を複数ワーカーに分散（Fan-out）し、`sync.WaitGroup` で全出力を1チャネルに統合（Fan-in）。結果順序は非保証 |
| Worker Pool | 固定数goroutineでジョブキューを処理しリソースを制御。CPU系は `runtime.NumCPU()`、I/O系は外部制約に合わせてワーカー数を決定 |
| errgroup | `sync.WaitGroup` の上位互換。`Go` でgoroutine起動、エラー発生時に自動キャンセル。`SetLimit` で同時実行数制限も可能 |
| Rate Limiter | `x/time/rate` でAPI呼び出し頻度を制限。`Wait`（ブロック）、`Allow`（即判定）、`Reserve`（予約）の3モード |
| goroutine爆発防止 | 無制限 `go` 起動はOOMの原因。セマフォ（バッファ付きチャネル）や Worker Pool で上限を設ける |
| goroutineリーク防止 | 受信者不在のチャネル送信は永久ブロック。`context` によるキャンセルとバッファ1チャネルで回避 |

---

## やってみよう！

**練習1: URLクローラー**
10個のURLを受け取り、`errgroup.SetLimit(3)` で並行にHTTP GETし、各URLのステータスコードを返す関数を実装せよ。

:::details 解答例を見る

```go
package main

import (
	"context"
	"fmt"
	"net/http"
	"time"

	"golang.org/x/sync/errgroup"
)

// CrawlResult は各URLのクロール結果
type CrawlResult struct {
	URL        string
	StatusCode int
}

// crawlURLs は最大3並行でURLをクロールし、各ステータスコードを返す
func crawlURLs(ctx context.Context, urls []string) ([]CrawlResult, error) {
	g, ctx := errgroup.WithContext(ctx)
	g.SetLimit(3) // 同時実行数を3に制限

	// インデックス指定でスライスに書き込むことでMutex不要
	// 各goroutineが異なるインデックスにアクセスするため安全
	results := make([]CrawlResult, len(urls))

	for i, url := range urls {
		g.Go(func() error {
			req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
			if err != nil {
				return fmt.Errorf("create request for %s: %w", url, err)
			}

			resp, err := http.DefaultClient.Do(req)
			if err != nil {
				return fmt.Errorf("fetch %s: %w", url, err)
			}
			defer resp.Body.Close()

			results[i] = CrawlResult{URL: url, StatusCode: resp.StatusCode}
			return nil
		})
	}

	if err := g.Wait(); err != nil {
		return nil, err // 最初のエラーを返す（他のgoroutineは自動キャンセル）
	}
	return results, nil
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	urls := []string{
		"https://httpbin.org/status/200",
		"https://httpbin.org/status/201",
		"https://httpbin.org/status/202",
		"https://httpbin.org/status/301",
		"https://httpbin.org/status/400",
		"https://httpbin.org/status/401",
		"https://httpbin.org/status/403",
		"https://httpbin.org/status/404",
		"https://httpbin.org/status/500",
		"https://httpbin.org/status/503",
	}

	results, err := crawlURLs(ctx, urls)
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}

	for _, r := range results {
		fmt.Printf("%s → %d\n", r.URL, r.StatusCode)
	}

	// 出力例:
	// https://httpbin.org/status/200 → 200
	// https://httpbin.org/status/201 → 201
	// ... (各URLのステータスコード)
}
```

**ポイント:** `errgroup.WithContext` を使うと、1つのgoroutineがエラーを返した時点で他のgoroutineのcontextもキャンセルされる。`SetLimit(3)` で同時実行数を制限することで、対象サーバーへの負荷を制御できる。

> ※ `context` の詳細は Chapter 04 で学びます。ここでは「キャンセルを伝搬する仕組み」と理解しておけばOKです。

:::

**練習2: Pipeline + Rate Limiter**
1-100の整数を生成 -> `rate.NewLimiter(rate.Limit(20), 5)` で毎秒20件に制限 -> 二乗するPipelineを構築せよ。全ステージをcontext対応にすること。

:::details 解答例を見る

```go
package main

import (
	"context"
	"fmt"
	"time"

	"golang.org/x/time/rate"
)

// generate は1〜100の整数をchannelに送信する
func generate(ctx context.Context) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for i := 1; i <= 100; i++ {
			select {
			case <-ctx.Done():
				return
			case out <- i:
			}
		}
	}()
	return out
}

// rateLimit はRate Limiterを通してデータを流す
func rateLimit(ctx context.Context, in <-chan int, limiter *rate.Limiter) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for v := range in {
			// Wait はトークンが取得できるまでブロックする
			if err := limiter.Wait(ctx); err != nil {
				return // contextキャンセル時に終了
			}
			select {
			case <-ctx.Done():
				return
			case out <- v:
			}
		}
	}()
	return out
}

// square は受信した値を二乗して送信する
func square(ctx context.Context, in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for v := range in {
			select {
			case <-ctx.Done():
				return
			case out <- v * v:
			}
		}
	}()
	return out
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	// 毎秒20件、バースト5のRate Limiter
	limiter := rate.NewLimiter(rate.Limit(20), 5)

	// Pipeline: generate → rateLimit → square → 出力
	start := time.Now()
	count := 0
	for v := range square(ctx, rateLimit(ctx, generate(ctx), limiter)) {
		count++
		fmt.Printf("[%6s] %d: %d\n",
			time.Since(start).Truncate(time.Millisecond), count, v)
	}

	// 出力例:
	// [   0ms] 1: 1
	// [   0ms] 2: 4      ← バースト5件は即座に処理
	// ...
	// [ 250ms] 6: 36     ← 以降は50msごと（毎秒20件）
}
```

**ポイント:** Rate LimiterをPipelineの1ステージとして挟むのが定番パターン。`limiter.Wait(ctx)` はcontextのキャンセルにも対応しているため、タイムアウト時にPipeline全体が確実に停止する。

> ※ `context` の詳細は Chapter 04 で学びます。

:::

**練習3: タイムアウト付きFan-out**
5つのAPIエンドポイント（`time.Sleep`でシミュレート可）を並行に呼び出し、3秒以内に全結果を集約する関数を実装せよ。1つでもタイムアウトしたら `context.DeadlineExceeded` を返すこと。

:::details 解答例を見る

```go
package main

import (
	"context"
	"fmt"
	"math/rand"
	"sync"
	"time"
)

// APIResult はAPIの呼び出し結果
type APIResult struct {
	Endpoint string
	Data     string
	Duration time.Duration
}

// callAPI はAPIエンドポイントの呼び出しをシミュレートする
func callAPI(ctx context.Context, endpoint string, latency time.Duration) (APIResult, error) {
	select {
	case <-ctx.Done():
		return APIResult{}, ctx.Err() // タイムアウト or キャンセル
	case <-time.After(latency):
		return APIResult{
			Endpoint: endpoint,
			Data:     fmt.Sprintf("response from %s", endpoint),
			Duration: latency,
		}, nil
	}
}

// fanOutWithTimeout は全APIを並行に呼び出し、全結果を集約する
// 3秒以内に完了しなければ context.DeadlineExceeded を返す
func fanOutWithTimeout(endpoints []string) ([]APIResult, error) {
	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()

	var (
		mu      sync.Mutex
		wg      sync.WaitGroup
		results []APIResult
		firstErr error
	)

	for _, ep := range endpoints {
		wg.Add(1)
		go func(endpoint string) {
			defer wg.Done()

			// 各APIは0〜4秒のランダムなレイテンシ
			latency := time.Duration(rand.Intn(4000)+500) * time.Millisecond
			result, err := callAPI(ctx, endpoint, latency)

			mu.Lock()
			defer mu.Unlock()
			if err != nil {
				if firstErr == nil {
					firstErr = fmt.Errorf("%s: %w", endpoint, err)
				}
				return
			}
			results = append(results, result)
		}(ep)
	}

	wg.Wait()

	// 1つでもタイムアウトしたらエラーを返す
	if firstErr != nil {
		return nil, firstErr
	}
	return results, nil
}

func main() {
	endpoints := []string{
		"/api/users",
		"/api/orders",
		"/api/products",
		"/api/reviews",
		"/api/inventory",
	}

	results, err := fanOutWithTimeout(endpoints)
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}

	fmt.Printf("All %d APIs responded within deadline:\n", len(results))
	for _, r := range results {
		fmt.Printf("  %s (%v)\n", r.Endpoint, r.Duration)
	}

	// 出力例（全API が3秒以内に応答した場合）:
	// All 5 APIs responded within deadline:
	//   /api/users (1.2s)
	//   /api/orders (0.8s)
	//   ...
	//
	// タイムアウトした場合:
	// Error: /api/inventory: context deadline exceeded
}
```

**ポイント:** `context.WithTimeout` で全goroutineに共通のデッドラインを設定する。`callAPI` 内の `select` で `ctx.Done()` を監視することで、デッドライン到達時に全goroutineが即座に終了する。`sync.WaitGroup` で全goroutineの完了を確実に待つことで、goroutineリークを防ぐ。

> ※ `context` の詳細は Chapter 04 で学びます。

:::

