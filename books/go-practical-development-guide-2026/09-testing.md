---
title: "テスト"
---

# Chapter 09: テスト

> httptest, テーブル駆動テスト, モック — 実践的なGoテスト

Goのテストは `_test.go` ファイルに `Test` プレフィックスの関数を書くだけで始められます。この章では実践で最も使う5つのテスト技法を身につけます。

## テーブル駆動テスト

Goで最も推奨されるパターン。テストケースをスライスで定義し、`t.Run` でループ実行する。

```go
func TestDivide(t *testing.T) {
    tests := []struct {
        name      string
        a, b      float64
        want      float64
        wantError bool
    }{
        {name: "正常な除算", a: 10, b: 2, want: 5},
        {name: "小数結果", a: 7, b: 3, want: 2.3333},
        {name: "ゼロ除算", a: 5, b: 0, wantError: true},
        {name: "負の数", a: -10, b: 2, want: -5},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Divide(tt.a, tt.b)
            if (err != nil) != tt.wantError {
                t.Fatalf("Divide(%v, %v) error = %v, wantError %v",
                    tt.a, tt.b, err, tt.wantError)
            }
            if !tt.wantError && math.Abs(got-tt.want) > 0.001 {
                t.Errorf("Divide(%v, %v) = %v, want %v",
                    tt.a, tt.b, got, tt.want)
            }
        })
    }
}
```

- **`t.Run` は必須** -- `go test -run TestDivide/ゼロ除算` で個別実行できる
- **`t.Parallel()`** をサブテスト内に追加すれば並列実行可能（Go 1.22+ではループ変数キャプチャ不要）
- 前提条件の検証には `t.Fatalf`（即中断）、独立した検証には `t.Errorf`（続行）

---

## httptest でHTTPハンドラをテスト

`net/http/httptest` はサーバーを起動せずにハンドラをテストできる標準パッケージ。

### NewRecorder -- ハンドラのユニットテスト

