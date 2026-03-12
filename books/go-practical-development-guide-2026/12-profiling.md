---
title: "プロファイリング"
---

# Chapter 12: プロファイリング

> pprof, trace, ベンチマーク — パフォーマンスの計測と改善

パフォーマンス最適化の鉄則は「推測するな、計測せよ」です。Goには強力なプロファイリングツールが標準で組み込まれており、ボトルネックの特定から改善の検証までを一貫して行えます。

この章では、**ベンチマークテスト**で現状を数値化し、**pprof**でホットスポットを特定し、**trace**で実行時の振る舞いを可視化する一連のフローを学びます。

---

## 12.1 ベンチマークテスト

最適化の起点はベンチマークです。`testing.B` を使って関数のパフォーマンスを定量的に測定します。

```go
// 最適化前: 文字列の + 連結
func ConcatStrings(strs []string) string {
    result := ""
    for _, s := range strs {
        result += s // 毎回新しい文字列を確保
    }
    return result
}

// 最適化後: strings.Builder + Grow
func ConcatStringsOpt(strs []string) string {
    var b strings.Builder
    size := 0
    for _, s := range strs {
        size += len(s)
    }
    b.Grow(size) // 事前にキャパシティ確保
    for _, s := range strs {
        b.WriteString(s)
    }
    return b.String()
}

func BenchmarkConcat(b *testing.B) {
    strs := make([]string, 1000)
    for i := range strs { strs[i] = "hello" }
    b.ResetTimer()
    b.ReportAllocs() // アロケーション情報を出力
    for b.Loop() { ConcatStrings(strs) }
}
// 想定出力例:
// BenchmarkConcat      500   2145678 ns/op  5308416 B/op  999 allocs/op
// BenchmarkConcatOpt  50000    28456 ns/op     5120 B/op    1 allocs/op
//                           （数十倍の高速化とアロケーション大幅削減が期待できる）
```

ベンチマークからプロファイルを取得するコマンドも覚えておきましょう。

```bash
go test -bench=BenchmarkConcat -cpuprofile=cpu.prof -count=5  # CPU
go test -bench=BenchmarkConcat -memprofile=mem.prof -count=5   # メモリ

# benchstat で最適化前後を統計的に比較
go install golang.org/x/perf/cmd/benchstat@latest
go test -bench=. -count=10 -benchmem > before.txt
# (最適化を適用後)
go test -bench=. -count=10 -benchmem > after.txt
benchstat before.txt after.txt
```

---

## 12.2 pprof — CPU・メモリプロファイリング

### CPUプロファイル

`go tool pprof` でCPUプロファイルを分析します。

```bash
# プロファイルの分析（インタラクティブモード）
go tool pprof cpu.prof

(pprof) top10          # CPU消費の多い関数トップ10
(pprof) list funcName  # 該当関数のソースコード付きコスト表示
(pprof) web            # コールグラフをブラウザで表示

# Web UIで分析（フレームグラフ付き）
go tool pprof -http=:8081 cpu.prof

# 2つのプロファイルの差分を比較
go tool pprof -diff_base=before.prof after.prof
```

**flat vs cum** の違いを理解するのが重要です。

- **flat**: その関数自身の処理時間
- **cum**: その関数 + 呼び出し先すべての合計時間
- flat と cum の差が大きい場合、ボトルネックは下流にある

### メモリプロファイル

```bash
# 現在使用中のメモリ（リーク検出向き）
go tool pprof -inuse_space mem.prof

# 累積アロケーション量（ホットパス特定向き）
go tool pprof -alloc_space mem.prof

# Web UIでフレームグラフ表示
go tool pprof -http=:8081 mem.prof
```

| モード | 用途 |
|--------|------|
| `-inuse_space` | メモリリーク検出（現在使用中のメモリ量） |
| `-inuse_objects` | GC圧力の調査（現在使用中のオブジェクト数） |
| `-alloc_space` | ホットパスの特定（累積アロケーション量） |
| `-alloc_objects` | アロケーション多発箇所の特定（累積回数） |

### CLIプログラムでのプロファイル取得

