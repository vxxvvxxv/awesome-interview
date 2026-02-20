# gRPC

Полезные ссылки:
- [gRPC — официальная документация](https://grpc.io/docs/)
- [gRPC-Go — GitHub](https://github.com/grpc/grpc-go)
- [Protobuf → Protobuf.md](Protobuf.md)

---

## Что такое gRPC?

**gRPC** (Google Remote Procedure Call) — высокопроизводительный фреймворк для межсервисного взаимодействия, разработанный Google. Позволяет вызывать методы удалённого сервиса так, будто они локальные.

**Технологический стек:**
- **Транспорт**: HTTP/2
- **Сериализация**: Protocol Buffers (бинарный формат)
- **IDL**: `.proto` файлы — строгий контракт API

```
Client                          Server
  │                               │
  │─── gRPC Call (HTTP/2) ───────►│
  │    [Protobuf payload]         │
  │◄─── Response ─────────────────│
```

## Почему gRPC быстрее REST+JSON?

| Характеристика | REST + JSON | gRPC + Protobuf |
|----------------|-------------|-----------------|
| Транспорт | HTTP/1.1 | HTTP/2 |
| Сериализация | Текст (JSON) | Бинарный (Protobuf) |
| Размер payload | Больше | В 2–10 раз меньше |
| Скорость сериализации | Медленнее | В 5–10 раз быстрее |
| Мультиплексирование | Нет | Да (HTTP/2 streams) |
| Двусторонний стриминг | Нет (WebSocket) | Встроен |
| Contract-first | Опционально | Обязательно (.proto) |
| Browser-поддержка | Да | Через grpc-web |

## HTTP/2 под капотом gRPC

gRPC использует возможности HTTP/2:

- **Мультиплексирование**: несколько параллельных RPC-вызовов по одному TCP-соединению.
- **Бинарный фрейминг**: данные в виде фреймов, не текста.
- **Сжатие заголовков** (HPACK): повторяющиеся заголовки не дублируются.
- **Flow control**: управление потоком на уровне стримов.

```
TCP Connection
├── Stream 1: GetUser(42)
├── Stream 3: ListOrders(user=42)   ← параллельно, одно соединение
└── Stream 5: StreamEvents()
```

## Четыре режима взаимодействия

### 1. Unary (запрос–ответ)

Аналог обычного HTTP-запроса.

```protobuf
rpc GetUser(GetUserRequest) returns (UserResponse);
```

```go
// Клиент
resp, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 42})

// Сервер
func (s *server) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.UserResponse, error) {
    return &pb.UserResponse{Id: req.Id, Name: "Alice"}, nil
}
```

### 2. Server Streaming (сервер стримит)

Клиент один запрос → сервер поток ответов.

```protobuf
rpc ListEvents(ListEventsRequest) returns (stream Event);
```

```go
// Сервер
func (s *server) ListEvents(req *pb.ListEventsRequest, stream pb.Service_ListEventsServer) error {
    for _, event := range db.GetEvents() {
        if err := stream.Send(&pb.Event{Name: event.Name}); err != nil {
            return err
        }
    }
    return nil
}

// Клиент
stream, _ := client.ListEvents(ctx, &pb.ListEventsRequest{})
for {
    event, err := stream.Recv()
    if err == io.EOF { break }
    process(event)
}
```

### 3. Client Streaming (клиент стримит)

Клиент поток запросов → сервер один ответ.

```protobuf
rpc UploadChunks(stream Chunk) returns (UploadResponse);
```

```go
// Клиент
stream, _ := client.UploadChunks(ctx)
for _, chunk := range chunks {
    stream.Send(&pb.Chunk{Data: chunk})
}
resp, err := stream.CloseAndRecv()
```

### 4. Bidirectional Streaming (двусторонний)

Обе стороны одновременно стримят.

```protobuf
rpc Chat(stream ChatMessage) returns (stream ChatMessage);
```

```go
stream, _ := client.Chat(ctx)

go func() {
    for msg := range userInput {
        stream.Send(&pb.ChatMessage{Text: msg})
    }
    stream.CloseSend()
}()

for {
    msg, err := stream.Recv()
    if err == io.EOF { break }
    display(msg)
}
```

## Определение сервиса (.proto)

```protobuf
syntax = "proto3";
package user;
option go_package = "github.com/example/gen/user;userpb";

service UserService {
    rpc GetUser      (GetUserRequest)      returns (UserResponse);
    rpc ListUsers    (ListUsersRequest)    returns (stream UserResponse);
    rpc UploadChunks (stream Chunk)        returns (UploadResponse);
    rpc Chat         (stream ChatMsg)      returns (stream ChatMsg);
}

message GetUserRequest { int64 id = 1; }
message UserResponse {
    int64  id    = 1;
    string name  = 2;
    string email = 3;
}
```

## Генерация кода

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

protoc \
  --go_out=. --go_opt=paths=source_relative \
  --go-grpc_out=. --go-grpc_opt=paths=source_relative \
  api/user.proto
```

Результат:
- `user.pb.go` — структуры данных.
- `user_grpc.pb.go` — интерфейсы клиента и сервера.

## Сервер на Go

```go
type userServer struct {
    pb.UnimplementedUserServiceServer // forward compatibility
}

func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.UserResponse, error) {
    return &pb.UserResponse{Id: req.Id, Name: "Alice"}, nil
}

func main() {
    lis, _ := net.Listen("tcp", ":50051")
    s := grpc.NewServer(grpc.ChainUnaryInterceptor(
        authInterceptor,
        loggingInterceptor,
    ))
    pb.RegisterUserServiceServer(s, &userServer{})
    s.Serve(lis)
}
```

## Interceptors (Middleware)

```go
func loggingInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    start := time.Now()
    resp, err := handler(ctx, req)
    log.Printf("method=%s duration=%s err=%v", info.FullMethod, time.Since(start), err)
    return resp, err
}
```

## Metadata (заголовки)

```go
// Клиент — отправить
md := metadata.Pairs("authorization", "Bearer token")
ctx := metadata.NewOutgoingContext(context.Background(), md)

