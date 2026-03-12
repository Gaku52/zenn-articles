---
title: "Gin / Echo"
---

# Chapter 06: Gin / Echo

> 人気Webフレームワークの使い方と使い分け

前章では `net/http` だけでWebサーバーを構築しました。標準ライブラリだけでも十分実用的ですが、ルーティングの柔軟性・バリデーション・ミドルウェアの管理を効率化したい場面では、Webフレームワークが強力な選択肢になります。

この章では、Goで最も人気のある2つのフレームワーク **Gin** と **Echo** を取り上げ、基本的な使い方から本番運用に必要なパターンまでを実践的に解説します。

---

## Ginの基本

Ginは GitHub Stars 80k超 を誇るGoで最も人気のあるWebフレームワークです。Radix tree ベースの高速ルーティングと、シンプルなAPIが特徴です。

```go
package main

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default() // Logger + Recovery ミドルウェア付き

    r.GET("/users", listUsers)
    r.POST("/users", createUser)
    r.GET("/users/:id", getUser)
    r.PUT("/users/:id", updateUser)
    r.DELETE("/users/:id", deleteUser)

    r.Run(":8080")
}

func getUser(c *gin.Context) {
    id := c.Param("id")                    // パスパラメータ
    c.JSON(http.StatusOK, gin.H{"id": id, "name": "Tanaka"})
}

func listUsers(c *gin.Context) {
    page := c.DefaultQuery("page", "1")    // クエリパラメータ（デフォルト値付き）
    limit := c.Query("limit")              // クエリパラメータ（空文字がデフォルト）
    c.JSON(http.StatusOK, gin.H{"page": page, "limit": limit})
}
```

`Default()` は `Logger` と `Recovery` ミドルウェアが自動で組み込まれたエンジンを返します。本番環境では `gin.New()` で素のエンジンを取得し、必要なミドルウェアを明示的に追加するのが推奨です。

ルーティングパラメータは `:id` 形式で定義し `c.Param("id")` で取得します。ワイルドカードは `*filepath` 形式です。

---

## Echoの基本

Echoは Gin と並ぶ人気フレームワークで、最大の特徴は**ハンドラが `error` を返す**点です。これによりエラーハンドリングが集約しやすくなります。

```go
package main

import (
    "errors"
    "net/http"

    "github.com/labstack/echo/v4"
    "github.com/labstack/echo/v4/middleware"
)

func main() {
    e := echo.New()
    e.Use(middleware.Logger())
    e.Use(middleware.Recover())

    e.GET("/users/:id", getUser)

    e.Logger.Fatal(e.Start(":8080"))
}

func getUser(c echo.Context) error {
    id := c.Param("id")
    user, err := findUserByID(c.Request().Context(), id)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            return echo.NewHTTPError(http.StatusNotFound, "user not found")
        }
        return echo.NewHTTPError(http.StatusInternalServerError, err.Error())
    }
    return c.JSON(http.StatusOK, user)
}
```

Gin では `c.JSON()` を呼んだ後に `return` を忘れるとバグになりますが、Echo では `return c.JSON(...)` と書くため、この種のミスが起きにくい設計です。

クエリパラメータは `c.QueryParam("page")` で取得します。Gin の `c.Query()` と役割は同じです。

---

## ミドルウェア

### Ginのルーティンググループとミドルウェア

Gin ではルーティンググループ単位でミドルウェアを適用できます。認証やロール制御はこのパターンで実装するのが定石です。

```go
func main() {
    r := gin.New()
    r.Use(gin.Recovery())

    // 公開API
    public := r.Group("/api/v1")
    {
        public.POST("/login", login)
        public.POST("/register", register)
    }

    // 認証必須API
    authorized := r.Group("/api/v1")
    authorized.Use(authMiddleware())
    {
        authorized.GET("/profile", getProfile)
        authorized.PUT("/profile", updateProfile)
    }

    // 管理者API
    admin := r.Group("/api/v1/admin")
    admin.Use(authMiddleware(), adminMiddleware())
    {
        admin.GET("/users", adminListUsers)
    }

    r.Run(":8080")
}

func authMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.AbortWithStatusJSON(401, gin.H{"error": "unauthorized"})
            return
        }
        claims, err := validateToken(token)
        if err != nil {
            c.AbortWithStatusJSON(401, gin.H{"error": "invalid token"})
            return
        }
        c.Set("userID", claims.UserID)
        c.Next() // 後続のミドルウェア/ハンドラを実行
    }
}
```