HTTPサーバーではなく短命なCLIプログラムの場合、`runtime/pprof` を直接使います。

```go
var cpuprofile = flag.String("cpuprofile", "", "CPUプロファイルの出力先")

func main() {
    flag.Parse()
    if *cpuprofile != "" {
        f, err := os.Create(*cpuprofile)
        if err != nil { log.Fatal(err) }
        defer f.Close()
        pprof.StartCPUProfile(f)
        defer pprof.StopCPUProfile()
    }
    doHeavyWork() // プロファイル対象の処理
}
```

```bash
go run main.go -cpuprofile=cpu.prof && go tool pprof -http=:8081 cpu.prof
```

---

## 12.3 net/http/pprof — 本番サーバーのプロファイリング

HTTPサーバーには `net/http/pprof` を使います。副作用インポートだけでエンドポイントが登録されます。

```go
package main

import (
    "log"
    "net/http"
    _ "net/http/pprof" // 副作用インポートでエンドポイント登録
)

func main() {
    // pprof は DefaultServeMux に登録されるため
    // 本番では別ポートで pprof 専用サーバーを起動する
    go func() {
        log.Println("pprof server: http://localhost:6060/debug/pprof/")
        log.Fatal(http.ListenAndServe("127.0.0.1:6060", nil))
    }()

    // アプリケーションサーバー
    mux := http.NewServeMux()
    mux.HandleFunc("/api/users", handleUsers)
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

主要なエンドポイント:

| エンドポイント | 内容 |
|--------------|------|
| `/debug/pprof/profile?seconds=30` | CPUプロファイル（30秒間） |
| `/debug/pprof/heap` | ヒープメモリプロファイル |
| `/debug/pprof/allocs` | メモリアロケーション累積 |
| `/debug/pprof/goroutine` | goroutine スタックトレース |
| `/debug/pprof/mutex` | ミューテックス競合プロファイル |
| `/debug/pprof/trace?seconds=5` | 実行トレース（5秒間） |

```bash
# 本番サーバーからCPUプロファイルを取得して分析
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# ヒーププロファイルをWeb UIで分析
go tool pprof -http=:8081 http://localhost:6060/debug/pprof/heap
```

**セキュリティ注意**: pprofは必ず `127.0.0.1` や内部ネットワークのみでリッスンしてください。公開ポートに露出させると、プロファイルデータからアプリケーションの内部構造が漏洩します。

---

## 12.4 runtime/trace — 実行トレース

pprofが「**何が**遅いか」を教えてくれるのに対し、traceは「**なぜ**遅いか」を教えてくれます。goroutineのスケジューリング、GCのタイミング、ネットワーク待ちなど、時系列のイベントを可視化できます。

```go
package main

import (
    "context"
    "os"
    "runtime/trace"
)

func main() {
    f, err := os.Create("trace.out")
    if err != nil {
        panic(err)
    }
    defer f.Close()

    if err := trace.Start(f); err != nil {
        panic(err)
    }
    defer trace.Stop()

    // タスクとリージョンで構造化
    ctx, task := trace.NewTask(context.Background(), "processOrder")
    defer task.End()

    trace.WithRegion(ctx, "validate", func() {
        validateOrder(ctx)
    })
    trace.WithRegion(ctx, "payment", func() {
        processPayment(ctx)
    })
    trace.Log(ctx, "status", "completed")
}
```

```bash
# トレースをブラウザで可視化
go tool trace trace.out

