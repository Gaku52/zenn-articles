---
title: "gRPC"
---

# Chapter 08: gRPC

> Protocol BuffersとGoによるサービス間通信

gRPCはGoogleが開発した高性能RPCフレームワークです。HTTP/2上でProtocol Buffers（バイナリ形式）を使い、**型安全なサービス定義**と**双方向ストリーミング**を実現します。マイクロサービス間通信の定番であり、RESTと比較して10〜100倍のスループットを発揮する場面もあります。

---

## Proto定義とコード生成

gRPCの開発は`.proto`ファイルからスタートします。サービスとメッセージを定義し、`buf generate`でGoコードを自動生成します。

```protobuf
syntax = "proto3";
package user.v1;
option go_package = "gen/user/v1;userv1";

import "google/protobuf/timestamp.proto";

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc WatchUsers(WatchUsersRequest) returns (stream UserEvent);      // Server Streaming
  rpc BatchCreate(stream CreateUserRequest) returns (BatchResponse); // Client Streaming
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);         // Bidirectional
}

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
  UserRole role = 4;
  google.protobuf.Timestamp created_at = 5;
}

enum UserRole {
  USER_ROLE_UNSPECIFIED = 0;  // 0番目は必ずUNSPECIFIED
  USER_ROLE_ADMIN = 1;
  USER_ROLE_MEMBER = 2;
}

message GetUserRequest  { int64 id = 1; }
message GetUserResponse { User user = 1; }
message CreateUserRequest { string name = 1; string email = 2; UserRole role = 3; }
message CreateUserResponse { User user = 1; }
message WatchUsersRequest { repeated int64 user_ids = 1; }
message UserEvent { string type = 1; User user = 2; }
message BatchResponse { int32 created_count = 1; repeated User users = 2; }
message ChatMessage { string sender_id = 1; string content = 2; }
```

`buf generate`を実行すると、`*.pb.go`（メッセージ型）と`*_grpc.pb.go`（サービスインターフェース）が生成されます。

### 4種類のRPCパターン

| パターン | メッセージ | 用途 |
|---------|----------|------|
| Unary | 1:1 | CRUD操作、認証 |
| Server Streaming | 1:N | リアルタイム通知、大量データ配信 |
| Client Streaming | N:1 | ファイルアップロード、バッチ処理 |
| Bidirectional | N:M | チャット、共同編集 |

---

## サーバー実装

生成された`UnimplementedUserServiceServer`を埋め込み、各RPCメソッドを実装します。

```go
type userServer struct {
    userv1.UnimplementedUserServiceServer
    mu     sync.RWMutex
    users  map[int64]*userv1.User
    nextID int64
}

func NewUserServer() *userServer {
    return &userServer{users: make(map[int64]*userv1.User), nextID: 1}
}

func (s *userServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
    if req.Id <= 0 {
        return nil, status.Errorf(codes.InvalidArgument, "invalid user id: %d", req.Id)
    }
    s.mu.RLock()
    defer s.mu.RUnlock()

    user, ok := s.users[req.Id]
    if !ok {
        return nil, status.Errorf(codes.NotFound, "user %d not found", req.Id)
    }
    return &userv1.GetUserResponse{User: user}, nil
}

func (s *userServer) CreateUser(ctx context.Context, req *userv1.CreateUserRequest) (*userv1.CreateUserResponse, error) {
    if req.Name == "" {
        return nil, status.Errorf(codes.InvalidArgument, "name is required")
    }
    s.mu.Lock()
    defer s.mu.Unlock()

    for _, u := range s.users {
        if u.Email == req.Email {
            return nil, status.Errorf(codes.AlreadyExists, "email %s already exists", req.Email)
        }
    }
    user := &userv1.User{
        Id: s.nextID, Name: req.Name, Email: req.Email,
        Role: req.Role, CreatedAt: timestamppb.Now(),
    }
    s.users[s.nextID] = user
    s.nextID++
    return &userv1.CreateUserResponse{User: user}, nil
}
```