```go
func TestCreateUserHandler(t *testing.T) {
    tests := []struct {
        name       string
        body       string
        wantStatus int
        wantKey    string
        wantValue  string
    }{
        {"正常作成", `{"name":"Alice","email":"alice@example.com"}`,
            http.StatusCreated, "name", "Alice"},
        {"名前が空", `{"name":"","email":"alice@example.com"}`,
            http.StatusBadRequest, "error", "name is required"},
        {"不正なJSON", `{invalid`, http.StatusBadRequest, "", ""},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            req := httptest.NewRequest("POST", "/api/users",
                strings.NewReader(tt.body))
            req.Header.Set("Content-Type", "application/json")
            rec := httptest.NewRecorder()

            NewRouter().ServeHTTP(rec, req) // テスト対象のルーター

            if rec.Code != tt.wantStatus {
                t.Errorf("status = %d, want %d", rec.Code, tt.wantStatus)
            }
            if tt.wantKey != "" {
                var got map[string]string
                json.Unmarshal(rec.Body.Bytes(), &got)
                if got[tt.wantKey] != tt.wantValue {
                    t.Errorf("%s = %q, want %q",
                        tt.wantKey, got[tt.wantKey], tt.wantValue)
                }
            }
        })
    }
}
```

### NewServer -- 外部APIのモック

外部APIを呼ぶクライアントのテストには `httptest.NewServer` でモックサーバーを立てる。

```go
func TestFetchUserFromAPI(t *testing.T) {
    mock := httptest.NewServer(http.HandlerFunc(
        func(w http.ResponseWriter, r *http.Request) {
            if r.Header.Get("Authorization") != "Bearer test-token" {
                w.WriteHeader(http.StatusUnauthorized)
                return
            }
            w.Header().Set("Content-Type", "application/json")
            json.NewEncoder(w).Encode(map[string]any{
                "id": 42, "name": "Alice",
            })
        },
    ))
    defer mock.Close()

    client := NewAPIClient(mock.URL, "test-token")
    user, err := client.FetchUser(42)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if user.Name != "Alice" {
        t.Errorf("name = %q, want Alice", user.Name)
    }
}
```

認証・CORS・レート制限などのミドルウェアも、テーブル駆動テスト + `httptest` の同じパターンで検証できる。

---

## モックとインターフェース活用

Goのモックは**インターフェースに依存する設計**が前提。テスト対象がインターフェースに依存していれば、テスト用の実装を差し替えられる。

```go
type EmailSender interface {
    Send(to, subject, body string) error
}

// Spy: 呼び出しを記録して検証に使う
type SpyEmailSender struct {
    Calls []struct{ To, Subject, Body string }
}
func (s *SpyEmailSender) Send(to, subject, body string) error {
    s.Calls = append(s.Calls, struct{ To, Subject, Body string }{to, subject, body})
    return nil
}

// Stub: 固定のエラーを返す
type StubEmailSender struct{ Err error }
func (s StubEmailSender) Send(to, subject, body string) error { return s.Err }

func TestNotificationService(t *testing.T) {
    t.Run("送信内容の検証", func(t *testing.T) {
        spy := &SpyEmailSender{}
        svc := NewNotificationService(spy)
        svc.NotifyUser("alice@example.com", "Welcome!")
        if len(spy.Calls) != 1 {
            t.Fatalf("calls = %d, want 1", len(spy.Calls))
        }
        if spy.Calls[0].To != "alice@example.com" {
            t.Errorf("to = %q, want alice@example.com", spy.Calls[0].To)
        }
    })

    t.Run("送信失敗時のエラー処理", func(t *testing.T) {
        stub := StubEmailSender{Err: errors.New("SMTP error")}
        svc := NewNotificationService(stub)
        if err := svc.NotifyUser("alice@example.com", "Welcome!"); err == nil {
            t.Error("expected error, got nil")
        }
    })
}
```

テストダブルの使い分け:

| 種類 | 用途 | コスト |
|------|------|--------|
| **Stub** | 固定値を返す | 最低 |
| **Spy** | 呼び出しを記録 | 低 |
| **Fake** | 簡易的な動作実装（インメモリDBなど） | 中 |
| **Mock** | 呼び出し順序・回数まで検証 | 高 |

まずは手書きのStub/Spyで十分なことが多い。`testify/mock` はモック生成を自動化したい場合に導入する。

### FakeClockパターン -- 時刻のテスト

`time.Now()` を直接呼ぶとテストが書きにくい。インターフェースで抽象化する。

```go
type Clock interface{ Now() time.Time }
type RealClock struct{}
func (RealClock) Now() time.Time { return time.Now() }

type FakeClock struct{ Current time.Time }
func (c *FakeClock) Now() time.Time     { return c.Current }
func (c *FakeClock) Advance(d time.Duration) { c.Current = c.Current.Add(d) }
```

本番では `RealClock`、テストでは `FakeClock` を注入する。

---

## ベンチマーク

`Benchmark` プレフィックスの関数で性能を計測する。`b.N` はフレームワークが自動調整する。

```go
func BenchmarkStringConcat(b *testing.B) {
    b.Run("Plus演算子", func(b *testing.B) {
        b.ReportAllocs()
        for b.Loop() {
            s := ""
            for j := 0; j < 100; j++ { s += "a" }
        }
    })
    b.Run("strings.Builder", func(b *testing.B) {
        b.ReportAllocs()
        for b.Loop() {
            var sb strings.Builder
            for j := 0; j < 100; j++ { sb.WriteString("a") }
            _ = sb.String()
        }
    })
}
```

```bash
go test -bench=BenchmarkStringConcat -benchmem ./...
# Plus演算子-8       50000   5200 ns/op  5248 B/op  99 allocs/op
# strings.Builder-8  500000   380 ns/op   512 B/op   8 allocs/op
```

`b.ReportAllocs()` でメモリアロケーションを計測、`benchstat` で変更前後の統計比較ができる。

---

## テストヘルパーと実行Tips

### t.Helper() と t.Cleanup()

テスト用ユーティリティには必ず `t.Helper()` をつける（失敗時に呼び出し元の行番号を表示）。リソース解放には `t.Cleanup()` を使う（ヘルパー内の `defer` と違い、テスト全体の終了時に実行される）。

```go
func setupTestDB(t *testing.T) *sql.DB {
    t.Helper()
    db, err := sql.Open("sqlite3", ":memory:")
    if err != nil {
        t.Fatalf("failed to open db: %v", err)
    }
    t.Cleanup(func() { db.Close() })
    return db
}

func newJSONRequest(t *testing.T, method, url string, body any) *http.Request {
    t.Helper()
    b, err := json.Marshal(body)
    if err != nil {
        t.Fatalf("failed to marshal: %v", err)
    }
    req := httptest.NewRequest(method, url, bytes.NewReader(b))
    req.Header.Set("Content-Type", "application/json")
    return req
}
```

### よく使うフラグ

```bash
go test ./...                      # 全パッケージ
go test -v -run TestDivide ./...   # 特定テスト（詳細出力）
go test -race ./...                # データレース検出
go test -cover ./...               # カバレッジ表示
go test -coverprofile=c.out ./...  # プロファイル出力
go tool cover -html=c.out          # HTMLレポート
go test -short ./...               # 重いテストをスキップ
go test -count=1 ./...             # キャッシュ無効化
go test -tags=integration ./...    # 統合テスト実行
```

### 統合テストの分離

ビルドタグで通常テストから分離する。`go test ./...` では実行されない。

```go
//go:build integration

package store_test

func TestPostgresUserStore(t *testing.T) {
    dsn := os.Getenv("TEST_DATABASE_URL")
    if dsn == "" {
        t.Skip("TEST_DATABASE_URL が設定されていません")
    }
    // DB接続が必要なテスト...
}
```

---

## アンチパターン

**time.Sleep に頼る** -- 環境でフレーキーになる。チャネルで同期を取る:

```go
// NG
go StartProcess()
time.Sleep(2 * time.Second)

// OK
done := make(chan struct{})
go func() { StartProcess(); close(done) }()
select {
case <-done:
case <-time.After(5 * time.Second):
    t.Fatal("タイムアウト")
}
```

**テスト間で状態を共有する** -- 各テストは独立して実行可能でなければならない。グローバル変数で状態を渡すと実行順序に依存する壊れやすいテストになる。

---

## まとめ

| 概念 | 要点 |
|------|------|
| テーブル駆動テスト | ケースをスライスで定義し `t.Run` でループ実行。個別実行・並列化が容易 |
| httptest.NewRecorder | サーバーを起動せずにハンドラをユニットテスト。ステータスコードとボディを検証 |
| httptest.NewServer | 外部APIのモックサーバーを立てて、クライアント側のテストに利用 |
| インターフェースによるモック | Stub/Spy/Fake/Mock をインターフェース経由で注入し、依存を差し替える |
| FakeClockパターン | `time.Now()` をインターフェースで抽象化し、テスト時に時刻を制御 |
| ベンチマーク | `Benchmark` 関数 + `b.ReportAllocs()` で性能とアロケーションを計測 |
| t.Helper / t.Cleanup | ヘルパーには `t.Helper()` で行番号表示、リソース解放には `t.Cleanup()` |
| 統合テストの分離 | `//go:build integration` タグで通常テストから分離し、CI等で個別実行 |

---

## やってみよう！

### 演習1: テーブル駆動テストを書く

`IsPalindrome(s string) bool`（回文判定）に対してテーブル駆動テストを書いてください。ケース: 回文（"racecar"）、空文字列、1文字、非回文、大文字小文字混在（"RaceCar" -- 仕様は自分で決める）。

:::details 解答例を見る

```go
package main

import (
	"fmt"
	"strings"
	"testing"
)

// IsPalindrome は大文字小文字を区別せずに回文かどうかを判定する。
// 仕様: 大文字小文字は無視して比較する（"RaceCar" → true）。
func IsPalindrome(s string) bool {
	s = strings.ToLower(s) // 大文字小文字を統一して比較
	for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
		if s[i] != s[j] {
			return false
		}
	}
	return true
}

func TestIsPalindrome(t *testing.T) {
	tests := []struct {
		name  string
		input string
		want  bool
	}{
		{name: "回文", input: "racecar", want: true},
		{name: "空文字列", input: "", want: true},
		{name: "1文字", input: "a", want: true},
		{name: "非回文", input: "hello", want: false},
		{name: "大文字小文字混在の回文", input: "RaceCar", want: true},
		{name: "偶数長の回文", input: "abba", want: true},
		{name: "偶数長の非回文", input: "abcd", want: false},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := IsPalindrome(tt.input)
			if got != tt.want {
				t.Errorf("IsPalindrome(%q) = %v, want %v", tt.input, got, tt.want)
			}
		})
	}
}

func main() {
	// 動作確認用
	words := []string{"racecar", "", "a", "hello", "RaceCar", "abba"}
	for _, w := range words {
		fmt.Printf("IsPalindrome(%q) = %v\n", w, IsPalindrome(w))
	}
	// 出力例:
	// IsPalindrome("racecar") = true
	// IsPalindrome("") = true
	// IsPalindrome("a") = true
	// IsPalindrome("hello") = false
	// IsPalindrome("RaceCar") = true
	// IsPalindrome("abba") = true
}
```

ポイント: テーブル駆動テストでは「正常系・境界値・異常系」を網羅する。仕様が曖昧な部分（大文字小文字の扱い）はテストケースの段階で決定し、テスト名に仕様の意図を明示する。`t.Run` を使うことで `go test -run TestIsPalindrome/空文字列` のように個別実行できる。

:::

### 演習2: httptestでAPIテスト

`GET /api/health` ハンドラを実装し、`httptest.NewRecorder` でテストしてください。期待値: ステータス200、Content-Type: application/json、ボディ `{"status":"ok","version":"1.0.0"}`。

:::details 解答例を見る

```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"net/http/httptest"
	"testing"
)

// HealthResponse はヘルスチェックのレスポンス構造体
type HealthResponse struct {
	Status  string `json:"status"`
	Version string `json:"version"`
}

// HealthHandler はヘルスチェック用のHTTPハンドラ
func HealthHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(HealthResponse{
		Status:  "ok",
		Version: "1.0.0",
	})
}

func TestHealthHandler(t *testing.T) {
	tests := []struct {
		name           string
		method         string
		wantStatus     int
		wantContentType string
		wantBody       HealthResponse
	}{
		{
			name:           "正常なGETリクエスト",
			method:         "GET",
			wantStatus:     http.StatusOK,
			wantContentType: "application/json",
			wantBody:       HealthResponse{Status: "ok", Version: "1.0.0"},
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			// httptest.NewRequest でリクエストを作成
			req := httptest.NewRequest(tt.method, "/api/health", nil)
			// httptest.NewRecorder でレスポンスを記録
			rec := httptest.NewRecorder()

			HealthHandler(rec, req)

			// ステータスコードの検証
			if rec.Code != tt.wantStatus {
				t.Errorf("status = %d, want %d", rec.Code, tt.wantStatus)
			}

			// Content-Type の検証
			ct := rec.Header().Get("Content-Type")
			if ct != tt.wantContentType {
				t.Errorf("Content-Type = %q, want %q", ct, tt.wantContentType)
			}

			// ボディの検証（JSONデコードして構造体で比較）
			var got HealthResponse
			if err := json.Unmarshal(rec.Body.Bytes(), &got); err != nil {
				t.Fatalf("failed to decode body: %v", err)
			}
			if got != tt.wantBody {
				t.Errorf("body = %+v, want %+v", got, tt.wantBody)
			}
		})
	}
}

func main() {
	http.HandleFunc("/api/health", HealthHandler)
	fmt.Println("サーバー起動: http://localhost:8080")
	http.ListenAndServe(":8080", nil)
	// 出力例: curl http://localhost:8080/api/health
	// {"status":"ok","version":"1.0.0"}
}
```

ポイント: `httptest.NewRecorder` はサーバーを起動せずにハンドラを直接テストできる。レスポンスの検証は「ステータスコード → ヘッダー → ボディ」の順で行い、ボディはJSON構造体にデコードして比較すると可読性が高い。

:::

### 演習3: インターフェースでモック可能にする

以下のコードにインターフェースを導入し、Spyでテスト可能にリファクタリングしてください:

```go
func GetWeather(city string) (string, error) {
    resp, err := http.Get("https://api.weather.com/" + city)
    if err != nil { return "", err }
    defer resp.Body.Close()
    // ... レスポンスをパースして天気を返す
}
```

ヒント: `WeatherFetcher` インターフェースを定義し、`GetWeather` を構造体のメソッドに変更する。

:::details 解答例を見る

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"testing"
)