# ベンチマークからトレースを取得
go test -bench=BenchmarkSerialize -trace=trace.out
```

| 比較項目 | pprof | trace |
|---------|-------|-------|
| 目的 | CPU/メモリのホットスポット特定 | 時系列のイベント分析 |
| 粒度 | 関数レベルの統計 | goroutineレベルのイベント |
| オーバーヘッド | 低（サンプリング） | 高（全イベント記録） |
| 取得時間 | 30秒〜数分 | **数秒〜10秒**推奨 |
| GC/スケジューラ分析 | 不可 | 可能 |

> **注意**: トレースは短時間（1〜5秒）に限定してください。長時間取得するとデータが膨大になり、UIがフリーズします。

---

## 12.5 最適化のベストプラクティス

### 最適化サイクル

```
1. 計測 → ベンチマークで現状を数値化
2. プロファイル → pprof でホットスポットを特定
3. 分析 → フレームグラフ・コールグラフで原因を理解
4. 最適化 → ボトルネック箇所のみを改善
5. 検証 → ベンチマークで効果を定量的に確認
6. 繰り返す
```

### よくある最適化パターン

**スライスのプリアロケーション** — サイズが分かっている場合は `make` でキャパシティを指定:

```go
// NG: append のたびに底層配列が再割り当て
var result []int
for i := 0; i < n; i++ {
    result = append(result, i*2)
}

// OK: 事前にキャパシティを確保
result := make([]int, 0, n)
for i := 0; i < n; i++ {
    result = append(result, i*2)
}
```

**マップのプリアロケーション** — `make(map[K]V, n)` でrehashを回避。

**Escape Analysis の確認** — ヒープ割り当てを減らすにはまず現状を知ること:

```bash
go build -gcflags="-m" ./...
# ./main.go:20:10: &User{...} escapes to heap
```

### やってはいけないこと

1. **プロファイリングなしの推測的最適化** — 「ここが遅いはず」は大抵外れます。必ずプロファイルに基づいて最適化してください
2. **sync.Pool の過剰利用** — 小さなオブジェクト（`int` 等）にPoolを使うとオーバーヘッドのほうが大きくなります。また `Pool.Get()` 後の `Reset()` 忘れは深刻なバグの原因です

---

## まとめ

| 概念 | 要点 |
|------|------|
| ベンチマーク | `testing.B` と `b.ReportAllocs()` で関数のパフォーマンスとアロケーションを定量的に測定する |
| benchstat | `-count=10` で複数回計測し、最適化前後の差を統計的に比較する |
| pprof (CPU) | `go tool pprof -http=:8081 cpu.prof` でフレームグラフを表示し、flat/cumの差からボトルネックの所在を特定する |
| pprof (メモリ) | `-inuse_space` でリーク検出、`-alloc_space` でホットパス特定と目的に応じてモードを使い分ける |
| net/http/pprof | 副作用インポートだけで本番サーバーにプロファイルエンドポイントを追加できる。`127.0.0.1` 限定で公開すること |
| runtime/trace | goroutineスケジューリング・GC・ネットワーク待ちを時系列で可視化し「なぜ遅いか」を分析する。取得は数秒に限定 |
| 最適化サイクル | 計測 → プロファイル → 分析 → 最適化 → 検証を繰り返す。推測ではなく計測に基づいて改善する |
| プリアロケーション | スライスやマップのキャパシティを事前確保し、再割り当てやrehashによるアロケーションを削減する |

---

## やってみよう！

### 演習1: ベンチマーク＆プロファイルのサイクル

以下の関数のベンチマークを書き、CPUプロファイルを取得してボトルネックを特定し、最適化してください。

```go
func CountWords(texts []string) map[string]int {
    counts := make(map[string]int)
    for _, text := range texts {
        words := strings.Split(text, " ")
        for _, w := range words {
            counts[strings.ToLower(w)]++
        }
    }
    return counts
}
```

**ヒント**: `b.ReportAllocs()` でアロケーション数を確認し、`go test -bench=. -cpuprofile=cpu.prof` でプロファイルを取得して `go tool pprof -http=:8081 cpu.prof` のフレームグラフを観察しましょう。

### 演習2: goroutineリークの検出

意図的にgoroutineリークを起こすプログラムを書き、`/debug/pprof/goroutine?debug=1` で増加を観察してください。その後、`context.WithCancel` を使ってリークを解消してください。

### 演習3: traceでGCを観察する

大量のアロケーションを発生させるプログラムを `runtime/trace` 付きで実行し、`go tool trace` のタイムラインでGCイベントの頻度とSTW（Stop-The-World）時間を観察してください。`GOGC` の値を変えるとGCの挙動がどう変わるか試してみましょう。