gRPCでは`status.Errorf`で適切なステータスコードを返します。`NotFound`、`InvalidArgument`、`AlreadyExists`など16種類のコードがあり、REST APIのHTTPステータスコードに相当します。

---

## ストリーミング

ストリーミングはgRPC最大の差別化ポイントです。RESTではWebSocketやSSEが必要な通信を、ネイティブにサポートします。

```go
// Server Streaming -- サーバーから連続データ送信
func (s *userServer) WatchUsers(req *userv1.WatchUsersRequest, stream userv1.UserService_WatchUsersServer) error {
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-stream.Context().Done():
            return nil // クライアント切断を検出
        case <-ticker.C:
            event := s.checkForChanges(req.UserIds)
            if event != nil {
                if err := stream.Send(event); err != nil {
                    return status.Errorf(codes.Internal, "send failed: %v", err)
                }
            }
        }
    }
}

// Client Streaming -- クライアントから連続データ受信
func (s *userServer) BatchCreate(stream userv1.UserService_BatchCreateServer) error {
    var created []*userv1.User
    for {
        req, err := stream.Recv()
        if err == io.EOF {
            return stream.SendAndClose(&userv1.BatchResponse{
                CreatedCount: int32(len(created)), Users: created,
            })
        }
        if err != nil {
            return err
        }
        resp, err := s.CreateUser(stream.Context(), req)
        if err != nil {
            continue // エラーをスキップして続行
        }
        created = append(created, resp.User)
    }
}
```

`stream.Context().Done()`によるクライアント切断の検出は必須です。これを忘れるとリソースリークの原因になります。

---

## クライアント実装

```go
type UserClient struct {
    conn   *grpc.ClientConn
    client userv1.UserServiceClient
}

func NewUserClient(addr string) (*UserClient, error) {
    conn, err := grpc.NewClient(addr,
        grpc.WithTransportCredentials(insecure.NewCredentials()), // 開発用。本番はTLS必須
        grpc.WithKeepaliveParams(keepalive.ClientParameters{
            Time: 10 * time.Second, Timeout: 3 * time.Second,
        }),
    )
    if err != nil {
        return nil, fmt.Errorf("grpc dial: %w", err)
    }
    return &UserClient{conn: conn, client: userv1.NewUserServiceClient(conn)}, nil
}

func (c *UserClient) Close() error { return c.conn.Close() }

func (c *UserClient) GetUser(ctx context.Context, id int64) (*userv1.User, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second) // タイムアウト必須
    defer cancel()

    resp, err := c.client.GetUser(ctx, &userv1.GetUserRequest{Id: id})
    if err != nil {
        return nil, fmt.Errorf("GetUser: %w", err)
    }
    return resp.User, nil
}
```

呼び出し側では**必ず`context.WithTimeout`でタイムアウトを設定**してください。デフォルトではタイムアウトが無く、サーバーが応答しない場合にクライアントが永遠にブロックされます。

---

## インターセプタ（ミドルウェア）

インターセプタはgRPC版のHTTPミドルウェアです。ロギング、パニックリカバリ、認証といった横断的関心事を処理します。

```go
// ロギング
func LoggingUnaryInterceptor(
    ctx context.Context, req any,
    info *grpc.UnaryServerInfo, handler grpc.UnaryHandler,
) (any, error) {
    start := time.Now()
    resp, err := handler(ctx, req)
    code := codes.OK
    if err != nil {
        code = status.Code(err)
    }
    log.Printf("[gRPC] %s code=%s duration=%v", info.FullMethod, code, time.Since(start))
    return resp, err
}

// パニックリカバリ
func RecoveryUnaryInterceptor(
    ctx context.Context, req any,
    info *grpc.UnaryServerInfo, handler grpc.UnaryHandler,
) (resp any, err error) {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("[PANIC] %s: %v", info.FullMethod, r)
            err = status.Errorf(codes.Internal, "internal server error")
        }
    }()
    return handler(ctx, req)
}
```

### サーバー起動とインターセプタチェーン

