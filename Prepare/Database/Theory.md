# Теория баз данных

Полезные ссылки:
- [PostgreSQL docs](https://www.postgresql.org/docs/)
- [Use the Index, Luke](https://use-the-index-luke.com/)
- [Designing Data-Intensive Applications (Kleppmann)](https://dataintensive.net/)
- [CAP theorem explained](https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html)

---

## ACID

Свойства транзакций, гарантирующие надёжность:

| Свойство | Суть |
|----------|------|
| **A**tomicity | Всё или ничего. Если что-то упало — весь откат. |
| **C**onsistency | БД переходит из одного корректного состояния в другое. |
| **I**solation | Транзакции изолированы друг от друга (как будто последовательны). |
| **D**urability | После COMMIT данные сохранены, даже при crash. |

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT; -- или ROLLBACK
```

## Уровни изоляции транзакций

Стандарт SQL определяет 4 уровня (от менее строгого к более строгому):

| Уровень | Dirty Read | Non-repeatable Read | Phantom Read |
|---------|-----------|---------------------|--------------|
| Read Uncommitted | Возможно | Возможно | Возможно |
| **Read Committed** | Нет | Возможно | Возможно |
| Repeatable Read | Нет | Нет | Возможно |
| Serializable | Нет | Нет | Нет |

> PostgreSQL по умолчанию: **Read Committed**. MySQL/InnoDB: Repeatable Read.

### Аномалии

**Dirty Read** — читаем незафиксированные данные другой транзакции:
```
T1: UPDATE users SET name = 'Alice' -- не зафиксировано
T2: SELECT name FROM users → 'Alice'  -- dirty read!
T1: ROLLBACK
```

**Non-repeatable Read** — повторное чтение одной строки даёт разный результат:
```
T1: SELECT balance → 100
T2: UPDATE accounts SET balance = 200; COMMIT
T1: SELECT balance → 200  -- другой результат!
```

**Phantom Read** — повторный запрос возвращает разное количество строк:
```
T1: SELECT COUNT(*) WHERE age > 18 → 5
T2: INSERT INTO users (age) VALUES (20); COMMIT
T1: SELECT COUNT(*) WHERE age > 18 → 6  -- фантом!
```

### Как PostgreSQL реализует изоляцию

PostgreSQL использует **MVCC** (Multi-Version Concurrency Control):
- Каждая запись имеет `xmin` (создана транзакцией) и `xmax` (удалена/обновлена).
- Читатели видят "снимок" данных на момент начала транзакции.
- Нет конкуренции читателей и писателей (не блокируют друг друга).

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
-- ваша логика
COMMIT;
```

## Блокировки

### Оптимистичный контроль конкуренции (Optimistic Locking)

Предполагает, что конфликты редки. Блокировки не захватываются при чтении:

```sql
-- При обновлении проверяем версию
UPDATE orders SET status = 'paid', version = version + 1
WHERE id = 42 AND version = 3;
-- Если rowsAffected = 0 → конфликт, повторить
```

```go
// Go ORM (GORM)
type Order struct {
    ID      uint
    Status  string
    Version int `gorm:"version"`
}
// GORM автоматически добавит AND version = ? при Update
```

**Плюсы:** нет блокировок → высокая пропускная способность.  
**Минусы:** нужно обрабатывать конфликты в приложении.

### Пессимистичный контроль (Pessimistic Locking)

Блокировка при чтении:

```sql
-- Блокирует строку до конца транзакции
SELECT * FROM orders WHERE id = 42 FOR UPDATE;
-- Другие транзакции ждут

-- Не ждать, вернуть ошибку
SELECT * FROM orders WHERE id = 42 FOR UPDATE NOWAIT;

-- Пропустить заблокированные строки
SELECT * FROM orders WHERE status = 'pending' FOR UPDATE SKIP LOCKED;
-- Применение: job queues, воркеры
```

**Плюсы:** гарантия эксклюзивности, проще в логике.  
**Минусы:** deadlock риск, конкуренция при высокой нагрузке.

## Индексы

**Индекс** — дополнительная структура данных для ускорения поиска.

### B-Tree (default)

Сбалансированное дерево поиска. Поддерживает `=`, `<`, `>`, `BETWEEN`, `LIKE 'prefix%'`.

```
Корень (root)
├── Узел [10, 20]
│   ├── [1, 5, 7, 9]
│   └── [11, 15, 18]
└── Узел [30, 50]
    ├── [21, 25, 28]
    └── [31, 40, 45]
```

**Операции:** O(log n). **Подходит для:** большинство случаев.

### Hash Index

Хеш-таблица. Только `=`. Быстрее B-tree для точных совпадений.

### GIN (Generalized Inverted Index)

Для составных типов: массивы, JSONB, full-text search, tsquery.

```sql
CREATE INDEX ON articles USING GIN(tags);
SELECT * FROM articles WHERE tags @> '{go, database}';

CREATE INDEX ON documents USING GIN(content_tsvector);
SELECT * FROM documents WHERE content_tsvector @@ to_tsquery('go & database');
```

### GiST (Generalized Search Tree)

Для геоданных, диапазонов, геометрии (PostGIS).

```sql
CREATE INDEX ON locations USING GIST(coords);
SELECT * FROM locations WHERE coords <-> point(55.7, 37.6) < 10; -- в 10 км
```

### BRIN (Block Range Index)

Для очень больших таблиц с монотонными данными (временные ряды, логи).

```sql
CREATE INDEX ON events USING BRIN(created_at);
-- Маленький индекс, грубая фильтрация по диапазону блоков
```

### Составные индексы (Composite Index)

```sql
-- Порядок колонок важен!
CREATE INDEX ON orders (user_id, created_at);

-- Эффективно: WHERE user_id = 5 AND created_at > '2024-01'
-- Эффективно: WHERE user_id = 5
-- НЕ эффективно: WHERE created_at > '2024-01' (без user_id)
```

**Правило left-prefix:** составной индекс `(a, b, c)` используется для запросов с `a`, `a + b`, `a + b + c`, но не с `b` или `c` отдельно.

### Частичный индекс (Partial Index)

```sql
-- Только активные заказы (маленький индекс)
CREATE INDEX ON orders (user_id) WHERE status = 'active';
```

### Функциональный индекс

```sql
-- Поиск без учёта регистра
CREATE INDEX ON users (lower(email));
SELECT * FROM users WHERE lower(email) = lower('User@Example.com');
```

## CAP теорема

Распределённая система может гарантировать только 2 из 3:

| Свойство | Описание |
|----------|---------|
| **C**onsistency | Все узлы видят одни данные |
| **A**vailability | Система всегда отвечает |
| **P**artition tolerance | Система работает при сетевых разделениях |

> **Partition** всегда будет. Поэтому выбор между C и A.

- **CP** системы: PostgreSQL, MongoDB, ZooKeeper — выбирают согласованность.
- **AP** системы: Cassandra, CouchDB, DynamoDB — выбирают доступность.

**PACELC** (расширение CAP): учитывает и нормальный режим (без partitions):
- Latency vs Consistency при нормальной работе.

## Нормализация

**1NF** — атомарные значения, нет повторяющихся групп.  
**2NF** — 1NF + нет частичных зависимостей от составного ключа.  
**3NF** — 2NF + нет транзитивных зависимостей.  
**BCNF** — строже 3NF: каждая детерминанта — кандидатный ключ.

Денормализация оправдана для производительности (но требует поддержки консистентности).

## SQL vs NoSQL

| | SQL (реляционные) | NoSQL |
|--|-------------------|-------|
| Структура | Схема (жёсткая) | Гибкая (документы, граф, KV) |
| Запросы | SQL (JOIN, агрегации) | Ограниченный query language |
| Транзакции | ACID | BASE (eventually consistent) |
| Масштабирование | Вертикальное (сложнее горизонтальное) | Горизонтальное |
| Примеры | PostgreSQL, MySQL | MongoDB, Redis, Cassandra, ClickHouse |
| Применение | Бизнес-данные, финансы | Сессии, кэш, аналитика, документы |

## Шардирование и репликация

### Репликация
```
Primary → Replica1, Replica2
                                
Синхронная: запись подтверждается когда реплики записали (сильная консистентность)
Асинхронная: запись подтверждается Primary без ожидания реплик (быстрее, но lag)
```

### Шардирование (Horizontal Partitioning)
```
Shard 1: users с ID 1-1000000
Shard 2: users с ID 1000001-2000000
```

**Проблемы:** cross-shard JOIN, перебалансировка, hotspot keys.

**Стратегии:** Hash sharding, Range sharding, Directory-based.

## Что спрашивают на собеседованиях

1. **Что такое ACID?** → Atomicity, Consistency, Isolation, Durability. Транзакция выполняется полностью или откатывается.
2. **Уровни изоляции?** → Read Uncommitted, Read Committed (PG default), Repeatable Read, Serializable.
3. **Что такое Dirty Read?** → Чтение незафиксированных данных другой транзакции.
4. **B-tree vs Hash index?** → B-tree: диапазоны и сортировка. Hash: только точное равенство, быстрее.
5. **FOR UPDATE vs FOR SHARE?** → FOR UPDATE: эксклюзивная блокировка (писатель). FOR SHARE: разделяемая (читатель, не даёт писать).
6. **MVCC?** → Multi-Version Concurrency Control: читатели и писатели не блокируют друг друга, каждая транзакция видит снимок данных.
7. **CAP теорема?** → Consistency, Availability, Partition tolerance — только 2 из 3. Partition неизбежен → выбор C vs A.
