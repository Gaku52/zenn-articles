---
title: "同期プリミティブ"
---

# Chapter 02: 同期プリミティブ

> Mutex, WaitGroup, Once — 共有リソースを安全に扱う

## sync.Mutex — 排他制御の基本

`sync.Mutex` はクリティカルセクションを保護する最も基本的な同期プリミティブです。

```go
type SafeCounter struct {
    mu sync.Mutex
    v  map[string]int
}

func (c *SafeCounter) Inc(key string) {
    c.mu.Lock()
    defer c.mu.Unlock() // defer で確実に解放
    c.v[key]++
}

func (c *SafeCounter) Value(key string) int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.v[key]
}
```

`defer c.mu.Unlock()` は鉄則です。早期リターンやpanicでもロックが確実に解放されます。

### Mutex の動作モデル

Go の Mutex は2モードで動作します。

```
通常モード: 新規goroutineがスピンでCAS → 高性能だが不公平
飢餓モード: 待機が1ms超の場合に遷移 → FIFO順序で公平にロック付与
```

---

## sync.RWMutex — 読み書きロック

読み取りが多く書き込みが少ない場合に有効です。複数のgoroutineが同時に `RLock` を取得でき、`Lock` は排他的です。

```go
type ConfigManager struct {
    mu     sync.RWMutex
    config map[string]string
}

func NewConfigManager() *ConfigManager {
    return &ConfigManager{config: make(map[string]string)}
}

// 読み取りは複数同時可能
func (cm *ConfigManager) Get(key string) string {
    cm.mu.RLock()
    defer cm.mu.RUnlock()
    return cm.config[key]
}

// GetAll はスナップショットを返す
func (cm *ConfigManager) GetAll() map[string]string {
    cm.mu.RLock()
    defer cm.mu.RUnlock()
    snapshot := make(map[string]string, len(cm.config))
    for k, v := range cm.config {
        snapshot[k] = v
    }
    return snapshot // コピーを返す（ロック外での変更を防ぐ）
}

// 書き込みは排他
func (cm *ConfigManager) Set(key, value string) {
    cm.mu.Lock()
    defer cm.mu.Unlock()
    cm.config[key] = value
}
```

読み取りが95%以上の場合に `RWMutex` が有利。書き込みが50%以上なら `Mutex` の方が良いケースもあります（RWMutexのオーバーヘッドのため）。

---

## sync.WaitGroup — goroutine の完了待ち

複数のgoroutineの完了を待つ定番パターンです。

```go
func fetchAll(urls []string) []string {
    var (
        mu      sync.Mutex
        results []string
        wg      sync.WaitGroup
    )

    for _, url := range urls {
        wg.Add(1)
        go func(u string) {
            defer wg.Done()
            resp, err := http.Get(u)
            if err != nil {
                return
            }
            defer resp.Body.Close()
            body, _ := io.ReadAll(resp.Body)

            mu.Lock()
            results = append(results, string(body))
            mu.Unlock()
        }(url)
    }

    wg.Wait() // 全goroutineの完了を待つ
    return results
}
```

**注意**: `wg.Add(1)` は必ず `go func()` の**前**で呼ぶこと。goroutine内で呼ぶとレースになります。

---

## sync.Once — 一度だけの初期化

```go
var (
    instance *Database
    once     sync.Once
)

func GetDB() *Database {
    once.Do(func() {
        // 何百回呼ばれても1度だけ実行
        instance = &Database{conn: connectDB()}
    })
    return instance
}
```

### Go 1.21+ の OnceValue / OnceValues

```go
// sync.OnceValue: 値を1回だけ計算
getConfig := sync.OnceValue(func() *Config {
    cfg, err := loadConfig("config.yaml")
    if err != nil {
        panic(err) // panicは再呼び出し時にも再送出
    }
    return cfg
})

cfg := getConfig() // 初回: ファイル読み込み
cfg = getConfig()  // 2回目: キャッシュ値を返す

// sync.OnceValues: 値+エラーを返す版
loadCert := sync.OnceValues(func() (*tls.Certificate, error) {
    return tls.LoadX509KeyPair("cert.pem", "key.pem")
})

cert, err := loadCert() // 初回: ファイル読み込み
cert, err = loadCert()  // 2回目: キャッシュ結果を返す
```

