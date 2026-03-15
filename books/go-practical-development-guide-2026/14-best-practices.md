---
title: "ベストプラクティス"
---

# Chapter 14: ベストプラクティス

> コード設計、プロジェクト構成、チーム開発のガイドライン

本章では、実務で繰り返し遭遇する設計判断に対して「Go らしい」回答を整理します。命名規則、プロジェクト構成、エラー設計、依存性注入、リンター/CI の5領域を扱います。

---

## プロジェクト構成

小規模プロジェクトではフラット構成で十分。中〜大規模では以下のレイヤー構成が標準になる。

```
myapp/
├── cmd/
│   └── server/
│       └── main.go           # エントリーポイント（薄く保つ）
├── internal/
│   ├── handler/              # HTTPハンドラ
│   ├── service/              # ビジネスロジック
│   ├── repository/           # データアクセス
│   └── model/                # ドメインモデル
├── pkg/                      # 外部公開ライブラリ（必要な場合のみ）
├── migrations/
├── go.mod
└── go.sum
```

守るべきルールは **依存の方向を一方向に保つ** ことだけです。

```
main → handler → service → repository → model
```

`model` はどこにも依存しない純粋なデータ構造。`internal/` は外部パッケージからのインポートをコンパイラレベルで防ぐ。循環依存が発生したら、インターフェースを使って依存を断ち切る。

---

## 命名規則

Go の命名は「パッケージ名 + 識別子名」で読みます。パッケージ名との重複を避けるのが最重要ルールです。

```go
// NG: パッケージ名の繰り返し
package user
func UserCreate() { ... }  // user.UserCreate() は冗長

// OK: パッケージ名を活用
package user
func Create() { ... }      // user.Create() で明確
```

主要な規約をまとめます。

| 対象 | 規約 | 例 |
|------|------|-----|
| パッケージ名 | 短い・小文字・単数形 | `http`, `user`, `order` |
| エクスポート | PascalCase | `ReadFile`, `HTTPClient` |
| 非エクスポート | camelCase | `readFile`, `httpClient` |
| インターフェース | 1メソッドなら `-er` | `Reader`, `Writer`, `Handler` |
| 頭字語 | 全大文字を維持 | `HTTPHandler`, `userID` |
| 定数 | PascalCase | `MaxRetryCount`（`MAX_RETRY_COUNT` ではない） |
| getter | `Get` を付けない | `Name()` / `SetName()` |

スコープが短い変数は短い名前でよいでしょう。`for i := range items` の `i` や、`func(r io.Reader)` の `r` は Go では自然です。

---

## エラー設計

Go のエラー設計には3つの道具があります。センチネルエラー、カスタムエラー型、エラーラッピングです。

```go
package apperr

import (
    "errors"
    "fmt"
)

// 1. センチネルエラー -- errors.Is() で判定
var (
    ErrNotFound  = errors.New("not found")
    ErrForbidden = errors.New("forbidden")
)

// 2. カスタムエラー型 -- errors.As() で判定
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation: %s - %s", e.Field, e.Message)
}

// 3. エラーラッピング -- コンテキストを付与して返す
func FindUser(id string) (*User, error) {
    user, err := db.Get(id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, fmt.Errorf("FindUser(%s): %w", id, ErrNotFound)
        }
        return nil, fmt.Errorf("FindUser(%s): %w", id, err)
    }
    return user, nil
}
```

エラー処理の判断基準は明快です。

- **呼び出し元がエラーの種類を判断する必要がある** --> センチネルエラー or カスタム型
- **それ以外** --> `fmt.Errorf("...: %w", err)` で文字列ラップ
- **ログはハンドラ層で1回だけ**。途中の層ではラップして返すだけにする
- **エラーを握りつぶさない**。`data, _ := json.Marshal(user)` は事故の元

| 関数 | 比較対象 | 使用場面 |
|------|---------|---------|
| `errors.Is(err, target)` | 値 | `ErrNotFound`, `sql.ErrNoRows` |
| `errors.As(err, &target)` | 型 | `*ValidationError`, `*os.PathError` |
| `errors.Join(e1, e2)` | 複数結合 | 後処理エラーをまとめる（Go 1.20+） |

---

## 依存性注入とインターフェース設計

Go のインターフェース設計には3原則があります。

1. **インターフェースは使う側で定義する**
2. **インターフェースを受け取り、構造体を返す**
3. **必要なメソッドだけを要求する**

```go
// service パッケージ -- 自分が必要なメソッドだけを定義
package service

type UserStore interface {
    GetByID(ctx context.Context, id int64) (*User, error)
}

type UserService struct {
    store UserStore
}

func NewUserService(store UserStore) *UserService {
    return &UserService{store: store}
}

func (s *UserService) GetUser(ctx context.Context, id int64) (*User, error) {
    if id <= 0 {
        return nil, fmt.Errorf("invalid user id: %d", id)
    }
    return s.store.GetByID(ctx, id)
}
```

`repository` パッケージは具体的な構造体を提供するだけ。暗黙的インターフェース実装により、`service.UserStore` の存在を知らなくてよい。

テスト時はモックを差し込む。

