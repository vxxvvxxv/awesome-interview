# Jaeger — распределённая трассировка

Полезные ссылки:
- [Официальная документация](https://www.jaegertracing.io/docs/)
- [OpenTelemetry Go](https://opentelemetry.io/docs/languages/go/)
- [OpenTelemetry Specification](https://opentelemetry.io/docs/specs/otel/)

---

## Что такое распределённая трассировка?

В монолите легко понять что происходит с запросом. В микросервисах запрос проходит через множество сервисов, и нужно понять:
- Где тратится время?
- Где произошла ошибка?
- Как сервисы взаимодействуют?

**Distributed tracing** = отслеживание пути запроса через все сервисы.

```
User Request
│
└── API Gateway (10ms)
    ├── User Service (15ms)
    │   └── PostgreSQL (12ms) ← узкое место
    └── Order Service (8ms)
        └── Redis (1ms)
```

## Ключевые понятия

### Trace

Полный путь одного запроса через систему. Состоит из **span'ов**.

### Span

Единица работы: название операции + временной промежуток + метаданные.

```
TraceID: abc123
│
└── Span: "HTTP GET /api/orders" (total: 35ms)
    ├── Span: "UserService.GetUser" (15ms)
    │   └── Span: "db.query" (12ms)
    │       tags: {db.type: "postgres", db.statement: "SELECT..."}
    └── Span: "OrderService.ListOrders" (8ms)
        └── Span: "redis.get" (1ms)
```

**Span содержит:**
- `TraceID` — идентификатор всего trace.
- `SpanID` — идентификатор данного span.
- `ParentSpanID` — кто создал этот span.
- `Operation Name` — название операции.
- `Start Time`, `Duration`.
- **Tags** — key-value атрибуты (db.type, http.url).
- **Logs/Events** — события внутри span.
- **Baggage** — контекст, передаваемый через все сервисы.

## OpenTelemetry (OTel)

Современный стандарт для трассировки, метрик, логов. Jaeger — один из бэкендов.

```
Ваш код → OpenTelemetry SDK → OTLP → Jaeger / Zipkin / Tempo / ...
```

## Инструментирование Go с OpenTelemetry

### Инициализация

```go
package main

import (
    "context"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
)

func initTracer(ctx context.Context) (func(), error) {
    // Экспортёр → Jaeger через OTLP
    exporter, err := otlptracehttp.New(ctx,
        otlptracehttp.WithEndpoint("jaeger:4318"),
        otlptracehttp.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }

    // Провайдер
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceName("user-service"),
            semconv.ServiceVersion("1.0.0"),
        )),
        sdktrace.WithSampler(sdktrace.AlwaysSample()), // 100% сэмплинг
        // Для production: TraceIDRatioBased(0.1) — 10%
    )

    otel.SetTracerProvider(tp)

    return func() { tp.Shutdown(context.Background()) }, nil
}
```

### Создание span'ов

```go
var tracer = otel.Tracer("user-service")

func GetUser(ctx context.Context, id int64) (*User, error) {
    // Создаём span
    ctx, span := tracer.Start(ctx, "GetUser",
        trace.WithAttributes(
            attribute.Int64("user.id", id),
        ),
    )
    defer span.End() // важно! закрыть span

    // Дочерний span для БД
    user, err := db.QueryUser(ctx, id) // ctx передаём дальше
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return nil, err
    }

    span.SetAttributes(attribute.String("user.name", user.Name))
    span.SetStatus(codes.Ok, "")
    return user, nil
}
```

### HTTP Middleware (сервер)

```go
import "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"

// Автоматически создаёт span для каждого HTTP запроса
mux := http.NewServeMux()
mux.HandleFunc("/api/users", getUsers)

handler := otelhttp.NewHandler(mux, "http-server")
http.ListenAndServe(":8080", handler)
```

### HTTP Клиент

```go
import "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"

// Автоматически пробрасывает trace context в заголовки
client := &http.Client{
    Transport: otelhttp.NewTransport(http.DefaultTransport),
}

req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
resp, _ := client.Do(req)
```

### gRPC инструментирование

```go
import (
    "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
)

// Сервер
s := grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler()),
)

// Клиент
conn, _ := grpc.NewClient(addr,
    grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
)
```

## Propagation — передача контекста между сервисами

Trace context передаётся через **HTTP заголовки** (W3C TraceContext):

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
              ^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^ ^^
              ver         trace-id                   parent-span-id  flags
```

```go
// Настройка propagator (обычно в main)
otel.SetTextMapPropagator(
    propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{}, // W3C TraceContext
        propagation.Baggage{},
    ),
)
```

## Сэмплинг

Трассировать 100% запросов дорого для высоконагруженных сервисов.

```go
// 100% (dev/staging)
sdktrace.AlwaysSample()

// 10% (production)
sdktrace.TraceIDRatioBased(0.1)

// Никогда
sdktrace.NeverSample()

// Приоритетный (если родитель семплирован — семплируем и мы)
sdktrace.ParentBased(sdktrace.TraceIDRatioBased(0.1))
```

## Jaeger Architecture

```
┌────────────────────────────────────┐
│  Ваши сервисы                      │
│  [Service A] → OTLP → [Service B] │
└────────────┬───────────────────────┘
             │ OTLP (gRPC :4317 / HTTP :4318)
             ▼
┌─────────────────────┐
│  Jaeger Collector   │
│  (принимает spans)  │
└──────────┬──────────┘
           │
    ┌──────▼───────┐
    │   Storage    │
    │ (Cassandra,  │
    │  Elasticsearch,│
    │  in-memory)  │
    └──────┬───────┘
           │
    ┌──────▼───────┐
    │  Jaeger UI   │
    │  :16686      │
    └──────────────┘
```

## Полезные теги/атрибуты (OpenTelemetry Semantic Conventions)

```go
// HTTP
attribute.String("http.method", "GET")
attribute.String("http.url", "https://api.example.com/users")
attribute.Int("http.status_code", 200)

// Database
attribute.String("db.system", "postgresql")
attribute.String("db.name", "mydb")
attribute.String("db.statement", "SELECT * FROM users WHERE id=$1")

// Message Queue
attribute.String("messaging.system", "kafka")
attribute.String("messaging.destination", "user-events")

// gRPC
attribute.String("rpc.system", "grpc")
attribute.String("rpc.service", "UserService")
attribute.String("rpc.method", "GetUser")
```

## Docker Compose для локальной разработки

```yaml
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # UI
      - "4317:4317"    # OTLP gRPC
      - "4318:4318"    # OTLP HTTP
    environment:
      - COLLECTOR_OTLP_ENABLED=true
```

## Что спрашивают на собеседованиях

1. **Зачем нужна трассировка?** → Понять путь запроса в микросервисах, найти узкое место.
2. **Что такое Trace, Span?** → Trace = полный путь; Span = единица операции с временем.
3. **Как span'ы связаны?** → ParentSpanID + общий TraceID.
4. **Как передаётся trace context?** → HTTP-заголовки (W3C traceparent) или gRPC metadata.
5. **Что такое сэмплинг?** → Записываем не все трейсы, а лишь часть (% или правило).
6. **OpenTelemetry vs Jaeger?** → OTel — стандарт + SDK; Jaeger — бэкенд для хранения и UI.