// Сервер — прочитать
md, _ := metadata.FromIncomingContext(ctx)
token := md.Get("authorization")
```

## Коды ошибок

```go
// Вернуть ошибку
return nil, status.Errorf(codes.NotFound, "user %d not found", req.Id)
return nil, status.Errorf(codes.InvalidArgument, "id must be positive")

// Обработать на клиенте
if st, ok := status.FromError(err); ok {
    switch st.Code() {
    case codes.NotFound:      // 404
    case codes.Unauthenticated: // 401
    }
}
```

| gRPC Code | HTTP аналог | Описание |
|-----------|-------------|---------|
| `OK` | 200 | Успех |
| `InvalidArgument` | 400 | Неверные аргументы |
| `NotFound` | 404 | Не найдено |
| `AlreadyExists` | 409 | Уже существует |
| `PermissionDenied` | 403 | Нет доступа |
| `Unauthenticated` | 401 | Не аутентифицирован |
| `ResourceExhausted` | 429 | Rate limit |
| `Internal` | 500 | Внутренняя ошибка |
| `Unavailable` | 503 | Сервис недоступен |
| `DeadlineExceeded` | 504 | Таймаут |

## Deadline и Cancellation

```go
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

resp, err := client.GetUser(ctx, req)
// Ошибка codes.DeadlineExceeded если сервер не ответил

// Сервер должен проверять ctx
func (s *server) SlowOp(ctx context.Context, req *pb.Req) (*pb.Resp, error) {
    select {
    case result := <-doWork():
        return result, nil
    case <-ctx.Done():
        return nil, status.FromContextError(ctx.Err())
    }
}
```

## TLS

```go
// Сервер с TLS
creds, _ := credentials.NewServerTLSFromFile("server.crt", "server.key")
s := grpc.NewServer(grpc.Creds(creds))

// Клиент с TLS
creds, _ := credentials.NewClientTLSFromFile("ca.crt", "")
conn, _ := grpc.NewClient("server:443", grpc.WithTransportCredentials(creds))
```

## Health Check

```go
import "google.golang.org/grpc/health"
import healthpb "google.golang.org/grpc/health/grpc_health_v1"

healthServer := health.NewServer()
healthpb.RegisterHealthServer(s, healthServer)
healthServer.SetServingStatus("user.UserService", healthpb.HealthCheckResponse_SERVING)
```

## Рефлексия (для grpcurl)

```go
import "google.golang.org/grpc/reflection"
reflection.Register(s) // только для dev/debug!
```

```bash
grpcurl -plaintext localhost:50051 list
grpcurl -plaintext -d '{"id": 42}' localhost:50051 user.UserService/GetUser
```

## gRPC vs REST

| Критерий | gRPC | REST |
|---------|------|------|
| Межсервисное взаимодействие | Отлично | Хорошо |
| Public API / браузер | Плохо (нужен grpc-web) | Отлично |
| Streaming | Встроен | WebSocket / SSE |
| Строгий контракт | Да (.proto) | Опционально (OpenAPI) |
| Latency | Ниже | Выше |
| Отладка (curl) | Сложно | Легко |