ミドルウェアの実行順序はスタック構造です。`c.Next()` で後続処理が実行され、戻ってきた後に after 処理が走ります。`c.Abort()` でチェーンを中断できます。

### Echoの組み込みミドルウェア

Echo は CORS、レート制限、セキュリティヘッダなどが標準で組み込まれています。Gin ではこれらをサードパーティや自前実装で賄う必要がある点が違いです。

```
Echo 組み込み:  CORS, RateLimiter, Secure, RequestID, Timeout, Gzip, CSRF, BodyLimit
Gin 組み込み:   Logger, Recovery, BasicAuth（他は別途追加）
```

---

## バリデーション

### Ginのバリデーション（binding タグ）

Gin は内部で `go-playground/validator` を使用しており、構造体タグでバリデーションルールを宣言できます。

```go
type CreateUserRequest struct {
    Name     string `json:"name" binding:"required,min=2,max=50"`
    Email    string `json:"email" binding:"required,email"`
    Age      int    `json:"age" binding:"gte=0,lte=150"`
    Password string `json:"password" binding:"required,min=8"`
}

func createUser(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    // バリデーション通過 -- ビジネスロジックへ
    c.JSON(http.StatusCreated, gin.H{"name": req.Name, "email": req.Email})
}
```

よく使う binding タグをまとめます。

| タグ | 意味 | 例 |
|------|------|-----|
| `required` | 必須 | `binding:"required"` |
| `email` | メール形式 | `binding:"email"` |
| `min` / `max` | 最小/最大値（または文字列長） | `binding:"min=3,max=100"` |
| `oneof` | 列挙値 | `binding:"oneof=admin user"` |
| `gte` / `lte` | 以上/以下 | `binding:"gte=0,lte=150"` |
| `url` | URL形式 | `binding:"url"` |

### Echoのバリデーション

Echo にはバリデータが組み込まれていないため、`echo.Validator` インターフェースを実装して登録します。内部で使うライブラリは同じ `go-playground/validator` です。

```go
type CustomValidator struct {
    validator *validator.Validate
}

func (cv *CustomValidator) Validate(i any) error {
    return cv.validator.Struct(i)
}

func main() {
    e := echo.New()
    e.Validator = &CustomValidator{validator: validator.New()}

    e.POST("/users", func(c echo.Context) error {
        u := new(CreateUserRequest)
        if err := c.Bind(u); err != nil {
            return echo.NewHTTPError(http.StatusBadRequest, err.Error())
        }
        if err := c.Validate(u); err != nil {
            return echo.NewHTTPError(http.StatusBadRequest, err.Error())
        }
        return c.JSON(http.StatusCreated, u)
    })
}
```

Gin は `ShouldBindJSON` で Bind + Validate が一括ですが、Echo は `c.Bind()` と `c.Validate()` が分離しています。好みの問題ですが、Echo の方がバインドとバリデーションの責務が明確です。

---

## Gin と Echo の選び方

### 比較表

| 項目 | Gin | Echo |
|------|-----|------|
| ハンドラ型 | `func(*gin.Context)` | `func(echo.Context) error` |
| エラーハンドリング | `c.JSON()` + `return` を手動 | `return error` で集約 |
| バリデーション | binding タグで一括 | Bind + Validate 分離 |
| 組み込みミドルウェア | 少ない（Logger, Recovery） | 豊富（CORS, Rate Limit等） |
| カスタムコンテキスト | `c.Set()/c.Get()` で値を格納 | カスタム Context 型を定義可能 |
| コミュニティ | GitHub Stars 80k+、情報量が最も多い | GitHub Stars 30k+、設計がクリーン |
| Swagger連携 | gin-swagger | echo-swagger |

### 選び方の指針

**Gin を選ぶとき:**
- 大きなコミュニティ、豊富な日本語情報が欲しい
- 既存の Gin プロジェクトに合わせる必要がある
- バリデーションの組み込みが手軽

