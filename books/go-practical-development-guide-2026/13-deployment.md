---
title: "デプロイ"
---

# Chapter 13: デプロイ

> Docker, CI/CD, クラウド — 本番環境への道

Goは**シングルバイナリ**にコンパイルされる。外部ランタイム不要、起動は数ミリ秒。この特性がデプロイを圧倒的にシンプルにする。この章ではビルドからコンテナ化、CI/CD、本番運用までの実践パターンを身につけます。

---

## ビルドとバイナリ

`CGO_ENABLED=0` で完全静的リンクのバイナリを生成できる。外部ライブラリへの依存がなくなり、空のコンテナ（scratch）でもそのまま動く。

```bash
# 本番向けビルドの基本形
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
  go build -trimpath -ldflags="-w -s" -o server ./cmd/server
```

**各フラグの意味:**

| フラグ | 効果 |
|--------|------|
| `CGO_ENABLED=0` | C依存を排除し完全静的リンク |
| `-trimpath` | ビルドパスを削除（セキュリティ向上） |
| `-ldflags="-w -s"` | デバッグ情報・シンボル削除（30-50%縮小） |
| `GOOS` / `GOARCH` | クロスコンパイルのターゲット指定 |

バージョン情報をバイナリに埋め込むのも `-ldflags` の仕事です。

```go
// ビルド時に -ldflags "-X main.version=v1.2.3" で注入
var version = "dev"

func main() {
	if len(os.Args) > 1 && os.Args[1] == "--version" {
		fmt.Printf("server %s (Go %s)\n", version, runtime.Version())
		return
	}
	// サーバー起動...
}
```

---

## Dockerマルチステージビルド

Goのデプロイで最も多いパターン。ビルド環境（850MB）と実行環境（15MB）を分離する。

```dockerfile
# ===== Stage 1: ビルド =====
FROM golang:1.24-alpine AS builder

RUN apk add --no-cache ca-certificates tzdata
RUN adduser -D -g '' appuser

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download && go mod verify

COPY . .

ARG VERSION=dev
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -trimpath \
    -ldflags="-w -s -X main.version=${VERSION}" \
    -o /app/server ./cmd/server

# ===== Stage 2: 実行（最小イメージ） =====
FROM gcr.io/distroless/static-debian12

COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /etc/passwd /etc/passwd

USER appuser
COPY --from=builder /app/server /server

EXPOSE 8080
ENTRYPOINT ["/server"]
```

**ポイント:**

- `go.mod` / `go.sum` を先にコピーして `go mod download` → Dockerレイヤキャッシュが効く。ソース変更時も依存の再取得をスキップできる
- `distroless/static` はシェルもパッケージマネージャもない最小イメージ（約2MB）。攻撃対象面を極限まで縮小する
- `USER appuser` で非rootユーザー実行。コンテナ内でroot権限を持たせない

**ベースイメージの選び方:**

| ベースイメージ | サイズ | シェル | 用途 |
|--------------|--------|--------|------|
| `golang:1.24-alpine` | 350MB | あり | ビルドステージ専用 |
| `distroless/static` | 2MB | なし | 本番推奨 |
| `scratch` | 0MB | なし | 最小構成（CA証明書の手動コピー必須） |
| `alpine:3.21` | 7MB | あり | CGO必要時 |

---

## CI/CD（GitHub Actions）

PR時にLint・テスト・セキュリティチェックを走らせ、タグpush時にリリースとDockerイメージをビルドする。

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "1.24"
      - uses: golangci/golangci-lint-action@v7
        with:
          version: latest

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "1.24"
      - run: go test -race -coverprofile=coverage.out ./...
      - uses: codecov/codecov-action@v5
        with:
          file: ./coverage.out

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "1.24"
      - run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...

  build:
    needs: [lint, test, security]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "1.24"
      - run: CGO_ENABLED=0 go build -trimpath -ldflags="-w -s" -o bin/server ./cmd/server
      - uses: actions/upload-artifact@v4
        with:
          name: server
          path: bin/server
```

**パイプラインの流れ:**

1. **Lint** -- `golangci-lint` でコードスタイルと潜在バグを検出
2. **Test** -- `-race` フラグでデータ競合も検出、カバレッジを収集
3. **Security** -- `govulncheck` で既知の脆弱性をスキャン
4. **Build** -- 全チェック通過後にバイナリを生成

タグpush時のリリースには **GoReleaser** を使うと、マルチプラットフォームバイナリ + Dockerイメージ + GitHub Releaseを一括で自動化できる。

---

## グレースフルシャットダウン

Kubernetesやコンテナオーケストレーターは停止時にSIGTERMを送る。これを受けて「新しいリクエストを止め、処理中のリクエストが完了するまで待ち、リソースを解放する」のがGraceful Shutdown。

```go
package main

