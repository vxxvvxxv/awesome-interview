# Отладка и профилирование Go приложений

Полезные ссылки:
- [pprof documentation](https://pkg.go.dev/net/http/pprof)
- [Delve debugger](https://github.com/go-delve/delve)
- [Go Blog: Profiling Go Programs](https://go.dev/blog/profiling-go-programs)
- [runtime/pprof](https://pkg.go.dev/runtime/pprof)

---

## pprof — профилирование производительности

**pprof** — встроенный профилировщик Go. Собирает данные о CPU, памяти, горутинах.

### Включение pprof через HTTP

```go
import (
    _ "net/http/pprof"  // регистрирует /debug/pprof/ endpoint
    "net/http"
)

func main() {
    // Отдельный HTTP сервер для профилирования (не на продакшн порту!)
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()
    
    // ... основное приложение
}
```

### Сбор профилей

```bash
# CPU профиль (30 секунд)
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Memory профиль (heap allocation)
go tool pprof http://localhost:6060/debug/pprof/heap

# Goroutines (все горутины с трассировками)
go tool pprof http://localhost:6060/debug/pprof/goroutine

# Блокировки mutex
go tool pprof http://localhost:6060/debug/pprof/mutex

# Blocking профиль (что блокирует горутины)
go tool pprof http://localhost:6060/debug/pprof/block

# Tracing (30 секунд детального трейса)
curl http://localhost:6060/debug/pprof/trace?seconds=30 > trace.out
go tool trace trace.out
```

### Интерактивный анализ

```bash
# После загрузки профиля: интерактивный режим
(pprof) top10          # топ 10 функций по CPU
(pprof) top10 -cum     # топ 10 с накопительным временем
(pprof) list funcName  # исходный код с метриками
(pprof) web            # открыть граф в браузере (нужен Graphviz)
(pprof) png            # сохранить граф как PNG

# Веб-интерфейс pprof (Go 1.10+)
go tool pprof -http=:8080 cpu.prof
```

### Программный сбор профилей (без HTTP)

```go
import (
    "os"
    "runtime/pprof"
)

// CPU профиль
f, _ := os.Create("cpu.prof")
defer f.Close()
pprof.StartCPUProfile(f)
defer pprof.StopCPUProfile()

// Memory профиль
f, _ := os.Create("mem.prof")
defer f.Close()
runtime.GC() // форсировать GC перед снимком
pprof.WriteHeapProfile(f)
```

### Бенчмарки с профилированием

```bash
# Запустить бенчмарки и сохранить профиль
go test -bench=. -cpuprofile=cpu.prof -memprofile=mem.prof ./...

# Анализировать
go tool pprof -http=:8080 cpu.prof
go tool pprof -http=:8081 mem.prof
```

## Типичные находки при профилировании

```bash
# Лишние аллокации
# Симптом: GC работает часто (go_gc_duration_seconds растёт)
# В pprof: high inuse_objects или alloc_space

# Горячие пути (hot paths)
# Симптом: top10 показывает 80%+ в одной функции
# Решение: оптимизация, кэширование, батчинг

# Горутин утечки
# Симптом: /debug/pprof/goroutine показывает тысячи горутин
# Решение: отменять горутины через context, добавить таймауты

# Mutex contention
# Симптом: /debug/pprof/mutex показывает долгие блокировки
# Решение: sync.RWMutex, sharding, lock-free алгоритмы
```

## Delve (dlv) — интерактивный отладчик

**Delve** — полноценный Go отладчик.

### Установка

```bash
go install github.com/go-delve/delve/cmd/dlv@latest
```

### Основные команды

```bash
# Запуск отладки
dlv debug main.go          # скомпилировать и запустить под дебаггером
dlv exec ./myapp           # отлаживать готовый бинарник
dlv attach <PID>           # подключиться к запущенному процессу
dlv test ./...             # отлаживать тесты
dlv core ./myapp core.dump # анализ coredump

# Подключение к удалённому процессу (например в Docker)
# На сервере: dlv --listen=:2345 --headless=true exec ./myapp
dlv connect localhost:2345
```

### Команды внутри dlv

```bash
# Точки останова (breakpoints)
(dlv) break main.main          # bp на функцию
(dlv) break main.go:42         # bp на строку
(dlv) break *0x4b1234          # bp на адрес
(dlv) breakpoints              # список bp
(dlv) clear 1                  # удалить bp #1
(dlv) clearall                 # удалить все bp

# Управление выполнением
(dlv) continue  (c)    # продолжить до следующего bp
(dlv) next      (n)    # следующая строка (не входить в функцию)
(dlv) step      (s)    # следующая строка (входить в функцию)
(dlv) stepout   (so)   # выйти из текущей функции
(dlv) restart   (r)    # перезапустить

# Осмотр переменных
(dlv) print x          # значение переменной
(dlv) locals           # все локальные переменные
(dlv) args             # аргументы функции
(dlv) vars             # глобальные переменные
(dlv) whatis x         # тип переменной

# Горутины
(dlv) goroutines       # список всех горутин
(dlv) goroutine 5      # переключиться на горутину #5
(dlv) goroutine 5 bt   # стек горутины #5

# Стек
(dlv) stack            # текущий стек
(dlv) frame 2          # перейти в фрейм #2
(dlv) up / down        # вверх/вниз по стеку

# Watchpoints
(dlv) watch -rw x      # остановка при чтении/записи x
```

### Dlv в IDE

```
VS Code: расширение "Go" + launch.json:
{
    "type": "go",
    "request": "launch",
    "name": "Debug",
    "program": "${workspaceFolder}",
    "args": ["--port", "8080"]
}

GoLand: встроенный GUI дебаггер (использует dlv под капотом)
```

## Трассировка (go tool trace)

```bash
import "runtime/trace"

// Начать трассировку
f, _ := os.Create("trace.out")
trace.Start(f)
defer trace.Stop()

// Анализ
go tool trace trace.out
# Открывает браузер с интерактивным timeline:
# - Goroutines schedule timeline
# - Heap memory
# - GC activity
# - Network I/O
```

## GODEBUG — отладочные переменные runtime

```bash
# GC логирование
GODEBUG=gctrace=1 ./myapp
# Вывод: gc 7 @64.568s 11%: 0.58+1.2+0.83 ms clock, 3.5+6.2+5.0 ms cpu, 4->4->2 MB, 5 MB goal, 4 P

# Планировщик
GODEBUG=schedtrace=1000 ./myapp  # каждые 1000 мс

# Memory allocator
GODEBUG=allocfreetrace=1 ./myapp

# HTTP/2
GODEBUG=http2debug=1 ./myapp

# cgocheck: проверка передачи Go указателей в C
GODEBUG=cgocheck=2 ./myapp

# asyncpreemptoff: отключить asynchronous preemption (для отладки)
GODEBUG=asyncpreemptoff=1 ./myapp
```

## Полезные runtime функции

```go
import "runtime"

// Количество горутин
n := runtime.NumGoroutine()
log.Printf("goroutines: %d", n)

// Статистика памяти
var m runtime.MemStats
runtime.ReadMemStats(&m)
log.Printf("HeapAlloc: %d MB, GC cycles: %d", m.HeapAlloc/1024/1024, m.NumGC)

// Стек текущей горутины
buf := make([]byte, 1024)
n = runtime.Stack(buf, false) // false = только текущая горутина
log.Printf("Stack:\n%s", buf[:n])

// Стек всех горутин
buf = make([]byte, 64*1024)
n = runtime.Stack(buf, true) // true = все горутины

// Принудительный GC
runtime.GC()

// Принудительный запуск finalizers
runtime.GC()
runtime.Gosched()
```

## expvar — метрики через HTTP

```go
import "expvar"

var requestCount = expvar.NewInt("requests_total")
var activeConns = expvar.NewInt("active_connections")

// В коде
requestCount.Add(1)
activeConns.Add(1)
defer activeConns.Add(-1)

// GET http://localhost:6060/debug/vars
// {"cmdline": [...], "memstats": {...}, "requests_total": 42, "active_connections": 3}
```

## Что спрашивают на собеседованиях

1. **Как профилировать Go приложение?** → `import _ "net/http/pprof"` + `/debug/pprof/profile`, `go tool pprof`.
2. **Как обнаружить утечку горутин?** → `/debug/pprof/goroutine`, goleak в тестах, Go 1.26 goroutine leak profiler.
3. **Как найти лишние аллокации?** → `go test -memprofile`, `go tool pprof mem.prof`, пометки `-gcflags=-m`.
4. **Delve vs print-debugging?** → Delve: условные breakpoints, watchpoints, горутины, без перекомпиляции. Print: проще для CI, удалённых серверов.
5. **Что показывает go tool trace?** → Timeline выполнения горутин, GC паузы, I/O, планировщик. Точнее pprof для анализа задержек.