---

## sync/atomic — ロックフリーの高速操作

単純なカウンタやフラグには `atomic` がMutexより高速です。Go 1.19+ で型安全なAPIが追加されました。

```go
type Metrics struct {
    RequestCount  atomic.Int64
    ErrorCount    atomic.Int64
    ActiveConns   atomic.Int32
    IsHealthy     atomic.Bool
}

func (m *Metrics) RecordRequest(success bool) {
    m.RequestCount.Add(1)
    if !success {
        m.ErrorCount.Add(1)
    }
}

func (m *Metrics) ConnectionOpened()  { m.ActiveConns.Add(1) }
func (m *Metrics) ConnectionClosed() { m.ActiveConns.Add(-1) }

// atomic.Value で任意の値を安全に読み書き
var config atomic.Value // *Config を格納

func UpdateConfig(cfg *Config) { config.Store(cfg) }
func GetConfig() *Config       { return config.Load().(*Config) }
```

---

## sync.Pool — オブジェクトの再利用

GC負荷を軽減するための一時オブジェクトプールです。**キャッシュではありません**（GCで予告なく回収されます）。

```go
var bufPool = sync.Pool{
    New: func() any {
        return new(bytes.Buffer)
    },
}

func processRequest(data []byte) string {
    buf := bufPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()       // 必ずリセットしてから
        bufPool.Put(buf)  // プールに返却
    }()

    buf.Write(data)
    return buf.String()
}
```

適切な用途: バッファ、エンコーダ等の頻繁に割り当て・解放される一時オブジェクト。
不適切: DB接続プール、長寿命オブジェクト、キャッシュ。

---

## sync.Map — 並行安全マップ

特定のパターン（キーが安定、読み取り主体）で有効ですが、一般的には `map + RWMutex` を推奨します。

| 項目 | sync.Map | map + RWMutex |
|------|----------|---------------|
| 読み取り性能 | 非常に高速 | 高速 |
| 書き込み性能 | 低〜中 | 中 |
| 型安全性 | `any`（型アサーション必要） | ジェネリクスで型安全 |
| 推奨場面 | キー安定・読み取り主体 | 一般的な用途 |

---

## 選択ガイド

| プリミティブ | 用途 | コスト |
|-------------|------|--------|
| sync.Mutex | 単純な排他制御 | 中 |
| sync.RWMutex | 読み多・書き少 | 中 |
| sync.WaitGroup | goroutine群の完了待ち | 低 |
| sync.Once | 1回だけの初期化 | 低 |
| sync.Pool | 一時オブジェクト再利用 | 低 |
| sync.Map | 特定パターンの並行map | 中 |
| atomic.Int64 | 単純なカウンタ | 最低 |
| atomic.Value | 任意値のアトミック読み書き | 低 |
| channel | データの所有権移転 | 中〜高 |

---

## アンチパターン

### Mutexのコピー

```go
// NG: 値レシーバ → Mutexがコピーされる！
func (c Counter) Value() int {
    c.mu.Lock() // コピーされた別のMutexをロック
    defer c.mu.Unlock()
    return c.n
}

// OK: ポインタレシーバ
func (c *Counter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.n
}
// go vet で "copies lock value" として検出可能
```

### デッドロック（ロック順序の不一致）

```go
// NG: goroutine1は A→B、goroutine2は B→A の順でロック
func transfer(from, to *Account, amount int) {
    from.mu.Lock()  // デッドロックの可能性！
    to.mu.Lock()
}

// OK: ID順で一貫したロック順序
func transfer(from, to *Account, amount int) {
    first, second := from, to
    if from.ID > to.ID {
        first, second = to, from
    }
    first.mu.Lock()
    second.mu.Lock()
    defer first.mu.Unlock()
    defer second.mu.Unlock()

    from.Balance -= amount
    to.Balance += amount
}
```

---

## まとめ