// WeatherFetcher は天気情報を取得するインターフェース。
// インターフェースで抽象化することで、テスト時に実装を差し替えられる。
type WeatherFetcher interface {
	FetchWeather(city string) (string, error)
}

// APIWeatherFetcher は実際のAPIを呼ぶ本番用の実装
type APIWeatherFetcher struct {
	BaseURL string
	Client  *http.Client
}

func (f *APIWeatherFetcher) FetchWeather(city string) (string, error) {
	resp, err := f.Client.Get(f.BaseURL + "/" + city)
	if err != nil {
		return "", fmt.Errorf("API呼び出し失敗: %w", err)
	}
	defer resp.Body.Close()

	body, err := io.ReadAll(resp.Body)
	if err != nil {
		return "", fmt.Errorf("レスポンス読み取り失敗: %w", err)
	}

	var result struct {
		Weather string `json:"weather"`
	}
	if err := json.Unmarshal(body, &result); err != nil {
		return "", fmt.Errorf("JSONパース失敗: %w", err)
	}
	return result.Weather, nil
}

// SpyWeatherFetcher はテスト用のSpy実装。
// 呼び出し記録を保持し、事前設定した値を返す。
type SpyWeatherFetcher struct {
	Calls   []string // 呼び出された都市名を記録
	Result  string   // 返す天気情報
	Err     error    // 返すエラー
}

