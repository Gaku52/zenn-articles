---
title: "net/http"
---

# Chapter 05: net/http

> 標準ライブラリだけで作るWebサーバー

## Handler の基本

Go のHTTPサーバーは `http.Handler` インターフェースを中心に設計されています。

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

`HandlerFunc` は関数を `Handler` に変換するアダプタ型です。この仕組みにより、普通の関数をそのままハンドラとして登録できます。

最小限のサーバーは以下の通りです。本番では `&http.Server{}` でタイムアウトを明示的に設定するのが必須です。

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /hello", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, World!")
})

server := &http.Server{
    Addr:         ":8080",
    Handler:      mux,
    ReadTimeout:  10 * time.Second,
    WriteTimeout: 10 * time.Second,
    IdleTimeout:  120 * time.Second,
}
log.Fatal(server.ListenAndServe())
```

---

## ServeMux（Go 1.22+）

Go 1.22 で `ServeMux` が大幅に強化されました。HTTPメソッド指定・パスパラメータ・ワイルドカードが標準サポートされ、多くのケースでサードパーティルーターが不要です。

```
パターン: "GET /users/{id}"
            │       │    │
            │       │    └── パスパラメータ: r.PathValue("id")
            │       └────── パスプレフィックス
            └────────────── HTTPメソッド制約

優先順位: "GET /users/me" > "GET /users/{id}" > "GET /users/{path...}"
```

### REST API の実装例

```go
package main

import (
	"encoding/json"
	"net/http"
	"strconv"
)

type User struct {
	ID    int    `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email"`
}

var users = map[int]*User{
	1: {ID: 1, Name: "Tanaka", Email: "tanaka@example.com"},
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /users", listUsers)
	mux.HandleFunc("POST /users", createUser)
	mux.HandleFunc("GET /users/{id}", getUser)
	mux.HandleFunc("DELETE /users/{id}", deleteUser)
	http.ListenAndServe(":8080", mux)
}

func listUsers(w http.ResponseWriter, r *http.Request) {
	list := make([]*User, 0, len(users))
	for _, u := range users {
		list = append(list, u)
	}
	writeJSON(w, http.StatusOK, list)
}

func getUser(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.Atoi(r.PathValue("id")) // Go 1.22+
	if err != nil {
		writeError(w, http.StatusBadRequest, "invalid user id")
		return
	}
	user, ok := users[id]
	if !ok {
		writeError(w, http.StatusNotFound, "user not found")
		return
	}
	writeJSON(w, http.StatusOK, user)
}

func createUser(w http.ResponseWriter, r *http.Request) {
	var user User
	if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
		writeError(w, http.StatusBadRequest, "invalid json")
		return
	}
	user.ID = len(users) + 1
	users[user.ID] = &user
	writeJSON(w, http.StatusCreated, user)
}

func deleteUser(w http.ResponseWriter, r *http.Request) {
	id, _ := strconv.Atoi(r.PathValue("id"))
	delete(users, id)
	w.WriteHeader(http.StatusNoContent)
}

func writeJSON(w http.ResponseWriter, status int, data any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(data)
}

func writeError(w http.ResponseWriter, status int, msg string) {
	writeJSON(w, status, map[string]string{"error": msg})
}
```

`r.PathValue("id")` でパスパラメータを取得するのが Go 1.22+ のポイントです。以前は `gorilla/mux` や `chi` が必要だった機能が標準で使えます。

---

## ミドルウェア

ミドルウェアは `func(http.Handler) http.Handler` シグネチャの関数です。リクエスト処理の前後に横断的関心事（ログ、認証、CORS等）を挿入します。

```
リクエスト → [Recovery] → [Logging] → [Auth] → [Handler]
レスポンス ← [Recovery] ← [Logging] ← [Auth] ←
```

### 実装と連結

```go
// ログ記録: ResponseWriterをラップしてステータスコードをキャプチャ
func loggingMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		rec := &statusRecorder{ResponseWriter: w, status: 200}
		next.ServeHTTP(rec, r)
		log.Printf("%s %s %d %v", r.Method, r.URL.Path, rec.status, time.Since(start))
	})
}

type statusRecorder struct {
	http.ResponseWriter
	status int
}

func (r *statusRecorder) WriteHeader(code int) {
	r.status = code
	r.ResponseWriter.WriteHeader(code)
}

// パニックリカバリ
func recoveryMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if err := recover(); err != nil {
				log.Printf("PANIC: %v", err)
				http.Error(w, "internal server error", 500)
			}
		}()
		next.ServeHTTP(w, r)
	})
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /api/data", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`{"data":"hello"}`))
	})

	// ミドルウェアチェーン（内側から外側に適用）
	handler := recoveryMiddleware(loggingMiddleware(mux))
	http.ListenAndServe(":8080", handler)
}
```

「ResponseWriterラッパー」でステータスコードを記録するパターンはミドルウェア実装の定石です。

---

## HTTPクライアント

外部APIを呼ぶ側も `net/http` で実装します。3つの鉄則があります。

1. **`http.DefaultClient` を使わない** -- タイムアウトが未設定
2. **`context` を必ず渡す** -- `NewRequestWithContext` でキャンセル伝搬
3. **`io.LimitReader` でボディサイズを制限** -- メモリ枯渇対策

```go
type APIClient struct {
	client  *http.Client
	baseURL string
}

