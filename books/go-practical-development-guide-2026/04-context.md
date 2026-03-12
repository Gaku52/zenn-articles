---
title: "Context"
---

# Chapter 04: Context

> キャンセル、タイムアウト、リクエストスコープの値 — goroutineの制御

## contextとは

`context.Context`は、goroutine間で**キャンセルシグナル**・**タイムアウト**・**リクエストスコープの値**を伝搬する標準インターフェースです。

HTTPリクエストを受けてDBに問い合わせ、外部APIも呼ぶ — そんな処理で「クライアントが切断した」ときに全goroutineを一斉停止できるのがcontextの仕事です。contextは**キャンセル伝搬**・**デッドライン管理**・**値の伝搬**・**Done()チャネル**の4つの機能を提供します。

### 5つの設計原則

1. **第一引数に渡す** — `func DoSomething(ctx context.Context, ...)`
2. **構造体に保存しない** — リクエストごとに異なるcontextを使えなくなる
3. **nilを渡さない** — 不明な場合は`context.TODO()`を使う
4. **値は横断的関心事のみ** — ビジネスロジックのパラメータを入れない
5. **cancel関数は必ず呼ぶ** — `defer cancel()`を取得直後に書く

---

## WithCancel / WithTimeout / WithDeadline

### WithCancel — 手動キャンセル

`cancel()`を呼ぶと、そのcontextから派生した全ての子contextもキャンセルされます。

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

func worker(ctx context.Context, id int, wg *sync.WaitGroup) {
	defer wg.Done()
	for {
		select {
		case <-ctx.Done():
			fmt.Printf("Worker %d: stopped (%v)\n", id, ctx.Err())
			return
		case <-time.After(200 * time.Millisecond):
			fmt.Printf("Worker %d: processing\n", id)
		}
	}
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())

	var wg sync.WaitGroup
	for i := 1; i <= 3; i++ {
		wg.Add(1)
		go worker(ctx, i, &wg)
	}

	time.Sleep(1 * time.Second)
	fmt.Println("Cancelling all workers...")
	cancel() // 全goroutineに一斉通知

	wg.Wait()
	fmt.Println("All workers stopped")
}
```

1つの`cancel()`で全ワーカーが停止します。`select`で`ctx.Done()`を監視するのがgoroutine停止の基本パターンです。

### WithTimeout — 時間制限付き処理

指定時間後に自動キャンセルされるcontextを生成します。

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"time"
)

func fetchWithTimeout(url string, timeout time.Duration) ([]byte, error) {
	ctx, cancel := context.WithTimeout(context.Background(), timeout)
	defer cancel() // タイムアウト前に完了しても必ず呼ぶ

	req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
	if err != nil {
		return nil, fmt.Errorf("create request: %w", err)
	}

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return nil, fmt.Errorf("do request: %w", err)
	}
	defer resp.Body.Close()

	return io.ReadAll(resp.Body)
}

func main() {
	// 成功するケース
	body, err := fetchWithTimeout("https://httpbin.org/delay/1", 5*time.Second)
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}
	fmt.Printf("Response: %d bytes\n", len(body))

	// タイムアウトするケース
	_, err = fetchWithTimeout("https://httpbin.org/delay/10", 2*time.Second)
	fmt.Printf("Expected timeout: %v\n", err) // context deadline exceeded
}
```

### WithDeadline とネストの挙動

`WithDeadline`は絶対時刻で指定する版です（`WithTimeout`は内部的にそのラッパー）。ネストしたcontextでは**最も短いタイムアウトが勝ちます** — 子に親より長い値を設定しても親のタイムアウトが優先されます。

### WithCancelCause (Go 1.20+)

キャンセル理由を付与できます。デバッグ時に「なぜキャンセルされたか」がわかります。

```go
ctx, cancel := context.WithCancelCause(context.Background())
cancel(errors.New("resource limit exceeded"))

fmt.Println(ctx.Err())           // context canceled
fmt.Println(context.Cause(ctx))  // resource limit exceeded
```

---

## WithValue — 値の伝搬

`WithValue`は**横断的関心事**（トレースID、認証情報、ロケール等）をcontextに格納します。

入れてよいもの: トレースID、リクエストID、認証トークン、ロケール、テナントID。入れてはいけないもの: ユーザーID、検索条件、ビジネスロジックのパラメータ、DB接続 — これらは関数の引数で渡します。

### 型安全なアクセサパターン

文字列キーはパッケージ間で衝突します。**独自の非公開型をキーにする**のが鉄則です。

```go
package ctxutil

import "context"

// 非公開型でキーの衝突を防ぐ
type contextKey int

const (
	requestIDKey contextKey = iota
	traceIDKey
)

func SetRequestID(ctx context.Context, id string) context.Context {
	return context.WithValue(ctx, requestIDKey, id)
}

func GetRequestID(ctx context.Context) (string, bool) {
	id, ok := ctx.Value(requestIDKey).(string)
	return id, ok
}

func SetTraceID(ctx context.Context, id string) context.Context {
	return context.WithValue(ctx, traceIDKey, id)
}

func GetTraceID(ctx context.Context) string {
	id, _ := ctx.Value(traceIDKey).(string)
	return id // 空文字列がデフォルト
}
```

HTTPミドルウェアでこのアクセサを使えば、ハンドラ以降で型安全にcontextの値を取得できます。

---

## ベストプラクティスとアンチパターン

### アンチパターン: 構造体にcontextを保存する

```go
// BAD: リクエスト毎に異なるcontextを使えない
type Service struct {
	ctx context.Context
	db  *sql.DB
}

// GOOD: メソッドの第1引数で受け取る
type Service struct {
	db *sql.DB
}

func (s *Service) GetUser(ctx context.Context, id int) (*User, error) {
	return s.db.QueryRowContext(ctx, "SELECT ...", id).Scan(...)
}
```

