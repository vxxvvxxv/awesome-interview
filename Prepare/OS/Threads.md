# Потоки (Threads) и процессы

Полезные ссылки:
- [Linux Kernel: task_struct](https://elixir.bootlin.com/linux/latest/source/include/linux/sched.h)
- [Go runtime: scheduler](https://go.dev/src/runtime/proc.go)
- [pthreads manual](https://man7.org/linux/man-pages/man7/pthreads.7.html)

---

## Процесс vs Поток

| | Процесс | Поток |
|--|---------|-------|
| Память | Изолированное адресное пространство | Разделяют память процесса |
| Переключение | Тяжёлое (page tables, TLB flush) | Лёгкое (только регистры и стек) |
| Создание | `fork()` — ~1ms | `pthread_create()` — ~10мкс |
| Коммуникация | IPC (pipe, socket, shm) | Разделяемая память (+ синхронизация) |
| Изоляция | Полная | Нет (ошибка в одном потоке роняет всё) |

## Что такое поток?

**Поток** — наименьшая единица выполнения внутри процесса. Каждый процесс может иметь несколько потоков. Потоки разделяют:
- Адресное пространство (heap, code, data).
- Открытые файловые дескрипторы.
- Сигналы.

Но имеют **собственные**:
- Стек вызовов (обычно 1-8 МБ в Linux).
- Регистры процессора (включая Program Counter).
- Состояние выполнения (TLS — Thread Local Storage).

## Как работает поток?

1. **Создание** — ОС выделяет стек (1-8 МБ по умолчанию), настраивает `task_struct` в ядре.
2. **Планирование** — планировщик ОС (CFS в Linux) распределяет CPU time между потоками (preemptive).
3. **Переключение контекста** — CPU сохраняет регистры текущего потока, загружает регистры следующего (~1-10 мкс).
4. **Синхронизация** — мьютексы, семафоры, condition variables для доступа к общим ресурсам.
5. **Завершение** — `pthread_join()` ожидает завершения, освобождает ресурсы.

## Почему потоки "дорогие"?

- **Память** — 1 поток = 1-8 МБ стека. 10 000 потоков = 10-80 ГБ только под стеки.
- **Создание/удаление** — системные вызовы (`clone()` в Linux).
- **Переключение контекста** — ~1-10 мкс + invalidation TLB cache (косвенные расходы).
- **Лимиты ОС** — `ulimit -u`, `/proc/sys/kernel/threads-max` ограничивают количество.
- **Синхронизация** — mutex lock/unlock → memory barriers → дорого при высоком contention.

## Kernel threads vs User threads (Green threads)

| | Kernel threads | User threads (Green/Goroutines) |
|--|---------------|--------------------------------|
| Планировщик | ОС kernel | User-space (runtime) |
| Переключение | ~1-10 мкс | ~100-300 нс |
| Стек | 1-8 МБ фиксированный | 2-8 КБ, динамический рост |
| Блокировка | Блокирует весь поток | Другие goroutines продолжают |
| Параллелизм | Истинный (один поток = один CPU) | M горутин на N потоков (M:N) |

## Go Goroutines vs OS Threads

Go использует **M:N threading model** (GMP):

```
Goroutines (G) → Processors (P) → OS Threads (M)
```

- **G** (goroutine) — начальный стек 2 КБ (Go 1.21+), растёт до 1 ГБ.
- **P** (processor) — логический CPU, очередь горутин. Количество = GOMAXPROCS.
- **M** (machine) — OS thread. Один M = один kernel thread.

```go
// Goroutine стартует с ~2 КБ стека
go func() {
    // 100_000+ горутин легко, потоков — нет
}()

// Количество OS потоков
runtime.GOMAXPROCS(runtime.NumCPU()) // по умолчанию

// Получить ID OS потока (для отладки)
import "syscall"
tid := syscall.Gettid()
```

**Переключение горутин:**
- Cooperative: вызов функции, channel ops, syscall, `runtime.Gosched()`.
- Preemptive (Go 1.14+): signal-based preemption каждые 10ms.

## Примитивы синхронизации

### Mutex (взаимное исключение)

```go
import "sync"

var mu sync.Mutex
var counter int

// Только один поток держит блокировку
mu.Lock()
counter++
mu.Unlock()

// defer unlock — идиоматично
func increment() {
    mu.Lock()
    defer mu.Unlock()
    counter++
}

// RWMutex: много читателей или один писатель
var rwMu sync.RWMutex
rwMu.RLock()    // читатели
defer rwMu.RUnlock()
// ...
rwMu.Lock()     // писатель
defer rwMu.Unlock()
```

### Semaphore (ограничение concurrency)

```go
// Go не имеет встроенного Semaphore — делаем через канал
sem := make(chan struct{}, 10) // максимум 10 горутин

for i := 0; i < 100; i++ {
    sem <- struct{}{}   // захватить слот
    go func() {
        defer func() { <-sem }() // освободить слот
        doWork()
    }()
}

// Или используем golang.org/x/sync/semaphore
import "golang.org/x/sync/semaphore"

sem := semaphore.NewWeighted(10)
sem.Acquire(ctx, 1)
defer sem.Release(1)
```

### Condition Variable

```go
var mu sync.Mutex
var ready bool
cond := sync.NewCond(&mu)

// Ожидающий
go func() {
    mu.Lock()
    for !ready {
        cond.Wait() // atomically: unlock + sleep; на выходе: lock
    }
    doWork()
    mu.Unlock()
}()

// Уведомляющий
mu.Lock()
ready = true
cond.Signal()   // разбудить одного
// cond.Broadcast() // разбудить всех
mu.Unlock()
```

### sync.Once

```go
var once sync.Once
var instance *Service

func GetService() *Service {
    once.Do(func() {
        instance = &Service{} // вызовется только один раз
    })
    return instance
}
```

### sync.WaitGroup

```go
var wg sync.WaitGroup

for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        doWork(id)
    }(i)
}

// Go 1.25+: удобный метод
wg.Go(func() { doWork(0) }) // Add(1) + Done()

wg.Wait() // ждём всех
```

### sync.Map (concurrent-safe map)

```go
var m sync.Map

m.Store("key", "value")
v, ok := m.Load("key")
m.LoadOrStore("key", "default")
m.Delete("key")
m.Range(func(k, v any) bool {
    fmt.Println(k, v)
    return true // continue
})
```

## Дедлок (Deadlock)

```
Goroutine A: Lock(mu1) → wait(mu2)
Goroutine B: Lock(mu2) → wait(mu1)
→ Дедлок!
```

**Go обнаруживает дедлоки** (все горутины заблокированы):
```
fatal error: all goroutines are asleep - deadlock!
```

**Предотвращение:**
- Всегда захватывать мьютексы в одном порядке.
- Использовать `sync/atomic` для простых счётчиков.
- Предпочитать каналы (`channel`) мьютексам для передачи данных.
- `context.WithTimeout` для операций с таймаутом.

## Race Condition

```go
// Гонка данных (Data Race)
var counter int

go func() { counter++ }() // читает и пишет без блокировки
go func() { counter++ }() // одновременно

// Обнаружение: go run -race main.go
// go build -race ./...
```

**Инструменты:**
```bash
go test -race ./...
go build -race ./...  # встроенный race detector
```

## Аффинность и NUMA

```go
// Ограничить горутину одним CPU (экзотика)
runtime.LockOSThread()  // текущая горутина всегда на одном OS thread
defer runtime.UnlockOSThread()
```

```bash
# Закрепить процесс за CPU
taskset -c 0,1 ./myapp   # использовать только CPU 0 и 1

# NUMA-aware: процесс на том же CPU, что и его память
numactl --cpunodebind=0 --membind=0 ./myapp
```

## Что спрашивают на собеседованиях

1. **Чем горутина отличается от потока?** → Горутина: 2 КБ стек, планируется runtime в user-space, M:N модель. OS Thread: 1-8 МБ, планируется ядром.
2. **Что такое переключение контекста?** → CPU сохраняет регистры текущего потока и загружает регистры другого. ~1-10 мкс для OS threads, ~100-300 нс для горутин.
3. **Deadlock vs Livelock?** → Deadlock: все ждут друг друга навсегда. Livelock: все активны, но прогресса нет (перебрасывают ресурс).
4. **Когда использовать sync.Mutex, а когда channel?** → Mutex: для защиты shared state. Channel: для передачи данных и сигналов между горутинами.
5. **Что делает sync.Once?** → Гарантирует одноразовое выполнение (singleton, lazy init). Thread-safe без мьютекса явно.
6. **Что такое race condition?** → Несинхронизированный доступ к shared state из нескольких горутин; результат непредсказуем.