import (
	"context"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	})
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(2 * time.Second) // 重い処理のシミュレーション
		w.Write([]byte("done"))
	})

	srv := &http.Server{
		Addr:         ":8080",
		Handler:      mux,
		ReadTimeout:  15 * time.Second,
		WriteTimeout: 15 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	// サーバーをgoroutineで起動
	go func() {
		log.Printf("Server starting on %s", srv.Addr)
		if err := srv.ListenAndServe(); err != http.ErrServerClosed {
			log.Fatalf("Server error: %v", err)
		}
	}()

	// SIGINT / SIGTERM を待つ
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	sig := <-quit
	log.Printf("Received signal: %s, shutting down...", sig)

	// 30秒以内に処理中のリクエストを完了させる
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	if err := srv.Shutdown(ctx); err != nil {
		log.Printf("Shutdown error: %v", err)
	}
	log.Println("Server stopped gracefully")
}
```

**3つのフェーズ:**

1. **新規リクエスト停止** -- `srv.Shutdown` がリスナーを閉じる
2. **処理中リクエスト完了待ち** -- タイムアウト付きで待機
3. **リソース解放** -- DB接続プール等のClose

**ヘルスチェックの実装も忘れずに:**

- `/healthz` -- Liveness Probe。プロセスが生きているか（常に200 OK）
- `/readyz` -- Readiness Probe。トラフィックを受ける準備ができているか（DB疎通チェック等）

---

## クラウドデプロイ

### Google Cloud Run

コンテナをデプロイするだけでスケーリングやTLSが自動で付く。Goとの相性が良い。

```bash
# ソースから直接デプロイ（Dockerfileがあれば自動ビルド）
gcloud run deploy myapp \
    --source . \
    --region asia-northeast1 \
    --allow-unauthenticated \
    --min-instances 0 \
    --max-instances 10 \
    --memory 256Mi \
    --cpu 1
```

Cloud Runは `PORT` 環境変数でリッスンポートを指定するので、`os.Getenv("PORT")` でポートを読み取ること。

### AWS Lambda

Go 1.24のバイナリをzipで固めてデプロイ。コールドスタートが数十ミリ秒と高速。

```bash
# ARM64（Graviton2）でビルドしてデプロイ
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 \
    go build -trimpath -ldflags="-w -s" -o bootstrap ./cmd/lambda
zip function.zip bootstrap

aws lambda update-function-code \
    --function-name myfunction \
    --zip-file fileb://function.zip \
    --architectures arm64
```

---

## 避けるべきアンチパターン

### ビルドステージをそのまま本番に使う

```dockerfile
# NG: golang:1.24 をそのままデプロイ（850MB+、ビルドツール入り）
FROM golang:1.24
COPY . .
RUN go build -o server .
CMD ["./server"]

# OK: マルチステージビルドで実行イメージを分離（15-20MB）
```

### Graceful Shutdownなし

```go
// NG: SIGTERMで処理中リクエストが即切断
func main() {
    http.ListenAndServe(":8080", handler)
}

// OK: Shutdown()で安全に停止（上のコード例を参照）
```

---

## まとめ

| 概念 | 要点 |
|------|------|
| ビルド | `CGO_ENABLED=0` + `-trimpath` + `-ldflags="-w -s"` で静的バイナリを生成。`-ldflags -X` でバージョン埋め込み |
| Dockerマルチステージビルド | ビルド用（golang:alpine）と実行用（distroless/static）を分離し、最終イメージを15-20MBに抑える |
| レイヤキャッシュ | `go.mod` / `go.sum` を先にCOPYして `go mod download` → ソース変更時の再ビルドを高速化 |
| CI/CD | GitHub ActionsでLint → Test（-race） → Security（govulncheck） → Buildの4段パイプライン |
| グレースフルシャットダウン | SIGTERM受信 → 新規リクエスト停止 → 処理中リクエスト完了待ち → リソース解放の3フェーズ |
| ヘルスチェック | `/healthz`（Liveness）と `/readyz`（Readiness）で、オーケストレーターにプロセス状態を伝える |
| クラウドデプロイ | Cloud Runはコンテナ、LambdaはzipでGoバイナリをデプロイ。どちらも起動が高速 |
| アンチパターン | ビルドイメージの本番利用（巨大+攻撃面大）、Graceful Shutdownなし（リクエスト断裂） |

---

## やってみよう！

### 練習1: Dockerマルチステージビルド

自分のGoプロジェクト（なければ `net/http` でHello Worldを返すサーバー）をDockerマルチステージビルドでコンテナ化してみよう。

- `distroless/static` ベースで最終イメージを作る
- `docker images` でイメージサイズを確認する
- 非rootユーザーで実行されていることを確認する

### 練習2: GitHub Actions CI

自分のリポジトリに以下を含むCIワークフローを作成してみよう。

- `go test -race ./...` によるテスト
- `golangci-lint` によるLintチェック
- `govulncheck` による脆弱性スキャン
- PRを出してCIが通ることを確認する

### 練習3: Graceful Shutdown

Graceful Shutdownを実装したHTTPサーバーを作り、以下を検証してみよう。

1. `curl` で3秒かかるリクエストを送信中に `Ctrl+C` でサーバーを停止
2. レスポンスが正常に返ることを確認
3. `Shutdown` のタイムアウトを1秒に短くして、同じことを試す。今度はどうなるか？