```go
type mockUserStore struct {
    users map[int64]*User
    err   error
}

func (m *mockUserStore) GetByID(_ context.Context, id int64) (*User, error) {
    if m.err != nil {
        return nil, m.err
    }
    u, ok := m.users[id]
    if !ok {
        return nil, apperr.ErrNotFound
    }
    return u, nil
}

func TestGetUser(t *testing.T) {
    tests := []struct {
        name    string
        id      int64
        mock    *mockUserStore
        wantErr bool
    }{
        {"success", 1, &mockUserStore{users: map[int64]*User{1: {ID: 1}}}, false},
        {"not found", 999, &mockUserStore{users: map[int64]*User{}}, true},
        {"invalid id", 0, &mockUserStore{}, true},
        {"db error", 1, &mockUserStore{err: fmt.Errorf("connection refused")}, true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            svc := NewUserService(tt.mock)
            _, err := svc.GetUser(context.Background(), tt.id)
            if (err != nil) != tt.wantErr {
                t.Errorf("GetUser() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}
```

**Functional Options パターン** は柔軟な初期化に使う。新オプション追加が既存コードを壊さない。

```go
type Option func(*Server)

func WithReadTimeout(d time.Duration) Option {
    return func(s *Server) { s.readTimeout = d }
}

func NewServer(addr string, opts ...Option) *Server {
    s := &Server{
        addr:        addr,
        readTimeout: 5 * time.Second,  // デフォルト値
        maxConn:     100,
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// 使用: NewServer(":8080", WithReadTimeout(30*time.Second))
```

---

## リンターと CI 設定

`golangci-lint` が業界標準。複数のリンターを統合実行できる。

```yaml
# .golangci.yml
linters:
  enable:
    - errcheck       # エラー無視の検出
    - govet          # 静的解析
    - staticcheck    # 高度な静的解析
    - gosimple       # 簡略化の提案
    - unused         # 未使用コードの検出
    - gofmt          # フォーマットチェック
    - goimports      # import 整理
    - gocritic       # コードレビュー指摘
    - bodyclose      # HTTPレスポンスBody閉じ忘れ
    - prealloc       # スライス事前割当の推奨
```

CI パイプラインに組み込む最小構成は以下のとおり。

```yaml
# GitHub Actions (.github/workflows/ci.yml)
name: CI
on: [push, pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "1.24"
      - run: go vet ./...
      - run: go test -race -cover ./...
      - uses: golangci/golangci-lint-action@v7
        with:
          version: latest
```

`go test -race` は並行処理のデータ競合を検出する。CI では必ず有効にすること。

---

## 避けるべきアンチパターン

**init() の乱用**: `init()` で DB 接続やファイル読み込みを行うと、テスト時にも実行されて問題になる。明示的な初期化関数 `NewApp(cfg Config) (*App, error)` を使う。

**goroutine の Fire-and-Forget**: `go sendEmail(user.Email)` のように投げっぱなしにすると、パニックでプロセスが落ちてもログすら残らない。`errgroup` で管理するか、`recover` 付きのワーカーを使う。

**早期リターンの不足**: ガード節でエラーケースを先に弾き、正常パスのネストを浅く保つ。

```go
// NG                              // OK
if user != nil {                   if user == nil {
    if user.Name != "" {               return errors.New("user is nil")
        // 処理                    }
    }                              if user.Name == "" {
}                                      return errors.New("name required")
                                   }
                                   // 処理
```

---

## まとめ

| 領域 | 要点 |
|------|------|
| プロジェクト構成 | `cmd/` + `internal/` + 一方向依存 |
| 命名 | パッケージ名と重複させない、頭字語は全大文字 |
| エラー設計 | `%w` でラップ、`Is`/`As` で判定、ログは1回 |
| インターフェース | 使う側で定義、小さく保つ、構造体を返す |
| CI | `golangci-lint` + `go test -race` を必ず組み込む |
| アンチパターン | `init()` 乱用、goroutine 放置、エラー握りつぶし |

---

## やってみよう！

### 練習1: エラー設計の実装

オンラインショップの `order` パッケージを設計せよ。以下の要件を満たすこと。

- `ErrOutOfStock`（在庫切れ）と `ErrPaymentFailed`（決済失敗）のセンチネルエラーを定義
- 注文金額が0以下の場合に `*ValidationError` を返す `PlaceOrder` 関数を実装
- ハンドラ層で `errors.Is` / `errors.As` を使ってHTTPステータスコードを振り分ける

### 練習2: インターフェースによるテスタビリティ

天気予報APIからデータを取得する `WeatherService` を設計せよ。

- HTTP呼び出しをインターフェース `WeatherFetcher` として抽象化
- テストではモックを差し込んで、ネットワーク不要でテストできるようにする
- テーブル駆動テストで「正常」「APIエラー」「タイムアウト」の3ケースを書く

### 練習3: golangci-lint の導入

自分の既存プロジェクト（または新規プロジェクト）に `golangci-lint` を導入せよ。

1. `.golangci.yml` を作成し、本章で紹介したリンターを有効にする
2. `golangci-lint run` を実行して指摘を確認する
3. GitHub Actions の CI に組み込み、PR ごとに自動チェックされるようにする