```go
func main() {
    lis, _ := net.Listen("tcp", ":50051")

    s := grpc.NewServer(
        grpc.ChainUnaryInterceptor(
            RecoveryUnaryInterceptor, // 1. パニック回復（最外側）
            LoggingUnaryInterceptor,  // 2. ロギング
            AuthUnaryInterceptor,     // 3. 認証
        ),
    )
    userv1.RegisterUserServiceServer(s, NewUserServer())

    // Graceful Shutdown
    go func() { s.Serve(lis) }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    s.GracefulStop() // 進行中のRPCが完了するまで待つ
}
```

`ChainUnaryInterceptor`に渡す順番が実行順です。パニックリカバリを最外側に置くことで、後段のインターセプタやハンドラのパニックを全てキャッチできます。

---

## アンチパターン

### 巨大メッセージを一括送信

```protobuf
// NG: 100万件を一括返却 → メモリ不足
message ListAllResponse { repeated User users = 1; }

// OK: ページネーションまたはServer Streamingで分割
rpc StreamUsers(StreamUsersRequest) returns (stream User);
```

### Contextを無視する

```go
// NG: クライアントがキャンセルしても止まらない
func (s *server) SlowRPC(ctx context.Context, req *pb.Request) (*pb.Response, error) {
    result := heavyComputation() // ctx.Done()をチェックしない
    return &pb.Response{Data: result}, nil
}

// OK: selectでキャンセルを検出して早期終了
func (s *server) SlowRPC(ctx context.Context, req *pb.Request) (*pb.Response, error) {
    ch := make(chan string, 1)
    go func() { ch <- heavyComputation() }()
    select {
    case result := <-ch:
        return &pb.Response{Data: result}, nil
    case <-ctx.Done():
        return nil, status.FromContextError(ctx.Err()).Err()
    }
}
```

---

## まとめ

| 概念 | 要点 |
|------|------|
| Protocol Buffers | `.proto`ファイルでサービスとメッセージを定義し、`buf generate`でGoコードを自動生成する |
| 4種類のRPCパターン | Unary（1:1）、Server Streaming（1:N）、Client Streaming（N:1）、Bidirectional（N:M） |
| ステータスコード | `status.Errorf`で`NotFound`・`InvalidArgument`・`AlreadyExists`など16種類のコードを返す |
| ストリーミング | RESTのWebSocket/SSEに相当する通信をネイティブにサポート。`stream.Context().Done()`でクライアント切断を検出必須 |
| クライアント実装 | `context.WithTimeout`でタイムアウトを必ず設定。デフォルトではタイムアウトが無くブロックされる |
| インターセプタ | gRPC版ミドルウェア。`ChainUnaryInterceptor`で渡す順番が実行順。パニックリカバリを最外側に配置 |
| Graceful Shutdown | `s.GracefulStop()`で進行中のRPCが完了するまで待ってから停止する |
| アンチパターン | 巨大メッセージの一括送信を避けストリーミングで分割。Contextを無視せず`ctx.Done()`で早期終了する |

---

## やってみよう！

**練習1: TODOサービスの実装**

以下の`.proto`定義からgRPCサーバーとクライアントを実装してください。`status.Errorf`を使ったエラーハンドリングも忘れずに。

```protobuf
service TodoService {
  rpc CreateTodo(CreateTodoRequest) returns (CreateTodoResponse);
  rpc ListTodos(ListTodosRequest) returns (ListTodosResponse);
  rpc CompleteTodo(CompleteTodoRequest) returns (CompleteTodoResponse);
}
```

**練習2: Server Streamingの体験**

練習1に`WatchTodos`メソッドを追加し、TODOが追加・完了されるたびにイベントをストリーミング配信してください。`stream.Context().Done()`によるキャンセル対応を実装すること。

**練習3: ロギングインターセプタの拡張**

本章のロギングインターセプタを拡張して、(1) メタデータから`x-request-id`を取得して出力、(2) エラー時にはエラーメッセージも出力、(3) レスポンスサイズの記録 — を追加してください。