| 概念 | 要点 |
|------|------|
| Mutex | 排他制御の基本。`defer Unlock()` を常に使う |
| RWMutex | 読み取り多の場合に性能向上 |
| WaitGroup | goroutine群の完了待ち。`Add` は `go` の前で |
| Once | 初期化を1回だけ安全に実行。Go 1.21+ で OnceValue 追加 |
| atomic | ロックフリーの高速な値操作。Go 1.19+ で型安全API |
| Pool | 一時オブジェクトの再利用でGC負荷低減。キャッシュではない |
| sync.Map | 特定パターン向け。一般的には map+RWMutex |

---

## やってみよう！

### 問題 1: スレッドセーフなキャッシュ

`sync.RWMutex` を使って、TTL（有効期限）付きのスレッドセーフなキャッシュを実装してください。`Get(key)` で期限切れエントリは自動削除し、`Set(key, value, ttl)` で登録。

:::details 解答例を見る

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// entry はキャッシュの1エントリ。値と有効期限を保持する
type entry struct {
	value     string
	expiresAt time.Time
}

// TTLCache はTTL付きスレッドセーフキャッシュ
type TTLCache struct {
	mu    sync.RWMutex
	items map[string]entry
}

func NewTTLCache() *TTLCache {
	return &TTLCache{items: make(map[string]entry)}
}

// Set はキーに値とTTLを設定する
func (c *TTLCache) Set(key, value string, ttl time.Duration) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.items[key] = entry{
		value:     value,
		expiresAt: time.Now().Add(ttl),
	}
}

// Get はキーの値を取得する。期限切れの場合は削除してfalseを返す
func (c *TTLCache) Get(key string) (string, bool) {
	// まず読み取りロックで確認
	c.mu.RLock()
	e, ok := c.items[key]
	c.mu.RUnlock()

	if !ok {
		return "", false
	}

	// 期限切れチェック
	if time.Now().After(e.expiresAt) {
		// 書き込みロックに昇格して削除
		c.mu.Lock()
		defer c.mu.Unlock()
		// ダブルチェック: RUnlock〜Lock間に他のgoroutineが削除した可能性
		if e2, ok := c.items[key]; ok && time.Now().After(e2.expiresAt) {
			delete(c.items, key)
		}
		return "", false
	}

	return e.value, true
}

func main() {
	cache := NewTTLCache()

	cache.Set("token", "abc123", 500*time.Millisecond)

	// 即座に取得 → ヒット
	if v, ok := cache.Get("token"); ok {
		fmt.Printf("Hit: %s\n", v)
	}

	// TTL経過後 → ミス（自動削除される）
	time.Sleep(600 * time.Millisecond)
	if _, ok := cache.Get("token"); !ok {
		fmt.Println("Miss: expired and removed")
	}

	// 出力:
	// Hit: abc123
	// Miss: expired and removed
}
```

**ポイント:** `RLock` で読み取り → 期限切れなら `Lock` に昇格して削除、という2段階ロックがRWMutexの典型的な使い方。RUnlockとLockの間に他のgoroutineが変更する可能性があるため、ダブルチェックパターンを使う。

:::

### 問題 2: 並行ダウンローダー

`sync.WaitGroup` と `sync.Mutex` を使って、URLリストから並行にダウンロードし、全結果を集約する関数を実装してください。エラーが発生したURLはスキップすること。

:::details 解答例を見る

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"sync"
)

// DownloadResult は各URLのダウンロード結果
type DownloadResult struct {
	URL  string
	Body string
	Err  error
}

// downloadAll はURLリストから並行にダウンロードし、全結果を返す
func downloadAll(urls []string) []DownloadResult {
	var (
		mu      sync.Mutex
		wg      sync.WaitGroup
		results []DownloadResult
	)

	for _, url := range urls {
		wg.Add(1) // 必ずgo文の前でAdd
		go func(u string) {
			defer wg.Done()

			resp, err := http.Get(u)
			if err != nil {
				// エラーでも結果に記録（スキップ扱い）
				mu.Lock()
				results = append(results, DownloadResult{URL: u, Err: err})
				mu.Unlock()
				return
			}
			defer resp.Body.Close()

			body, err := io.ReadAll(resp.Body)
			if err != nil {
				mu.Lock()
				results = append(results, DownloadResult{URL: u, Err: err})
				mu.Unlock()
				return
			}

			// 共有スライスへの追加はMutexで保護
			mu.Lock()
			results = append(results, DownloadResult{URL: u, Body: string(body)})
			mu.Unlock()
		}(url)
	}

	wg.Wait() // 全goroutineの完了を待つ
	return results
}

func main() {
	urls := []string{
		"https://httpbin.org/get",
		"https://httpbin.org/status/404",
		"https://invalid.example.com", // エラーになるURL
	}

	results := downloadAll(urls)
	for _, r := range results {
		if r.Err != nil {
			fmt.Printf("[SKIP] %s: %v\n", r.URL, r.Err)
		} else {
			fmt.Printf("[OK]   %s: %d bytes\n", r.URL, len(r.Body))
		}
	}

	// 出力例:
	// [OK]   https://httpbin.org/get: 283 bytes
	// [OK]   https://httpbin.org/status/404: 0 bytes
	// [SKIP] https://invalid.example.com: Get "...": dial tcp: ...
}
```