func (s *SpyWeatherFetcher) FetchWeather(city string) (string, error) {
	s.Calls = append(s.Calls, city)
	return s.Result, s.Err
}

// WeatherService は WeatherFetcher に依存するサービス層
type WeatherService struct {
	fetcher WeatherFetcher
}

func NewWeatherService(f WeatherFetcher) *WeatherService {
	return &WeatherService{fetcher: f}
}

func (ws *WeatherService) GetWeather(city string) (string, error) {
	return ws.fetcher.FetchWeather(city)
}

func TestWeatherService(t *testing.T) {
	t.Run("正常に天気を取得", func(t *testing.T) {
		spy := &SpyWeatherFetcher{Result: "晴れ"}
		svc := NewWeatherService(spy)

		got, err := svc.GetWeather("tokyo")
		if err != nil {
			t.Fatalf("unexpected error: %v", err)
		}
		if got != "晴れ" {
			t.Errorf("weather = %q, want 晴れ", got)
		}
		// Spy で呼び出し引数を検証
		if len(spy.Calls) != 1 || spy.Calls[0] != "tokyo" {
			t.Errorf("calls = %v, want [tokyo]", spy.Calls)
		}
	})

	t.Run("API失敗時のエラー処理", func(t *testing.T) {
		spy := &SpyWeatherFetcher{Err: fmt.Errorf("connection refused")}
		svc := NewWeatherService(spy)

		_, err := svc.GetWeather("tokyo")
		if err == nil {
			t.Error("expected error, got nil")
		}
	})
}

func main() {
	// 本番ではAPIWeatherFetcherを注入
	fetcher := &APIWeatherFetcher{
		BaseURL: "https://api.weather.com",
		Client:  http.DefaultClient,
	}
	svc := NewWeatherService(fetcher)
	weather, err := svc.GetWeather("tokyo")
	if err != nil {
		fmt.Printf("エラー: %v\n", err)
		return
	}
	fmt.Printf("東京の天気: %s\n", weather)
	// 出力例: 東京の天気: 晴れ
}
```

ポイント: 外部API依存をインターフェースで抽象化するのがGoのテスト設計の基本。Spyは「何が呼ばれたか」を記録して検証に使う。本番実装とテスト実装を同じインターフェースで差し替えられる設計にすることで、ネットワーク不要の高速なユニットテストが書ける。

:::
