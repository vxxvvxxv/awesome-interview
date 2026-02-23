# Cgo: Go и C interoperability

Полезные ссылки:
- [cgo documentation](https://pkg.go.dev/cmd/cgo)
- [Go Blog: C? Go? Cgo!](https://go.dev/blog/cgo)
- [runtime/cgo source](https://go.dev/src/runtime/cgo/)
- [Dave Cheney: cgo is not Go](https://dave.cheney.net/2016/01/18/cgo-is-not-go)

---

## Что такое Cgo?

**Cgo** — инструмент, позволяющий Go-коду вызывать C-код, и наоборот. Включается директивой `import "C"`.

**Когда использовать:**
- Нужна библиотека, существующая только на C (OpenSSL, SQLite, libavcodec).
- Критичная по производительности числодробилка уже написана на C.
- Системные вызовы, недоступные через `syscall`.

**Когда НЕ использовать:**
- Есть нативный Go аналог.
- Кросс-компиляция важна (`CGO_ENABLED=0`).
- Нужна скорость: вызов через Cgo медленнее прямого Go вызова (~30-60ns overhead).

## Базовый пример

```go
package main

/*
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void hello(const char* name) {
    printf("Hello, %s!\n", name);
}

int add(int a, int b) {
    return a + b;
}
*/
import "C" // нет пробела между комментарием и import!
import (
    "fmt"
    "unsafe"
)

func main() {
    // Вызов C функции
    result := C.add(3, 4)
    fmt.Println(int(result)) // 7

    // Передача строки в C
    name := C.CString("World") // выделяет C.malloc
    defer C.free(unsafe.Pointer(name)) // ОБЯЗАТЕЛЬНО освободить!
    C.hello(name)
}
```

## Типы данных

### Соответствие типов Go ↔ C

| C тип | Go тип (через C.*) |
|-------|-------------------|
| `char` | `C.char` |
| `int` | `C.int` |
| `long` | `C.long` |
| `double` | `C.double` |
| `void*` | `unsafe.Pointer` |
| `char*` | `*C.char` |
| `size_t` | `C.size_t` |
| `uint8_t` | `C.uint8_t` |

### Конвертация строк

```go
// Go string → C string (CString выделяет heap через C.malloc)
cStr := C.CString("hello")
defer C.free(unsafe.Pointer(cStr))
// ВАЖНО: defer обязателен, иначе утечка памяти C

// C string → Go string
cPtr := someC_function() // возвращает *C.char
goStr := C.GoString(cPtr)

// Go []byte → C *char с длиной
goBytes := []byte{1, 2, 3}
cPtr2 := C.CBytes(goBytes)
defer C.free(cPtr2)

// C bytes → Go []byte
goSlice := C.GoBytes(cPtr2, C.int(3))
```

### Числовые типы

```go
// Явное приведение типов обязательно
var n C.int = 42
goInt := int(n)

cLen := C.strlen(cStr) // C.size_t
goLen := int(cLen)
```

## Структуры C в Go

```go
/*
#include <stdint.h>

typedef struct {
    int x;
    int y;
    double radius;
} Circle;

double area(Circle* c) {
    return 3.14159 * c->radius * c->radius;
}
*/
import "C"
import "fmt"

func main() {
    c := C.Circle{
        x:      10,
        y:      20,
        radius: 5.0,
    }
    a := C.area(&c)
    fmt.Printf("Area: %.2f\n", float64(a))
}
```

## Экспорт Go функций в C

```go
package main

import "C"
import "fmt"

//export GoAdd
func GoAdd(a, b C.int) C.int {
    return a + b
}

//export GoHello
func GoHello(name *C.char) {
    fmt.Printf("Hello from Go, %s!\n", C.GoString(name))
}

func main() {} // нужна пустая main для CGO_BUILDMODE=c-shared
```

```bash
# Скомпилировать как C shared library
go build -buildmode=c-shared -o libmylib.so mylib.go
# Генерирует: libmylib.so + libmylib.h
```

## Директивы Cgo

```go
/*
#cgo CFLAGS: -I/usr/local/include -Wall -O2
#cgo LDFLAGS: -L/usr/local/lib -lmylib -lpthread

#cgo linux CFLAGS: -DLINUX
#cgo darwin CFLAGS: -DMACOS

#include "mylib.h"
*/
import "C"
```

```bash
# Переменные окружения
CGO_CFLAGS="-I/path"
CGO_LDFLAGS="-L/path -lname"
CGO_ENABLED=0  # отключить Cgo (статическая сборка)
```

## Управление памятью

**Ключевое правило:** Go GC не знает о C памяти.

```go
// C память — выделяем и освобождаем вручную
ptr := C.malloc(C.size_t(100))
defer C.free(ptr)

// Go память — GC управляет, но нельзя передавать указатели в C надолго
// (GC может переместить память)

// Правильно: скопировать данные в C память
goSlice := []byte("hello")
cBuf := C.CBytes(goSlice) // копирует в C heap
defer C.free(cBuf)
cFunc((*C.char)(cBuf), C.int(len(goSlice)))
```

### Правила передачи Go указателей в C (CGO Rule)

```
1. Go код может передавать Go указатели в C, если:
   - Go память, на которую указывает указатель, НЕ содержит Go указателей
   - C не хранит указатель после вызова

2. C код НЕ ДОЛЖЕН хранить Go указатель после возврата в Go

3. CGO_CHECK_POINTER=1 (по умолчанию) — runtime проверяет эти правила
```

```go
// Правильно: передать данные как C.CString (копия)
name := C.CString(goName)
defer C.free(unsafe.Pointer(name))
C.setName(name) // OK

// Неправильно: передать Go указатель напрямую для длительного хранения
// obj := &MyStruct{...}
// C.setObject(unsafe.Pointer(obj)) // UB если GC перемещает obj!

// Решение: runtime.Pinner (Go 1.21+)
var p runtime.Pinner
p.Pin(obj)
defer p.Unpin()
C.setObject(unsafe.Pointer(obj)) // OK — GC не переместит
```

## Переход Go → C: overhead

Вызов через Cgo медленнее из-за:
1. Переключение goroutine stack → OS thread stack.
2. Сохранение/восстановление Go регистров.
3. Проверка CGO_CHECK_POINTER.

```
Обычный Go вызов:         ~1 нс
Вызов через Cgo:          ~30-60 нс (Go 1.25)
                          ~20-40 нс (Go 1.26, ~30% улучшение)
```

**Оптимизация:** батчевать вызовы в C, минимизировать количество переходов.

## Кросс-компиляция и CGO_ENABLED

```bash
# Статическая сборка без Cgo (рекомендуется для Docker)
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o app .

# С Cgo нужен соответствующий C компилятор
# Для linux/arm64 на darwin:
CC=aarch64-linux-gnu-gcc CGO_ENABLED=1 GOOS=linux GOARCH=arm64 go build .

# Проверить, использует ли бинарник Cgo
ldd ./myapp  # если MUSL или glibc → Cgo
file ./myapp # статически слинкован → no Cgo
```

## Реальные примеры использования Cgo

### SQLite

```go
import (
    "database/sql"
    _ "github.com/mattn/go-sqlite3" // cgo binding
)

db, _ := sql.Open("sqlite3", "./mydb.db")
```

### OpenSSL через cgo (для специфичных алгоритмов)

```go
// Обычно лучше использовать crypto/tls из stdlib
// Cgo для OpenSSL нужен если FIPS-certified модуль требуется
```

### libpq (PostgreSQL нативный драйвер)

```go
import "github.com/lib/pq" // uses cgo для libpq
// vs
import "github.com/jackc/pgx/v5" // pure Go (рекомендуется)
```

## Альтернативы Cgo

| Альтернатива | Когда |
|-------------|-------|
| `syscall` / `golang.org/x/sys` | Системные вызовы POSIX/Linux |
| `plugin` | Go плагины (ограниченная поддержка) |
| WASM | Запуск C кода как WebAssembly |
| gRPC/TCP процесс | C сервер + Go клиент (лучшая изоляция) |
| Чистый Go аналог | Предпочтительно |

## Инструменты

```bash
# Посмотреть сгенерированный C код
go tool cgo main.go

# Сборка с отладочной информацией
CGO_CFLAGS="-g" go build .

# Профилирование C кода (pprof работает через cgo)
go tool pprof cpu.prof
```

## Что спрашивают на собеседованиях

1. **Что такое Cgo и зачем он нужен?** → Механизм вызова C из Go и обратно. Нужен для использования C библиотек без Go аналогов.
2. **Почему Cgo замедляет программу?** → Переключение stacks, сохранение регистров, pointer checks. ~30-60 нс на вызов.
3. **Почему CGO_ENABLED=0 важен?** → Статическая сборка, простая кросс-компиляция, меньше зависимостей. Почти всегда предпочтительнее в Docker.
4. **Как передать Go строку в C?** → `C.CString()` копирует в C heap. Обязательно `C.free()` после использования.
5. **Почему нельзя передавать Go указатели в C для долгосрочного хранения?** → GC может переместить память. Решение: `runtime.Pinner` (Go 1.21+).
6. **Cgo vs gRPC?** → Cgo: tight coupling, сложная сборка, быстро. gRPC/процесс: изоляция, простота, latency сети.