**ポイント:** 共有スライスへの `append` はスレッドセーフではないため、必ず `sync.Mutex` で保護する。`wg.Add(1)` を `go` 文の前に置くのは鉄則。エラーが起きたURLもスキップしつつ記録することで、呼び出し元がエラー情報を利用できる。

:::

### 問題 3: メトリクス収集器

`atomic` パッケージを使って、リクエスト数・エラー数・アクティブ接続数・レイテンシ（`atomic.Int64` でナノ秒）を記録する `Metrics` 構造体を実装してください。Mutexは使わないこと。

:::details 解答例を見る

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"time"
)

// Metrics はロックフリーのメトリクス収集器
// 全フィールドがatomic型なのでMutex不要
type Metrics struct {
	RequestCount atomic.Int64
	ErrorCount   atomic.Int64
	ActiveConns  atomic.Int32
	TotalLatency atomic.Int64 // ナノ秒の累計
}

// RecordRequest はリクエストを記録する
func (m *Metrics) RecordRequest(start time.Time, success bool) {
	m.RequestCount.Add(1)
	if !success {
		m.ErrorCount.Add(1)
	}
	// レイテンシをナノ秒で累計加算
	latency := time.Since(start).Nanoseconds()
	m.TotalLatency.Add(latency)
}

// ConnOpened はアクティブ接続数を+1
func (m *Metrics) ConnOpened() { m.ActiveConns.Add(1) }

// ConnClosed はアクティブ接続数を-1
func (m *Metrics) ConnClosed() { m.ActiveConns.Add(-1) }

// AvgLatency は平均レイテンシを返す
func (m *Metrics) AvgLatency() time.Duration {
	count := m.RequestCount.Load()
	if count == 0 {
		return 0
	}
	return time.Duration(m.TotalLatency.Load() / count)
}

// Snapshot はメトリクスの現在値を返す
func (m *Metrics) Snapshot() string {
	return fmt.Sprintf(
		"requests=%d errors=%d active_conns=%d avg_latency=%v",
		m.RequestCount.Load(),
		m.ErrorCount.Load(),
		m.ActiveConns.Load(),
		m.AvgLatency(),
	)
}

func main() {
	var metrics Metrics
	var wg sync.WaitGroup

	// 100個のリクエストを並行にシミュレート
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()

			metrics.ConnOpened()
			defer metrics.ConnClosed()

			start := time.Now()
			time.Sleep(time.Duration(id%10) * time.Millisecond) // 処理シミュレート
			success := id%7 != 0                                // 7の倍数はエラー扱い
			metrics.RecordRequest(start, success)
		}(i)
	}

	wg.Wait()
	fmt.Println(metrics.Snapshot())
	// 出力例: requests=100 errors=15 active_conns=0 avg_latency=4.2ms
}
```

**ポイント:** 単純なカウンタやフラグには `atomic` がMutexより高速。`atomic.Int64` / `atomic.Int32` / `atomic.Bool` はGo 1.19+で追加された型安全なAPIで、`Add` / `Load` / `Store` のメソッドで操作する。平均レイテンシの計算は累計÷回数で近似できる。

:::

