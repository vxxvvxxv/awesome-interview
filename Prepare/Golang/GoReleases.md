# История релизов Go

> Go выходит каждые **6 месяцев**: в феврале и августе. Поддерживаются два последних major-релиза (security patches).

Полезные ссылки:
- [Release History — golang.org](https://go.dev/doc/devel/release)
- [Go features by version — antonz.org](https://antonz.org/which-go/)
- [Wikipedia: Go (programming language)](https://en.wikipedia.org/wiki/Go_(programming_language))

---

## Быстрая шпаргалка

| Версия | Дата | Ключевое |
|--------|------|---------|
| **1.14** | Фев 2020 | `//go:embed` precursor, defer overhead снижен |
| **1.15** | Авг 2020 | Улучшения линковщика |
| **1.16** | Фев 2021 | `//go:embed`, `io/fs`, модульный режим по умолчанию |
| **1.17** | Авг 2021 | Register-based calling convention (amd64), `go.mod` graphy |
| **1.18** | Мар 2022 | **Generics**, Fuzzing, Workspaces |
| **1.19** | Авг 2022 | `GOMEMLIMIT`, улучшенные doc-комментарии |
| **1.20** | Фев 2023 | PGO preview, `errors.Join`, slice→array |
| **1.21** | Авг 2023 | PGO GA, `min`/`max`/`clear`, `log/slog`, `slices`, `maps` |
| **1.22** | Фев 2024 | Фикс loop variable, `math/rand/v2`, range по int |
| **1.23** | Авг 2024 | Range over functions (итераторы), generic alias (exp.) |
| **1.24** | Фев 2025 | **Swiss Tables** maps, generic aliases GA, weak pointers, FIPS 140-3 |
| **1.25** | Авг 2025 | `WaitGroup.Go`, DWARF 5, JSON v2 (exp.), `testing/synctest` GA |
| **1.26** | Фев 2026 | **Green Tea GC** по умолчанию, `new(expr)`, -30% cgo overhead |

---

## Go 1.14 (февраль 2020)

**Производительность и инструментарий:**
- Defer стал почти нулевой стоимостью (overhead снижен за счёт оптимизации в компиляторе — `open-coded defer`).
- Module support стал production-ready.
- `go test -run` теперь поддерживает несколько паттернов.
- Интерфейсы с пустыми методами могут теперь иметь overlapping методы (подготовка к будущим фичам).

```go
// Пример: open-coded defer (почти как inline)
func f() {
    defer cleanup() // теперь inlined компилятором — нет heap-аллокации
}
```

---

## Go 1.15 (август 2020)

**Линковщик и инструменты:**
- Новый линковщик: значительно быстрее, меньше потребляет памяти.
- Улучшенная обработка `time.Timer` и `time.Ticker` (меньше горутин).
- `time.NewTimer` больше не блокируется при остановке.

---

## Go 1.16 (февраль 2021)

**Встраивание файлов в бинарник — `//go:embed`:**

```go
import _ "embed"

//go:embed config.yaml
var configData []byte

//go:embed templates/*.html
var templates embed.FS
```

- Пакет `io/fs` — абстрактная файловая система.
- **Модульный режим по умолчанию** (`GO111MODULE=on`).
- `go install pkg@version` — установка конкретной версии утилит.

**Собеседование:** часто спрашивают про `//go:embed` — как встроить файл в бинарник без внешних зависимостей.

---

## Go 1.17 (август 2021)

**Register-based calling convention:**
- На `amd64` аргументы функций передаются через **регистры** вместо стека.
- Ускорение большинства программ на **~5%** без изменения кода.
- `go.mod` теперь хранит граф зависимостей — `require` блок включает indirect deps.

```
// go.mod
require (
    golang.org/x/text v0.3.7 // indirect — теперь явно
)
```

---

## Go 1.18 (март 2022) — самый крупный релиз

### Generics (тип-параметры)

Главная фича за 10 лет разработки языка.

```go
// Функция с type parameter
func Map[T, U any](s []T, f func(T) U) []U {
    result := make([]U, len(s))
    for i, v := range s {
        result[i] = f(v)
    }
    return result
}

// Constraint через interface
type Number interface {
    ~int | ~int64 | ~float64
}

func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}
```

- Оператор `~` (tilde) — означает "все типы, чей underlying type — T".
- Ограничения через интерфейсы с type sets (`int | string | float64`).

### Fuzzing

Встроенный фаззинг — автоматическое тестирование с генерацией входных данных:

```go
func FuzzParseJSON(f *testing.F) {
    f.Add([]byte(`{}`))
    f.Fuzz(func(t *testing.T, data []byte) {
        var v interface{}
        json.Unmarshal(data, &v) // не должен паниковать
    })
}
```

```bash
go test -fuzz=FuzzParseJSON -fuzztime=30s
```

### Workspaces (`go work`)

Работа с несколькими модулями локально:

```bash
go work init ./moduleA ./moduleB
```

```
// go.work
go 1.18

use ./moduleA
use ./moduleB
```

**Собеседование:** Generics — обязательная тема. Спросят про `~`, type sets, ограничения `comparable` vs `any`.

---

## Go 1.19 (август 2022)

### GOMEMLIMIT — мягкий лимит памяти

```bash
GOMEMLIMIT=512MiB ./myapp
```

```go
import "runtime/debug"
debug.SetMemoryLimit(512 * 1024 * 1024) // 512 MiB
```

Отличие от `GOGC`:
- `GOGC` контролирует **частоту** GC (через отношение live heap к следующему запуску).
- `GOMEMLIMIT` — **жёсткая цель**: GC агрессивнее работает, если память превышает лимит.

Работают вместе: если хотим ограничить память контейнера — ставим `GOMEMLIMIT=80%` от лимита контейнера.

### Улучшенные doc-комментарии

```go
// Package http provides HTTP client and server implementations.
//
// Get, Head, Post, and PostForm make HTTP (or HTTPS) requests:
//
//	resp, err := http.Get("http://example.com/")
//
// [Client] is the main struct for making requests.
// See [Transport] for connection settings.
```

Теперь поддерживаются: ссылки `[Type]`, списки (`-`, `*`, числа), заголовки.

---

## Go 1.20 (февраль 2023)

### Profile-Guided Optimization (PGO) — preview

Оптимизация компилятора на основе реального профиля выполнения:

```bash
# 1. Соберите и запустите с профилировкой
go build -o myapp . && ./myapp -cpuprofile=cpu.pprof
# 2. Скопируйте профиль
cp cpu.pprof default.pgo
# 3. Пересоберите — компилятор использует профиль
go build -pgo=default.pgo -o myapp .
```

### `errors.Join`

```go
err1 := errors.New("db error")
err2 := errors.New("cache error")

combined := errors.Join(err1, err2)
// combined.Error() == "db error\ncache error"

errors.Is(combined, err1) // true
```

### Конвертация слайса в массив

```go
s := []int{1, 2, 3, 4}

// Go 1.17+: только pointer
p := (*[3]int)(s) // *[3]int

// Go 1.20: значение (copy)
a := [3]int(s) // [3]int{1, 2, 3}
```

### `http.ResponseController`

```go
rc := http.NewResponseController(w)
rc.SetWriteDeadline(time.Now().Add(5 * time.Second))
rc.Flush()
```

### `crypto/ecdh`

```go
privateKey, _ := ecdh.P256().GenerateKey(rand.Reader)
publicKey := privateKey.PublicKey()
```

---

## Go 1.21 (август 2023)

### PGO — General Availability

PGO теперь включается автоматически если в директории main-пакета лежит файл `default.pgo`. Ускорение: **2–7%** для большинства программ.

### Новые встроенные функции

```go
// min / max — работают с любыми ordered типами
a := min(1, 2, 3)    // 1
b := max(1.5, 2.5)   // 2.5

// clear — очищает map или обнуляет slice
m := map[string]int{"a": 1}
clear(m) // теперь пустая

s := []int{1, 2, 3}
clear(s) // [0, 0, 0]
```

### Пакет `log/slog` — структурированные логи

```go
import "log/slog"

logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

logger.Info("user login",
    slog.String("user", "alice"),
    slog.Int("status", 200),
)
// {"time":"...","level":"INFO","msg":"user login","user":"alice","status":200}
```

### Пакеты `slices` и `maps`

```go
import (
    "slices"
    "maps"
)

s := []int{3, 1, 4, 1, 5}
slices.Sort(s)           // [1, 1, 3, 4, 5]
idx, _ := slices.BinarySearch(s, 3) // 2

m := map[string]int{"a": 1, "b": 2}
maps.Keys(m)   // ["a", "b"] (порядок не гарантирован)
maps.Clone(m)  // глубокая копия
```

### Пакет `cmp`

```go
import "cmp"

cmp.Compare(1, 2) // -1 (1 < 2)
cmp.Or(0, "", "default") // "default" — первое ненулевое значение
```

### Гарантии совместимости GODEBUG

```go
// go.mod
go 1.21

// При обновлении — новое поведение включается только если go X.Y >= версии изменения
```

**Собеседование:** `log/slog` — часто спрашивают как организовать structured logging; `slices.Sort` vs `sort.Slice` — почему новый быстрее (generic, нет interface{}).

---

## Go 1.22 (февраль 2024)

### Фикс loop variable — историческая ловушка Go

До 1.22 классическая ошибка:

```go
// Go < 1.22: BUG — все горутины захватывают одну переменную i
for i := 0; i < 3; i++ {
    go func() {
        fmt.Println(i) // всегда 3, 3, 3
    }()
}

// Go >= 1.22: каждая итерация — новая переменная
for i := 0; i < 3; i++ {
    go func() {
        fmt.Println(i) // 0, 1, 2 (в каком-то порядке)
    }()
}
```

Аналогично для `range`:
```go
// Go >= 1.22
for _, v := range slice {
    go func() {
        use(v) // v — копия для каждой итерации
    }()
}
```

### Range по целым числам

```go
for i := range 5 {
    fmt.Println(i) // 0 1 2 3 4
}
```

### `math/rand/v2` — первый v2 пакет в stdlib

```go
import "math/rand/v2"

n := rand.IntN(100) // [0, 100) — без глобального сидирования вручную
```

Изменения:
- `Int63`/`Intn` → `Int64`/`IntN` (лучшие имена).
- Топ-уровневые функции рандомно засеяны по умолчанию.
- Алгоритмы: `PCG` и `ChaCha8` (вместо устаревшего LFSR).

### Routing improvements в `net/http`

```go
mux := http.NewServeMux()

// Теперь поддерживаются паттерны с методом и wildcards
mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    fmt.Fprintln(w, "User:", id)
})
```

**Собеседование:** Loop variable fix — классический вопрос. Нужно уметь объяснить что изменилось и почему это ломало старый код.

---

## Go 1.23 (август 2024)

### Range over Functions — пользовательские итераторы

Главная фича: `range` теперь работает с функциями-итераторами:

```go
// Тип итератора — func(yield func(K, V) bool)
func Fibonacci() iter.Seq[int] {
    return func(yield func(int) bool) {
        a, b := 0, 1
        for {
            if !yield(a) { // yield возвращает false — стоп
                return
            }
            a, b = b, a+b
        }
    }
}

// Использование
for n := range Fibonacci() {
    if n > 100 {
        break
    }
    fmt.Println(n)
}
```

Три сигнатуры итераторов (`iter.Seq`):
```go
func(yield func() bool)         // без значений
func(yield func(V) bool)        // iter.Seq[V]  — одно значение
func(yield func(K, V) bool)     // iter.Seq2[K, V] — ключ + значение
```

```go
import "iter"

// Итератор по парам из map (детерминированный порядок)
func SortedPairs[K cmp.Ordered, V any](m map[K]V) iter.Seq2[K, V] {
    return func(yield func(K, V) bool) {
        keys := slices.Sorted(maps.Keys(m))
        for _, k := range keys {
            if !yield(k, m[k]) {
                return
            }
        }
    }
}
```

### `iter` пакет

```go
import "iter"

// Стандартные утилиты
iter.Pull(seq)    // конвертирует push-итератор в pull-итератор
iter.Pull2(seq2)
```

### Изменения в `time.Timer` / `time.Ticker`

- `Timer.C` теперь гарантированно пустой после `Stop()` или `Reset()`.
- Исчезла классическая проблема с `drain`:

```go
// Раньше нужно было:
if !t.Stop() {
    <-t.C // drain
}
t.Reset(d)

// Теперь (Go 1.23) — просто:
t.Reset(d) // безопасно
```

### Generic type alias — preview

```go
// GOEXPERIMENT=aliastypeparams
type MyMap[K, V any] = map[K]V // generic alias (экспериментально)
```

---

## Go 1.24 (февраль 2025)

### Swiss Tables — новая реализация Map

[Подробно в Map.md](Map.md) — раздел Swiss Tables.

Коротко: новая хэш-таблица на основе Swiss Tables (от Google/Abseil):
- До **60% быстрее** в микробенчмарках (lookup, insert, delete).
- Снижение CPU overhead на **2–3%** в целом по рантайму.
- SIMD-поддержка на amd64.
- Полная обратная совместимость — код менять не нужно.

### Generic Type Aliases — GA

```go
// Теперь работает без GOEXPERIMENT
type Result[T any] = struct {
    Value T
    Err   error
}

type StringResult = Result[string]
```

### Weak Pointers

```go
import "weak"

// Слабая ссылка — не мешает GC собирать объект
ptr := weak.Make(&MyStruct{})

// Позже:
if obj, ok := ptr.Value(); ok {
    // объект ещё жив
    use(obj)
}
```

Применение: кэши без утечек памяти, интернирование строк.

### `runtime.AddCleanup` — замена `SetFinalizer`

```go
// Более гибкий, чем SetFinalizer
runtime.AddCleanup(&obj, func(data *Resource) {
    data.Close() // вызывается когда obj собирается GC
}, resource)
```

Преимущества над `SetFinalizer`:
- Не препятствует сборке в циклах.
- Можно навесить несколько.
- Явная передача аргумента (нет замыканий с захватом).

### `testing.B.Loop` для бенчмарков

```go
func BenchmarkSomething(b *testing.B) {
    // Новый способ — безопаснее, без b.N цикла
    b.Loop(func() {
        doWork()
    })
}
```

### `os.Root` — изолированная файловая система

```go
root, _ := os.OpenRoot("/var/app/data")
defer root.Close()

f, _ := root.Open("config.yaml") // только внутри /var/app/data
// Попытка "../etc/passwd" — ошибка
```

### Криптография

- **ML-KEM** (post-quantum, FIPS 203 / Kyber): `crypto/mlkem`
- `crypto/hkdf` — HKDF (RFC 5869)
- `crypto/pbkdf2` — PBKDF2 (RFC 8018)
- `crypto/sha3` — SHA-3 / SHAKE
- **FIPS 140-3** compliance встроен в stdlib

### WebAssembly: `//go:wasmexport`

```go
//go:wasmexport add
func add(a, b int32) int32 {
    return a + b
}
```

---

## Go 1.25 (август 2025)

### `sync.WaitGroup.Go` — удобный запуск горутин

```go
var wg sync.WaitGroup

// Старый способ
wg.Add(1)
go func() {
    defer wg.Done()
    doWork()
}()

// Go 1.25: новый способ
wg.Go(func() {
    doWork() // wg.Add/Done — автоматически
})

wg.Wait()
```

### DWARF 5

Дебаг-информация теперь в формате DWARF 5:
- Меньше размер бинарников.
- Быстрее линковка.
- Улучшенная поддержка дебаггерами (dlv, gdb).

### `testing/synctest` — GA

Пакет для тестирования конкурентного кода с контролем времени:

```go
import "testing/synctest"

func TestTimeout(t *testing.T) {
    synctest.Run(func() {
        ctx, cancel := context.WithTimeout(context.Background(), time.Second)
        defer cancel()

        synctest.Sleep(2 * time.Second) // продвигает виртуальное время

        select {
        case <-ctx.Done():
            // OK — таймаут произошёл
        }
    })
}
```

### JSON v2 (экспериментально)

```bash
GOEXPERIMENT=jsonv2 go build .
```

```go
import "encoding/json/v2"

// Нет автоматического OmitEmpty
// Строгий parsingMode (ошибка на неизвестные поля по умолчанию)
// Более быстрый, нет reflect overhead для простых типов

type User struct {
    Name string `json:",omitempty"`  // изменился синтаксис тегов
}
```

### Нет языковых изменений

Go 1.25 — первый релиз, в котором удалено понятие **core types** из спецификации языка (упрощение формальной модели generics). На реальный код это не влияет.

---

## Go 1.26 (февраль 2026)

### Green Tea GC — по умолчанию

[Подробно в Garbage_collector.md](Garbage_collector.md) — раздел Green Tea GC.

Коротко: новый сборщик мусора, ориентированный на **пространственную локальность**:
- Сканирует объекты **спанами по 8 KiB** вместо отдельных объектов.
- Фокус на малых объектах (≤ 512 байт) — именно они составляют большинство.
- SIMD-ускорение на amd64 (AVX-512).
- Снижение GC overhead на **10–40%** в реальных приложениях.
- **50% меньше** промахов L1/L2 кэша.

Для отключения: `GOEXPERIMENT=nogreenteagc` (будет убрано в Go 1.27).

### `new(expr)` — расширение встроенной функции

До 1.26 `new` принимал только тип:
```go
p := new(int)       // *int, *p == 0
```

Go 1.26 позволяет передать выражение с начальным значением:
```go
p := new(42)        // *int, *p == 42
s := new("hello")   // *string, *s == "hello"
```

Полезно для инициализации без промежуточной переменной:
```go
type Config struct{ Debug bool }
cfg := new(Config{Debug: true})  // *Config
```

### Снижение overhead cgo на ~30%

Базовые накладные расходы при вызове C-функций из Go снижены примерно на **30%** за счёт оптимизации переключения контекста между Go и C рантаймом.

### Рандомизация адреса heap

На 64-битных платформах base-адрес heap теперь **рандомизируется при старте**:
- Затрудняет эксплуатацию уязвимостей через предсказание адресов (ASLR на уровне heap).
- Особенно актуально при использовании cgo.

### Профилирование goroutine leaks (experimental)

```bash
GOEXPERIMENT=goroutineleakprofile go build .
```

Новый тип профиля `goroutineleak` в `runtime/pprof`:
```go
pprof.Lookup("goroutineleak").WriteTo(os.Stdout, 1)
```

Также доступен как HTTP-endpoint: `GET /debug/pprof/goroutineleak`.

### Stack-allocated slices

Компилятор теперь **чаще** размещает backing array слайса на стеке (escape analysis улучшен). Это снижает нагрузку на heap и GC для короткоживущих слайсов.

```go
func process() {
    // Компилятор может разместить [10]int на стеке, а не в heap
    s := make([]int, 0, 10)
    for i := range 10 {
        s = append(s, i*i)
    }
    use(s)
}
```

---

## Что спрашивают на собеседованиях

### Типичные вопросы по версиям

1. **"Что такое generics в Go и как они работают?"** → Go 1.18, type parameters, constraints, `~`
2. **"Что изменилось с loop variables?"** → Go 1.22 — каждая итерация создаёт новую переменную
3. **"Как работают итераторы через range?"** → Go 1.23 — `iter.Seq`, yield-функции
4. **"Почему maps стали быстрее в Go 1.24?"** → Swiss Tables — span-based lookup, SIMD
5. **"Что такое Green Tea GC?"** → Go 1.26 — span scanning, spatial locality
6. **"Как ограничить потребление памяти?"** → `GOMEMLIMIT` (Go 1.19)
7. **"Что такое PGO?"** → Profile-Guided Optimization (Go 1.21 GA)
8. **"Как встроить файл в бинарник?"** → `//go:embed` (Go 1.16)
9. **"Что такое weak pointer?"** → Go 1.24 — ссылка, не мешающая GC
10. **"Как сделать structured logging?"** → `log/slog` (Go 1.21)