**Echo を選ぶとき:**
- `error` 戻り値パターンが好み
- 組み込みミドルウェアの豊富さを活かしたい
- カスタムコンテキストで型安全に値を扱いたい

**どちらでもないとき:**
- Go 1.22+ の `net/http` 新ルーティングで十分な小規模 API → 標準ライブラリ
- gRPC 中心のマイクロサービス → フレームワーク不要

パフォーマンスの差はほぼありません。チームの好みとプロジェクトの要件で選んでください。

---

## アンチパターン

### gin.Context を goroutine に渡す

```go
// BAD: gin.Context はリクエストのライフサイクルに紐づく
func handler(c *gin.Context) {
    go func() {
        time.Sleep(5 * time.Second)
        c.JSON(200, gin.H{"ok": true}) // レスポンスは既に返却済みの可能性
    }()
}

// GOOD: 必要な値をコピーしてから goroutine に渡す
func handler(c *gin.Context) {
    userID := c.GetString("userID")
    ctx := c.Request.Context()
    go func() {
        processAsync(ctx, userID)
    }()
    c.JSON(200, gin.H{"accepted": true})
}
```

`gin.Context` はリクエストが完了すると再利用されるため、goroutine に渡すと不正な動作やパニックの原因になります。

### バリデーションの二重実装

```go
// BAD: binding タグで定義したのに、ハンドラ内でも手書きチェック
if len(req.Name) < 2 { ... }
if !strings.Contains(req.Email, "@") { ... }

// GOOD: バリデーションは binding タグに集約
type CreateUserRequest struct {
    Name  string `json:"name" binding:"required,min=2,max=50"`
    Email string `json:"email" binding:"required,email"`
}
```

バリデーションロジックが分散するとルール変更時に漏れが発生します。構造体タグに集約しましょう。

---

## まとめ

| 概念 | 要点 |
|------|------|
| Gin の基本 | `gin.Default()` で Logger+Recovery 付きエンジンを取得。`:id` 形式のパスパラメータと `c.Query()` でリクエスト情報を取得 |
| Echo の基本 | ハンドラが `error` を返す設計で、`return c.JSON(...)` によりレスポンス返却忘れを防止 |
| ミドルウェア | Gin はルーティンググループ単位で適用。Echo は CORS・RateLimit 等が標準組み込みで豊富 |
| バリデーション | Gin は `binding` タグで Bind+Validate 一括。Echo は `c.Bind()` と `c.Validate()` が分離し責務が明確 |
| 選び方 | Gin はコミュニティと情報量、Echo は設計のクリーンさが強み。性能差はほぼなくチームの好みで選ぶ |
| Context と goroutine | `gin.Context` はリクエスト完了後に再利用されるため、goroutine には必要な値をコピーして渡す |
| バリデーション集約 | ハンドラ内の手書きチェックと binding タグの二重実装を避け、構造体タグに一元化する |

---

## やってみよう！

### 演習1: Gin で TODO API を作る

以下の仕様で TODO 管理の REST API を Gin で実装してください。

- `POST /todos` -- TODO 作成（Title 必須、3文字以上）
- `GET /todos` -- TODO 一覧
- `GET /todos/:id` -- TODO 取得
- `PUT /todos/:id` -- TODO 更新（完了フラグの切り替え）
- `DELETE /todos/:id` -- TODO 削除

データはメモリ上（`map` や `slice`）で管理して構いません。バリデーションには binding タグを使いましょう。

### 演習2: Echo で同じ API を作り直す

演習1 と同じ仕様の TODO API を Echo で実装してください。以下の点に注目して違いを体感しましょう。

- ハンドラが `error` を返すパターン
- `c.Bind()` + `c.Validate()` の分離
- `echo.NewHTTPError()` によるエラー返却

### 演習3: 認証ミドルウェアを追加する

演習1 または 2 の API に、簡易的な API キー認証ミドルウェアを追加してください。

- `Authorization: Bearer <api-key>` ヘッダーを検証
- 正しいキーは環境変数 `API_KEY` から読み取る
- `GET /todos`（一覧取得）は公開、それ以外のエンドポイントに認証を適用（グループ機能を使う）