func NewAPIClient(baseURL string) *APIClient {
	return &APIClient{
		client: &http.Client{
			Timeout: 30 * time.Second,
			Transport: &http.Transport{
				MaxIdleConns:        100,
				MaxIdleConnsPerHost: 10,
				IdleConnTimeout:     90 * time.Second,
			},
		},
		baseURL: baseURL,
	}
}

func (c *APIClient) Get(ctx context.Context, path string, result any) error {
	req, err := http.NewRequestWithContext(ctx, "GET", c.baseURL+path, nil)
	if err != nil {
		return fmt.Errorf("create request: %w", err)
	}
	req.Header.Set("Accept", "application/json")

	resp, err := c.client.Do(req)
	if err != nil {
		return fmt.Errorf("do request: %w", err)
	}
	defer resp.Body.Close()

	body, err := io.ReadAll(io.LimitReader(resp.Body, 10<<20)) // 10MB制限
	if err != nil {
		return fmt.Errorf("read body: %w", err)
	}
	if resp.StatusCode >= 400 {
		return fmt.Errorf("HTTP %d: %s", resp.StatusCode, body)
	}
	return json.Unmarshal(body, result)
}

// 使い方
func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	client := NewAPIClient("https://httpbin.org")
	var result map[string]any
	client.Get(ctx, "/get", &result)
}
```

---

## Graceful Shutdown

本番サーバーでは、シグナルを受けたら処理中のリクエストを完了してから停止します。

```go
server := &http.Server{Addr: ":8080", Handler: mux}

go func() {
    log.Fatal(server.ListenAndServe())
}()

// SIGINT/SIGTERM を待機
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

// 処理中リクエストの完了を最大30秒待つ
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
if err := server.Shutdown(ctx); err != nil {
    log.Printf("Shutdown error: %v", err)
}
```

`server.Shutdown(ctx)` は新規接続を拒否しつつ、処理中のリクエスト完了を待ちます。Kubernetes環境ではヘルスチェックを503に切り替えてからShutdownする2段階停止が定石です。

---

## アンチパターン

### タイムアウト未設定

```go
// NG: Slowloris攻撃に脆弱
http.ListenAndServe(":8080", mux)

// OK: 必ずタイムアウトを設定
server := &http.Server{
    Addr:         ":8080",
    Handler:      mux,
    ReadTimeout:  10 * time.Second,
    WriteTimeout: 10 * time.Second,
    IdleTimeout:  120 * time.Second,
}
```

### WriteHeader後のエラー返却

```go
// NG: WriteHeader後にhttp.Errorを呼んでも効かない
func badHandler(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    http.Error(w, "error", 500) // 無視される！
}

// OK: エラーチェックを先に、return で抜ける
func goodHandler(w http.ResponseWriter, r *http.Request) {
    data, err := process()
    if err != nil {
        http.Error(w, "error", 500)
        return
    }
    w.Write(data)
}
```

---

## まとめ

| 概念 | 要点 |
|------|------|
| Handler | `ServeHTTP(w, r)` を持つインターフェース |
| ServeMux (1.22+) | メソッド指定・パスパラメータ・ワイルドカードを標準サポート |
| ミドルウェア | `func(http.Handler) http.Handler` で横断的関心事を分離 |
| Server設定 | Read/Write/IdleTimeout は本番では必須 |
| HTTPクライアント | DefaultClient禁止、Timeout + Transport + Context |
| Graceful Shutdown | `server.Shutdown(ctx)` + シグナルハンドリング |

---

## やってみよう！

### 問題 1: TODOリストAPI

Go 1.22+ の構文で TODO 管理 REST API を作成。`GET /todos`、`POST /todos`、`PUT /todos/{id}`（完了切替）、`DELETE /todos/{id}` の4エンドポイント。データはメモリ上の `map` で管理し、`r.PathValue("id")` を使うこと。

### 問題 2: リクエストロガー

メソッド・パス・ステータスコード・処理時間を記録するロギングミドルウェアを実装。`ResponseWriter` ラッパーでステータスコードをキャプチャし、問題1のAPIに組み込んで動作確認。

### 問題 3: タイムアウト付き外部API呼び出し

`NewAPIClient` を参考に、`https://httpbin.org/delay/3` へ2秒タイムアウト付きGETを送信。タイムアウト時のエラーメッセージ表示を確認。`context.WithTimeout` を使うこと。
