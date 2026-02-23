# Apache Kafka

Полезные ссылки:
- [Kafka Documentation](https://kafka.apache.org/documentation/)
- [Designing Data-Intensive Applications — Chapter 11](https://dataintensive.net/)
- [segmentio/kafka-go](https://github.com/segmentio/kafka-go)
- [confluent-kafka-go](https://github.com/confluentinc/confluent-kafka-go)

---

## Что такое Kafka?

**Apache Kafka** — распределённый журнал сообщений (distributed commit log). Изначально создан в LinkedIn для обработки миллиардов событий в день.

**Применение:**
- Event streaming (события в реальном времени).
- Messaging (замена RabbitMQ для высокой нагрузки).
- Change Data Capture (CDC) — репликация изменений из БД.
- Activity tracking, metrics aggregation, log aggregation.

## Архитектура

```
┌───────────────────────────────────────────────────────────┐
│                    Kafka Cluster                           │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐                 │
│  │ Broker 1│   │ Broker 2│   │ Broker 3│                 │
│  │         │   │         │   │         │                 │
│  │Topic A  │   │Topic A  │   │Topic A  │                 │
│  │Part 0 L │   │Part 1 L │   │Part 2 L │  L=Leader       │
│  │Part 1 F │   │Part 0 F │   │Part 1 F │  F=Follower     │
│  └─────────┘   └─────────┘   └─────────┘                 │
└───────────────────────────────────────────────────────────┘
         ▲                              │
    Producers                      Consumers
    (write to Leader)          (read from Leader)
```

## Ключевые концепции

### Topic

Логический канал для сообщений. Похож на таблицу в БД.

```bash
# Создать топик
kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic orders --partitions 6 --replication-factor 3

# Посмотреть топики
kafka-topics.sh --bootstrap-server localhost:9092 --list
kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic orders
```

### Partition

Топик разбит на **партиции** — упорядоченные, неизменяемые последовательности сообщений.

```
Topic "orders":
  Partition 0: [msg0][msg1][msg2][msg3]...
  Partition 1: [msg0][msg1][msg2]...
  Partition 2: [msg0][msg1][msg2][msg3][msg4]...
                                            ↑
                                          offset
```

- **Порядок** гарантируется только внутри партиции.
- Ключ сообщения определяет партицию: `partition = hash(key) % numPartitions`.
- Сообщения без ключа → round-robin по партициям.

### Offset

**Offset** — монотонно возрастающий номер сообщения внутри партиции. Consumer сам управляет offsets (хранятся в `__consumer_offsets`).

```
Partition 0: [0][1][2][3][4]...
                        ↑
                 Consumer committed offset=4
                 Следующее сообщение к чтению: offset=4
```

### Replication

```
Leader Partition  → принимает записи
Follower Partition → реплицирует от Leader (синхронно для ISR)

ISR (In-Sync Replicas) — реплики в синхроне с Leader
```

```bash
# Конфигурация надёжности
acks = all       # ждать подтверждения от всех ISR реплик
min.insync.replicas = 2  # минимум 2 реплики должны быть в ISR
replication.factor = 3   # 3 копии данных
```

### Consumer Group

**Consumer Group** — группа consumers, делящих нагрузку. Каждая партиция читается только одним consumer в группе.

```
Topic: 6 partitions
Consumer Group (3 consumers):
  Consumer A → Partition 0, 1
  Consumer B → Partition 2, 3
  Consumer C → Partition 4, 5

Parallelism = min(partitions, consumers_in_group)
```

**Rebalancing** — перераспределение партиций при добавлении/удалении consumer. Во время rebalance группа не читает (stop-the-world).

Стратегии rebalance:
- `range` — по умолчанию
- `round-robin` — равномерно
- `sticky` — минимальное перемещение партиций
- `cooperative-sticky` (Kafka 2.4+) — постепенный rebalance без stop-the-world

## Виды гарантий доставки

### At-Most-Once (не более одного раза)

Сообщение может быть потеряно, но никогда не дублируется.

```go
// Consumer: commit перед обработкой
msg := reader.ReadMessage(ctx) // читаем
reader.CommitMessages(ctx, msg) // коммитим ДО обработки
processMessage(msg) // если упадёт — сообщение потеряно
```

### At-Least-Once (не менее одного раза) — по умолчанию

Сообщение будет доставлено, но возможны дубликаты.

```go
// Consumer: commit после обработки
msg := reader.ReadMessage(ctx)
processMessage(msg)             // если упадёт — сообщение перечитается
reader.CommitMessages(ctx, msg) // коммитим ПОСЛЕ обработки
```

```go
// Producer: retries при ошибке
w := &kafka.Writer{
    Addr:     kafka.TCP("localhost:9092"),
    Topic:    "orders",
    Balancer: &kafka.LeastBytes{},
    MaxAttempts: 3, // повтор при ошибке
}
```

### Exactly-Once (ровно один раз)

Самый сложный режим. Нужен идемпотентный producer + транзакции.

```go
// Идемпотентный producer (Kafka >= 0.11)
// Дедупликация на стороне Kafka по (producerID, sequence number)
w := &kafka.Writer{
    Addr:         kafka.TCP("localhost:9092"),
    Topic:        "orders",
    RequiredAcks: kafka.RequireAll,
    // Включить идемпотентность (kafka-go не поддерживает полностью,
    // используйте confluent-kafka-go для транзакций)
}
```

## Транзакционный Producer

```go
// confluent-kafka-go — поддерживает транзакции
p, _ := kafka.NewProducer(&kafka.ConfigMap{
    "bootstrap.servers":        "localhost:9092",
    "transactional.id":         "my-producer-1",   // уникальный per producer instance
    "enable.idempotence":       true,
    "acks":                     "all",
})

p.InitTransactions(ctx)

p.BeginTransaction()
p.Produce(&kafka.Message{
    TopicPartition: kafka.TopicPartition{Topic: &topic, Partition: kafka.PartitionAny},
    Value:          []byte("order created"),
}, nil)
p.CommitTransaction(ctx)   // atomic commit
// или p.AbortTransaction(ctx)
```

## Паттерн Inbox/Outbox

Решает проблему: как атомарно записать в БД и отправить в Kafka?

### Outbox Pattern

```
1. Transactional write: INSERT INTO orders + INSERT INTO outbox
2. Outbox poller читает outbox и публикует в Kafka
3. После подтверждения Kafka → DELETE FROM outbox
```

```go
// В одной транзакции
tx.Exec("INSERT INTO orders (id, status) VALUES ($1, $2)", orderID, "created")
tx.Exec("INSERT INTO outbox (event_type, payload) VALUES ($1, $2)",
    "order.created", mustMarshal(event))
tx.Commit()

// Поллер (отдельная горутина)
for {
    rows := db.Query("SELECT id, event_type, payload FROM outbox ORDER BY id LIMIT 100")
    for rows.Next() {
        // publish to Kafka
        // delete from outbox on success
    }
    time.Sleep(100 * time.Millisecond)
}
```

Альтернатива: **Debezium** (CDC) — читает WAL PostgreSQL и публикует изменения в Kafka без изменения приложения.

### Inbox Pattern

```
1. Consumer читает из Kafka
2. INSERT INTO inbox (message_id, payload) -- дедупликация по message_id
3. Обработчик читает inbox и обрабатывает
```

## Consumer Lag

**Consumer Lag** = `latest offset - committed offset` для каждой партиции.

```bash
# Посмотреть lag
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group my-group

# GROUP  TOPIC    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# g1     orders   0          1000            1050            50
# g1     orders   1          2000            2001            1
```

```go
// Мониторинг lag через Prometheus + kafka-exporter
// или сами публикуем метрику
gauge := prometheus.NewGaugeVec(prometheus.GaugeOpts{
    Name: "kafka_consumer_lag",
}, []string{"topic", "partition"})
```

## Kafka в Go

### segmentio/kafka-go (рекомендуется для простых случаев)

```go
import "github.com/segmentio/kafka-go"

// Producer
writer := &kafka.Writer{
    Addr:         kafka.TCP("localhost:9092"),
    Topic:        "orders",
    Balancer:     &kafka.Hash{}, // по ключу — гарантия порядка
    RequiredAcks: kafka.RequireAll,
    MaxAttempts:  3,
}
defer writer.Close()

err := writer.WriteMessages(ctx,
    kafka.Message{
        Key:   []byte("order-123"),
        Value: mustMarshal(order),
        Headers: []kafka.Header{
            {Key: "trace-id", Value: []byte(traceID)},
        },
    },
)

// Consumer
reader := kafka.NewReader(kafka.ReaderConfig{
    Brokers:        []string{"localhost:9092"},
    Topic:          "orders",
    GroupID:        "orders-processor",
    MinBytes:       10e3,         // 10 КБ
    MaxBytes:       10e6,         // 10 МБ
    CommitInterval: time.Second,  // автоматический commit каждую секунду
    // StartOffset: kafka.FirstOffset, // с начала (FirstOffset) или только новые (LastOffset)
})
defer reader.Close()

for {
    msg, err := reader.ReadMessage(ctx)
    if err != nil {
        if errors.Is(err, context.Canceled) {
            return
        }
        log.Println("error reading:", err)
        continue
    }
    if err := processOrder(msg.Value); err != nil {
        log.Println("processing error:", err)
        // at-least-once: не коммитим → перечитаем
        continue
    }
    // CommitInterval уже позаботится о коммите
}
```

### Конфигурация для production

```go
writer := &kafka.Writer{
    Addr:         kafka.TCP("kafka1:9092", "kafka2:9092", "kafka3:9092"),
    Topic:        "orders",
    Balancer:     &kafka.Hash{},
    RequiredAcks: kafka.RequireAll,         // durability
    MaxAttempts:  10,
    BatchSize:    100,                      // батчинг для throughput
    BatchTimeout: 10 * time.Millisecond,
    Async:        false,                    // синхронный (надёжнее)
    Compression:  kafka.Snappy,             // сжатие
}
```

## Сравнение Kafka vs RabbitMQ

| | Kafka | RabbitMQ |
|--|-------|---------|
| Модель | Pull-based (consumer тянет) | Push-based (broker толкает) |
| Хранение | Сохраняет (retention period) | Удаляет после ACK |
| Порядок | В пределах партиции | Нет |
| Throughput | Миллионы/сек | Тысячи/сек |
| Replay | Да (по offset) | Нет |
| Сложность | Выше | Ниже |
| Use case | Event streaming, CDC, high throughput | Task queues, простые сценарии |

## Что спрашивают на собеседованиях

1. **Что такое Kafka partition?** → Упорядоченная последовательность сообщений внутри топика. Единица параллелизма.
2. **Чем ограничен параллелизм Consumer Group?** → Числом партиций — не более 1 consumer на партицию.
3. **At-least-once vs exactly-once?** → At-least: коммит после обработки, возможны дубликаты при restart. Exactly-once: идемпотентный producer + транзакции Kafka.
4. **Что такое Consumer Lag?** → Отставание consumer от конца лога. Метрика для мониторинга производительности.
5. **Outbox pattern — зачем?** → Атомарная запись в БД и отправка в Kafka без двухфазного commit; fallback через поллер.
6. **Rebalancing — проблема?** → При rebalance группа не читает (stop-the-world). Решение: cooperative-sticky assignor (постепенный rebalance).
7. **Когда Kafka, когда RabbitMQ?** → Kafka: event streaming, replay, высокий throughput. RabbitMQ: простые task queues, routing, dead-letter queues.