### アンチパターン: cancel()を呼ばない

```go
// BAD: タイマーgoroutineがリークする
ctx, _ := context.WithTimeout(parentCtx, 5*time.Second)

// GOOD: 取得直後にdefer cancel()
ctx, cancel := context.WithTimeout(parentCtx, 5*time.Second)
defer cancel()
```

---

## HTTPサーバーでのcontext実践

### リクエストからDBまでの伝搬チェーン

`r.Context()`を起点に、サービス層 -> リポジトリ層 -> DBへとcontextが伝搬します。

```go
// Handler層: r.Context()を起点にタイムアウトを追加
func handleGetUser(svc *UserService) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		ctx, cancel := context.WithTimeout(r.Context(), 10*time.Second)
		defer cancel()

		user, err := svc.GetUser(ctx, r.PathValue("id"))
		if err != nil {
			if ctx.Err() != nil {
				// クライアント切断 or タイムアウト → ログだけ残す
				log.Printf("Request cancelled: %v", ctx.Err())
				return
			}
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		json.NewEncoder(w).Encode(user)
	}
}

// Service層: contextをそのまま伝搬
func (s *UserService) GetUser(ctx context.Context, id string) (*User, error) {
	return s.repo.FindByID(ctx, id)
}

// Repository層: contextをDBクエリに渡す
func (r *UserRepository) FindByID(ctx context.Context, id string) (*User, error) {
	var user User
	err := r.db.QueryRowContext(ctx,
		"SELECT id, name, email FROM users WHERE id = $1", id,
	).Scan(&user.ID, &user.Name, &user.Email)
	return &user, err
}
```

### Go 1.21+ の新機能

**`context.WithoutCancel`** — 親のキャンセルを伝搬しないcontextを作ります。監査ログやメトリクス送信など、リクエスト終了後も完了させたい処理に使います。

```go
func handleRequest(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	result := doMainWork(ctx)

	// リクエスト終了後も監査ログは書き切りたい
	bgCtx := context.WithoutCancel(ctx)
	bgCtx, bgCancel := context.WithTimeout(bgCtx, 30*time.Second)
	go func() {
		defer bgCancel()
		writeAuditLog(bgCtx, result) // r.Context()が切れても影響しない
	}()

	json.NewEncoder(w).Encode(result)
}
```

**`context.AfterFunc`** — キャンセル時にコールバックを実行します。

```go
ctx, cancel := context.WithCancel(context.Background())

// キャンセル時にリソースを自動解放
stop := context.AfterFunc(ctx, func() {
	db.Close()
	cache.Close()
})
defer stop() // 不要になったら登録解除も可能

// ... 処理 ...
cancel() // → AfterFuncに登録した関数が実行される
```

### Contextツリーの全体像

```
context.Background()
    │
    ├── WithCancel ──────────── HTTP Handler
    │       │
    │       ├── WithTimeout(10s) ── DB Query
    │       │
    │       ├── WithTimeout(5s) ─── External API
    │       │
    │       └── WithoutCancel ───── Audit Log (リクエスト後も継続)
    │
    └── WithCancel ──────────── Background Worker
            │
            └── WithValue(traceID) ── Logger

キャンセルの伝搬: 親がキャンセル → 全ての子もキャンセル
                 (WithoutCancelの子は除く)
```

---

## まとめ

| 概念 | 要点 |
|------|------|
| context.Background | アプリケーションのルートcontextとして使う。main関数やリクエストの起点で生成する |
| context.TODO | 適切なcontextが不明な場合の仮置き。後でBackground/親contextに置き換える |
| WithCancel | 手動キャンセル用。`cancel()`呼び出しで派生した全子contextも一斉停止する |
| WithTimeout / WithDeadline | 時間制限付きcontext。ネスト時は最も短いタイムアウトが優先される |
| WithCancelCause (Go 1.20+) | キャンセル理由を付与でき、`context.Cause(ctx)`でデバッグ時に原因を特定できる |
| WithValue | トレースID・認証情報など横断的関心事のみ格納。独自の非公開型をキーにして衝突を防ぐ |
| WithoutCancel (Go 1.21+) | 親のキャンセルを伝搬しないcontext。監査ログなどリクエスト後も完了させたい処理に使う |
| 5つの設計原則 | 第一引数に渡す・構造体に保存しない・nilを渡さない・値は横断的関心事のみ・cancel()は必ず呼ぶ |

---

## やってみよう！

### 練習1: タイムアウト付きAPIクライアント

以下の仕様でHTTP GETクライアントを実装してください。

- 全体のタイムアウトは5秒
- 3つのURLに対して並行にリクエストを送る
- 1つでもタイムアウトしたら全リクエストをキャンセルする
- ヒント: `context.WithCancel`で親contextを作り、各goroutineで`http.NewRequestWithContext`を使う

### 練習2: WithValueを使ったリクエストID伝搬

以下を実装してください。

1. 独自のキー型と`SetRequestID`/`GetRequestID`アクセサ関数を定義
2. HTTPミドルウェアで`X-Request-ID`ヘッダーからリクエストIDを取得し、contextに設定
3. ハンドラでcontextからリクエストIDを取得してレスポンスに含める
4. ヘッダーが未設定の場合はUUIDを自動生成する

### 練習3: Graceful Shutdown

以下の仕様でHTTPサーバーを実装してください。

- `SIGINT`/`SIGTERM`を受信したらGraceful Shutdownを開始
- 処理中のリクエストの完了を**最大30秒**待つ
- バックグラウンドワーカー（1秒ごとにログ出力）も同時に停止する
- ヒント: `signal.Notify`でシグナルを受け、`server.Shutdown(ctx)`にタイムアウト付きcontextを渡す
