# Паттерны микросервисной архитектуры

Полезные ссылки:
- [microservices.io — Chris Richardson](https://microservices.io/patterns/)
- [Designing Distributed Systems (Burns)](https://www.oreilly.com/library/view/designing-distributed-systems/9781491983638/)
- [CQRS by Martin Fowler](https://martinfowler.com/bliki/CQRS.html)

---

## Что такое микросервисная архитектура?

**Монолит** — одно приложение, все функции вместе. Просто развернуть, сложно масштабировать по частям.

**Микросервисы** — набор маленьких автономных сервисов, каждый отвечает за свою бизнес-функцию.

```
Монолит:                  Микросервисы:
┌──────────────┐          ┌────────┐ ┌─────────┐ ┌─────────┐
│  Order       │          │ Order  │ │ Payment │ │ Catalog │
│  Payment     │   →      │ Service│ │ Service │ │ Service │
│  Catalog     │          └────────┘ └─────────┘ └─────────┘
│  Delivery    │              │           │           │
└──────────────┘          ┌───────────────────────────┐
                          │         Message Bus        │
                          └───────────────────────────┘
```

## API Gateway

Единая точка входа для клиентов:

```
Client → API Gateway → Service A
                    → Service B
                    → Service C

Функции API Gateway:
- Аутентификация / Авторизация
- Rate limiting
- SSL termination
- Load balancing
- Request routing
- Request/Response transformation
- Circuit breaking
- Observability (logging, tracing)
```

```go
// Простой API Gateway на Go с chi
func main() {
    r := chi.NewRouter()
    r.Use(authMiddleware)       // JWT проверка
    r.Use(rateLimitMiddleware)  // Rate limit
    r.Use(tracingMiddleware)    // OpenTelemetry

    // Прокси к микросервисам
    r.Mount("/orders", httputil.NewSingleHostReverseProxy(orderServiceURL))
    r.Mount("/products", httputil.NewSingleHostReverseProxy(catalogServiceURL))

    http.ListenAndServe(":80", r)
}
```

**Популярные решения:** Kong, Nginx, Envoy, AWS API Gateway, Traefik.

## Database per Service

**Каждый микросервис имеет свою БД** — слабая связанность.

```
Order Service   → PostgreSQL (orders DB)
Payment Service → PostgreSQL (payments DB)
Catalog Service → MongoDB (catalog DB)
Session Service → Redis (sessions)
```

**Преимущества:**
- Независимое масштабирование.
- Независимый выбор БД.
- Слабая связанность.

**Проблемы:**
- Нет JOIN через сервисы.
- Распределённые транзакции (решается через Saga).
- Сложнее поддерживать консистентность.

## Shared Database (анти-паттерн)

Несколько сервисов используют одну БД.

```
Order Service ─┐
Payment Service┤→ Shared PostgreSQL (ПЛОХО)
Catalog Service┘
```

**Проблемы:** тесная связанность, конкуренция за ресурсы, сложность масштабирования. Допустимо только при постепенной миграции с монолита.

## Saga Pattern

Распределённые транзакции без двухфазного commit.

### Choreography Saga (хореография)

Сервисы реагируют на события без центрального координатора:

```
Order Service          Payment Service        Inventory Service
      │                      │                      │
      │── OrderCreated ──────►│                      │
      │                      │── PaymentProcessed ──►│
      │                      │                      │── ItemReserved ──►
      │◄──────────────────────────────────── OrderCompleted ──
```

```go
// Order Service публикует событие
kafka.Publish("order.created", OrderCreatedEvent{OrderID: id, Amount: total})

// Payment Service слушает и реагирует
kafka.Subscribe("order.created", func(e OrderCreatedEvent) {
    if err := processPayment(e); err != nil {
        kafka.Publish("payment.failed", PaymentFailedEvent{OrderID: e.OrderID})
        return
    }
    kafka.Publish("payment.completed", PaymentCompletedEvent{OrderID: e.OrderID})
})
```

**Плюс:** нет единой точки отказа.  
**Минус:** сложно понять общий flow, трудно отлаживать.

### Orchestration Saga (оркестрация)

Центральный оркестратор управляет флоу:

```
Order Orchestrator:
  1. → Payment Service: ProcessPayment
  2. ← Success/Failure
  3. → Inventory Service: ReserveItems
  4. ← Success/Failure
  5. → Shipping Service: CreateShipment
  ...
  
При ошибке — компенсирующие транзакции:
  Failure → Inventory: CancelReservation
          → Payment: RefundPayment
```

```go
type OrderSaga struct {
    state      SagaState
    orderID    string
    compensations []func() error
}

func (s *OrderSaga) Run(ctx context.Context) error {
    // Шаг 1: Payment
    if err := s.processPayment(); err != nil {
        return s.compensate()
    }
    s.compensations = append(s.compensations, s.refundPayment)

    // Шаг 2: Reserve
    if err := s.reserveInventory(); err != nil {
        return s.compensate()
    }
    s.compensations = append(s.compensations, s.releaseInventory)

    // Шаг 3: Ship
    if err := s.createShipment(); err != nil {
        return s.compensate()
    }
    return nil
}

func (s *OrderSaga) compensate() error {
    // Компенсирующие транзакции в обратном порядке
    for i := len(s.compensations) - 1; i >= 0; i-- {
        if err := s.compensations[i](); err != nil {
            log.Error("compensation failed", err)
        }
    }
    return ErrSagaFailed
}
```

## Circuit Breaker

Защита от каскадных отказов:

```
States:
  CLOSED (normal) → OPEN (failing) → HALF-OPEN (testing)
       ↑__________________________|

CLOSED:    все запросы проходят, считаем ошибки
           если errors > threshold → переход в OPEN

OPEN:      все запросы сразу отклоняются (fast fail)
           через timeout → переход в HALF-OPEN

HALF-OPEN: пускаем несколько запросов
           если успех → CLOSED
           если ошибка → OPEN
```

```go
import "github.com/sony/gobreaker"

cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "payment-service",
    MaxRequests: 3,          // в HALF-OPEN: max 3 запроса
    Interval:    10 * time.Second, // окно подсчёта ошибок
    Timeout:     30 * time.Second, // время в OPEN
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        return counts.ConsecutiveFailures > 5
    },
    OnStateChange: func(name string, from, to gobreaker.State) {
        log.Info("CB state changed", "from", from, "to", to)
    },
})

result, err := cb.Execute(func() (interface{}, error) {
    return paymentService.Process(ctx, payment)
})
if err == gobreaker.ErrOpenState {
    // Circuit open — fast fail
    return ErrServiceUnavailable
}
```

## CQRS (Command Query Responsibility Segregation)

Разделение операций чтения и записи:

```
Write side (Command):
  API → Command Handler → Write DB (PostgreSQL)
                       → Publish Event → Event Bus

Read side (Query):
  API → Query Handler → Read DB (ElasticSearch/Redis/PG read replica)
                        (денормализованные данные, оптимизированы для чтения)
```

```go
// Write side
type CreateOrderCommand struct {
    UserID   int64
    Products []ProductItem
}

type CommandHandler struct {
    db    *pgxpool.Pool
    kafka *kafka.Writer
}

func (h *CommandHandler) Handle(ctx context.Context, cmd CreateOrderCommand) error {
    order := createOrder(cmd)
    h.db.Exec(ctx, "INSERT INTO orders ...")
    h.kafka.WriteMessages(ctx, kafka.Message{
        Value: mustMarshal(OrderCreatedEvent{OrderID: order.ID}),
    })
    return nil
}

// Read side
type QueryHandler struct {
    elastic *elasticsearch.Client
}

func (h *QueryHandler) GetUserOrders(ctx context.Context, userID int64) ([]OrderView, error) {
    // Из ElasticSearch (быстрый поиск, агрегации)
    result, _ := h.elastic.Search(
        h.elastic.Search.WithIndex("orders"),
        h.elastic.Search.WithQuery(`{"term":{"user_id":` + strconv.FormatInt(userID, 10) + `}}`),
    )
    return parseOrderViews(result)
}
```

## Event Sourcing

Хранить не текущее состояние, а **поток событий**:

```
Обычно:          Event Sourcing:
┌──────────┐     ┌────────────────────────────────────────┐
│ orders   │     │ events                                  │
│ id: 42   │     │ OrderCreated   {id:42, products:[...]} │
│ status:  │     │ PaymentAdded   {orderId:42, amount:100}│
│  shipped │     │ OrderShipped   {orderId:42, tracking:X}│
└──────────┘     └────────────────────────────────────────┘

Текущее состояние = проигрывание всех событий
```

```go
type EventStore struct {
    db *pgxpool.Pool
}

func (s *EventStore) Append(ctx context.Context, streamID string, events []Event) error {
    tx, _ := s.db.Begin(ctx)
    for i, event := range events {
        _, err := tx.Exec(ctx,
            `INSERT INTO events (stream_id, sequence, type, data)
             VALUES ($1, (SELECT COALESCE(MAX(sequence), 0) + 1 FROM events WHERE stream_id = $1), $2, $3)`,
            streamID, event.Type, mustMarshal(event.Data),
        )
        if err != nil {
            tx.Rollback(ctx)
            return err
        }
    }
    return tx.Commit(ctx)
}

func (s *EventStore) Load(ctx context.Context, streamID string) ([]Event, error) {
    rows, _ := s.db.Query(ctx,
        `SELECT type, data, sequence FROM events WHERE stream_id = $1 ORDER BY sequence`,
        streamID)
    // ...
}

// Восстановить состояние из событий
func ReplayOrder(events []Event) *Order {
    order := &Order{}
    for _, e := range events {
        switch e.Type {
        case "OrderCreated":
            var data OrderCreatedData
            json.Unmarshal(e.Data, &data)
            order.ID = data.ID
            order.Products = data.Products
        case "OrderShipped":
            order.Status = "shipped"
        }
    }
    return order
}
```

## Service Discovery

Как сервисы находят друг друга:

```
Client-side discovery:
  Service A → Consul/etcd (где B?) → IP:port → B напрямую

Server-side discovery:
  Service A → Load Balancer → [B1, B2, B3] (LB знает адреса)
```

```go
// Consul SDK
import "github.com/hashicorp/consul/api"

// Регистрация
client, _ := api.NewDefaultClient()
client.Agent().ServiceRegister(&api.AgentServiceRegistration{
    ID:      "order-service-1",
    Name:    "order-service",
    Address: "10.0.0.1",
    Port:    8080,
    Check: &api.AgentServiceCheck{
        HTTP:     "http://10.0.0.1:8080/health",
        Interval: "10s",
    },
})

// Поиск
services, _, _ := client.Health().Service("payment-service", "", true, nil)
```

В Kubernetes — DNS-based service discovery:

```go
// "payment-service.payments-ns.svc.cluster.local:8080"
conn, _ := grpc.Dial("payment-service.payments-ns.svc.cluster.local:50051", grpc.WithInsecure())
```

## Sidecar Pattern

Вспомогательный контейнер рядом с основным:

```
Pod:
  ┌─────────────────────────────────────┐
  │  [App Container]  [Sidecar]         │
  │   Go Service  →  Envoy Proxy        │
  │                  (mTLS, metrics,    │
  │                   circuit breaking) │
  └─────────────────────────────────────┘
```

**Примеры:**
- **Istio/Envoy** — service mesh (mTLS, traffic shaping, observability).
- **Fluentd/Filebeat** — сбор логов.
- **OPA** — policy enforcement.

## Strangler Fig Pattern

Постепенная замена монолита микросервисами:

```
Phase 1:        Phase 2:        Phase 3:
Client          Client          Client
  │               │               │
  ▼               ▼               ▼
Monolith        Proxy           Proxy
                  │              │
                  ├─ Monolith   ├─ Order Service (new)
                  └─ Orders ──► └─ Payment Service (new)
                    (intercepted) └─ Tiny Monolith (old)
```

```go
// Proxy перехватывает и перенаправляет
func proxy(w http.ResponseWriter, r *http.Request) {
    if strings.HasPrefix(r.URL.Path, "/orders") {
        // → новый Orders микросервис
        reverseProxy(orderServiceURL, w, r)
        return
    }
    // → старый монолит
    reverseProxy(monolithURL, w, r)
}
```

## Bulkhead Pattern

Изоляция ресурсов для предотвращения каскадных отказов:

```
Thread pool для каждого downstream сервиса:
  Payment pool (10 threads) → если переполнен: fail fast, не трогает Orders pool
  Orders pool (20 threads)
  Catalog pool (5 threads)
```

```go
// Ограничение параллельных запросов через семафор
paymentSem := semaphore.NewWeighted(10) // максимум 10 параллельных

func callPayment(ctx context.Context, req PaymentRequest) (*PaymentResponse, error) {
    if !paymentSem.TryAcquire(1) {
        return nil, ErrPaymentBulkheadFull // fast fail
    }
    defer paymentSem.Release(1)
    return paymentClient.Process(ctx, req)
}
```

## Retry с Exponential Backoff

```go
func withRetry(ctx context.Context, maxAttempts int, fn func() error) error {
    var err error
    for attempt := 0; attempt < maxAttempts; attempt++ {
        if err = fn(); err == nil {
            return nil
        }
        if !isRetryable(err) {
            return err
        }
        // Exponential backoff с jitter
        wait := time.Duration(math.Pow(2, float64(attempt))) * 100 * time.Millisecond
        jitter := time.Duration(rand.Int63n(int64(wait / 2)))
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(wait + jitter):
        }
    }
    return fmt.Errorf("max attempts reached: %w", err)
}
```

## Сравнение паттернов Saga

| | Choreography | Orchestration |
|--|-------------|--------------|
| Управление | Распределённое | Централизованное |
| Связанность | Слабая | Выше |
| Видимость | Сложно понять flow | Явный flow |
| Отладка | Сложнее | Проще |
| Единая точка отказа | Нет | Да (оркестратор) |
| Use case | Простые flow | Сложные, с компенсациями |

## Что спрашивают на собеседованиях

1. **Saga vs распределённые транзакции?** → Saga: компенсирующие транзакции при ошибке. Нет двухфазного commit. Choreography или Orchestration.
2. **Circuit Breaker — зачем?** → Предотвращает каскадные отказы. Если сервис упал → fast fail вместо бесконечных ожиданий.
3. **CQRS — зачем разделять?** → Чтение и запись имеют разные требования. Запись: consistency. Чтение: скорость, денормализация.
4. **Database per Service — проблемы?** → Нельзя JOIN, нужен Saga для транзакций, eventual consistency.
5. **Event Sourcing — преимущества?** → Полная история изменений, возможность replay событий, аудит, time-travel debugging.
6. **API Gateway vs Service Mesh?** → Gateway: edge, клиент → сервисы. Mesh: east-west трафик между сервисами (Istio/Envoy sidecar).
