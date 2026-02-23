# PostgreSQL

Полезные ссылки:
- [PostgreSQL 17 изнутри](https://edu.postgrespro.ru/postgresql_internals-17.pdf)
- [Книги на postgrespro.ru](https://postgrespro.ru/education/books/internals)
- [Use the Index, Luke — PG edition](https://use-the-index-luke.com/sql/table-of-contents)
- [EXPLAIN Depesz](https://explain.depesz.com/)

---

## Индексы

### Типы индексов PostgreSQL

| Тип | Операторы | Применение |
|-----|----------|-----------|
| **B-tree** | `=`, `<`, `>`, `BETWEEN`, `LIKE 'x%'` | По умолчанию, большинство случаев |
| **Hash** | `=` | Точное совпадение, чуть быстрее B-tree |
| **GIN** | `@>`, `@@`, `&&` | JSONB, массивы, full-text search |
| **GiST** | геометрия, диапазоны | PostGIS, `tsrange` |
| **BRIN** | диапазоны блоков | Временные ряды, огромные таблицы |
| **SP-GiST** | непересекающиеся данные | IP адреса, точки |

```sql
-- Создание индексов
CREATE INDEX ON users (email);                         -- B-tree
CREATE INDEX ON orders (user_id, created_at DESC);     -- составной
CREATE INDEX ON products USING GIN (attributes);       -- JSONB
CREATE INDEX ON events USING BRIN (created_at);        -- временные ряды
CREATE INDEX ON users (lower(email));                  -- функциональный
CREATE INDEX ON orders (id) WHERE status = 'active';   -- частичный

-- Concurrent (без блокировки таблицы)
CREATE INDEX CONCURRENTLY ON large_table (column);

-- Посмотреть индексы таблицы
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'orders';
```

### B-tree внутри

```
Leaf pages (листы) связаны в двусвязный список → эффективный range scan
Страница (block = 8 КБ) хранит ключи и указатели на heap (строки таблицы)

Index Scan: индекс → heap = random I/O (медленно для большого %%)
Index Only Scan: данные берутся из indpex без обращения к heap (нужен visibility map)
Bitmap Index Scan: собирает битмап страниц → затем batch-читает heap
```

## EXPLAIN и EXPLAIN ANALYZE

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 42;
-- Показывает план запроса (оценки)

EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42;
-- Выполняет запрос + реальное время и rows

EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT ...;
-- Максимально подробно: буферы (hit/read), JSON формат
```

### Как читать EXPLAIN

```
Seq Scan on orders  (cost=0.00..450.00 rows=100 width=64) (actual time=0.1..15.2 rows=98)
                          ↑        ↑      ↑         ↑               ↑       ↑      ↑
                       start    total  оценка    ширина          start   total   реальные
                       cost    cost    строк    строки          time    time     строки

Seq Scan — полное сканирование (нет индекса или выбирается планировщиком)
Index Scan — использует индекс
Index Only Scan — только индекс (heap не читается)
Bitmap Heap Scan — batch read через bitmap
Hash Join — соединение через хеш-таблицу
Nested Loop — вложенный цикл
```

### Красные флаги в EXPLAIN

```sql
-- Плохо: estimates << actual (устаревшая статистика)
--   (cost=...rows=10...)  (actual time=...rows=10000...)
--   Решение: ANALYZE table_name;

-- Плохо: Seq Scan на большой таблице с фильтром
--   Решение: добавить индекс

-- Плохо: sort + no index
--   Sort  (cost=1000... rows=...)
--   Решение: CREATE INDEX ON t(col DESC);

-- Плохо: cost=0..12345 при rows=1 → PG переоценил
--   Решение: ANALYZE или pg_stats

-- Хорошо:
--   cost оценка ≈ actual время
--   rows оценка ≈ реальные rows
```

### Включить slow query log

```sql
-- postgresql.conf
log_min_duration_statement = 100  -- логировать запросы > 100ms
log_statement = 'none'            -- или 'all', 'ddl', 'mod'

-- pg_stat_statements (расширение)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
SELECT query, calls, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

## MVCC и VACUUM

PostgreSQL использует MVCC (Multi-Version Concurrency Control):
- Старые версии строк (dead tuples) накапливаются.
- **VACUUM** очищает мёртвые туплы, освобождает место.
- **ANALYZE** обновляет статистику для планировщика.
- **autovacuum** делает это автоматически.

```sql
-- Ручной VACUUM
VACUUM orders;               -- мягкий (не освобождает место ОС)
VACUUM FULL orders;          -- полная (блокирует, перезаписывает)
VACUUM ANALYZE orders;       -- vacuum + analyze
VACUUM (VERBOSE) orders;     -- с деталями

-- Статистика по dead tuples
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'orders';
```

## Транзакции и изоляция

```sql
-- Уровень изоляции по умолчанию: READ COMMITTED
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- ...
COMMIT;

-- Savepoints (вложенные транзакции)
BEGIN;
INSERT INTO orders ...;
SAVEPOINT before_payment;
UPDATE payments ...;
-- Ошибка? → ROLLBACK TO SAVEPOINT before_payment;
COMMIT;
```

## Блокировки

```sql
-- Эксклюзивная блокировка строки
SELECT * FROM orders WHERE id = 42 FOR UPDATE;

-- SKIP LOCKED — воркер берёт следующую доступную строку
SELECT * FROM job_queue WHERE status = 'pending'
ORDER BY id LIMIT 1
FOR UPDATE SKIP LOCKED;
-- Идеально для параллельных воркеров

-- NOWAIT — вернуть ошибку немедленно
SELECT * FROM orders WHERE id = 42 FOR UPDATE NOWAIT;

-- Посмотреть блокировки
SELECT pid, relation::regclass, mode, granted
FROM pg_locks
WHERE relation IS NOT NULL;

-- Убить блокирующий запрос
SELECT pg_cancel_backend(pid);  -- мягко (cancel query)
SELECT pg_terminate_backend(pid); -- жёстко (kill connection)
```

## Connection Pooling

PostgreSQL создаёт отдельный OS process на каждое соединение. **Соединение дорогое!**

**PgBouncer** — connection pooler:

```ini
# pgbouncer.ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction  # session | transaction | statement
max_client_conn = 1000
default_pool_size = 20
```

**Режимы:**
- `session` — соединение на время сессии. Поддерживает все PG фичи.
- `transaction` — соединение на время транзакции. Не поддерживает LISTEN/NOTIFY.
- `statement` — на каждый запрос. Не поддерживает транзакции.

```go
// Go: database/sql с connection pool
db, _ := sql.Open("postgres", dsn)
db.SetMaxOpenConns(25)      // максимум открытых соединений
db.SetMaxIdleConns(10)      // максимум idle соединений
db.SetConnMaxLifetime(5 * time.Minute) // время жизни соединения
```

## Партиционирование

```sql
-- Range partitioning по дате (PG 10+)
CREATE TABLE events (
    id       BIGSERIAL,
    type     TEXT,
    created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2024 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
CREATE TABLE events_2025 PARTITION OF events
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

-- Hash partitioning
CREATE TABLE users (id BIGINT, name TEXT)
PARTITION BY HASH (id);
CREATE TABLE users_0 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 0);
-- ...

-- Индексы создаются отдельно для каждой партиции
-- PG 11+: CREATE INDEX ON ONLY events (created_at) → автоматически для партиций
```

## Полнотекстовый поиск

```sql
-- tsvector — вектор документа, tsquery — поисковый запрос
SELECT to_tsvector('russian', 'PostgreSQL мощная база данных') @@
       to_tsquery('russian', 'база & данных');

-- Индекс для FTS
ALTER TABLE articles ADD COLUMN fts tsvector
    GENERATED ALWAYS AS (to_tsvector('english', title || ' ' || body)) STORED;

CREATE INDEX ON articles USING GIN(fts);

SELECT title FROM articles
WHERE fts @@ to_tsquery('english', 'database & performance')
ORDER BY ts_rank(fts, to_tsquery('english', 'database & performance')) DESC;
```

## JSONB

```sql
-- JSONB — бинарное хранение JSON (быстрее JSON для запросов)
CREATE TABLE products (
    id   SERIAL PRIMARY KEY,
    data JSONB
);

INSERT INTO products VALUES (1, '{"name": "Widget", "price": 9.99, "tags": ["sale"]}');

-- Операторы
SELECT data->>'name' FROM products;          -- текстовое значение
SELECT data->'tags'->0 FROM products;        -- вложенный элемент
SELECT * FROM products WHERE data @> '{"name": "Widget"}'; -- containment
SELECT * FROM products WHERE data ? 'tags';  -- ключ существует

-- Индекс по JSONB
CREATE INDEX ON products USING GIN(data);
CREATE INDEX ON products ((data->>'price'));  -- по конкретному ключу
```

## Оптимизация производительности

```sql
-- Посмотреть медленные запросы (pg_stat_statements)
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 10;

-- Посмотреть размеры таблиц и индексов
SELECT
    relname AS table,
    pg_size_pretty(pg_total_relation_size(relid)) AS total,
    pg_size_pretty(pg_relation_size(relid)) AS table_size,
    pg_size_pretty(pg_total_relation_size(relid) - pg_relation_size(relid)) AS index_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- Cache hit ratio (должен быть > 99%)
SELECT
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) AS cache_hit_ratio
FROM pg_statio_user_tables;

-- Неиспользуемые индексы
SELECT schemaname, relname, indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY relname;
```

## PostgreSQL в Go

```go
import (
    "database/sql"
    _ "github.com/lib/pq"
    "github.com/jmoiron/sqlx"
    // или pgx (рекомендуется)
    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"
)

// pgx — нативный драйвер, быстрее database/sql
pool, err := pgxpool.New(ctx, "postgres://user:pass@localhost/mydb")
if err != nil {
    log.Fatal(err)
}
defer pool.Close()

// Запрос
var id int
var name string
err = pool.QueryRow(ctx, "SELECT id, name FROM users WHERE id = $1", 42).Scan(&id, &name)

// Batch запросы
batch := &pgx.Batch{}
batch.Queue("INSERT INTO events (type) VALUES ($1)", "login")
batch.Queue("UPDATE users SET last_login = now() WHERE id = $1", userID)
br := pool.SendBatch(ctx, batch)
_, err = br.Exec()
br.Close()

// Транзакция
tx, err := pool.Begin(ctx)
defer tx.Rollback(ctx) // no-op если уже commit
_, err = tx.Exec(ctx, "UPDATE ...")
tx.Commit(ctx)

// LISTEN/NOTIFY
conn, _ := pool.Acquire(ctx)
defer conn.Release()
conn.Exec(ctx, "LISTEN events_channel")
notification, _ := conn.Conn().WaitForNotification(ctx)
fmt.Println(notification.Payload)
```

## Что спрашивают на собеседованиях

1. **Типы индексов в PostgreSQL?** → B-tree (default), Hash, GIN (JSONB/массивы), GiST (гео), BRIN (временные ряды).
2. **Как читать EXPLAIN ANALYZE?** → cost=start..total rows=estimated width=bytes | actual time=start..total rows=real.
3. **Что такое MVCC?** → Каждая строка имеет xmin/xmax (транзакции). Читатели видят снимок. Dead tuples очищает VACUUM.
4. **Зачем PgBouncer?** → PG процесс на соединение дорогой. Pooler переиспользует соединения: 1000 клиентов → 20 реальных соединений.
5. **FOR UPDATE SKIP LOCKED?** → Параллельные воркеры берут следующую доступную задачу из очереди без deadlock.
6. **Когда добавлять индекс?** → Медленный запрос + Seq Scan на большой таблице + высокая селективность запроса (< 10% строк).
7. **GIN vs B-tree для JSONB?** → GIN для `@>` (containment), `?` (key exists). B-tree для конкретного поля с `=`, `<`, `>`.
