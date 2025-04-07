### Что такое контекст?

**`context`** в Go — это инструмент для управления жизненным циклом операций в Go. 
Также это механизм для передачи данных, сигналов отмены и дедлайнов между горутинами. 

#### Контекст помогает

- **Отменять** долгие операции (например, HTTP-запросы или БД-запросы).
- **Передавать** данные между цепочками вызовов (например, request-scoped данные: user ID, trace ID).
- **Контролировать** время выполнения операций (deadlines/timeouts).

#### Основные типы контекстов

- `context.Background()` — **базовый** контекст (обычно используется в `main()` или при инициализации).
- `context.TODO()` — **временный** контекст, когда нужный контекст ещё не ясен.
- `context.WithCancel()`, `context.WithTimeout()`, `context.WithDeadline()` — производные контексты **для отмены**.

### Что такое `gracefull shutdown`?

**Graceful Shutdown** — это корректное завершение работы сервера/приложения, при котором:
- Завершаются все активные соединения.
- Дожидается окончания текущих запросов.
- Освобождаются ресурсы (БД, файлы и т.д.).

**Зачем это нужно?**
- Чтобы не обрывать клиентские запросы резко (иначе пользователи получат ошибки).
- Чтобы избежать потери данных (например, если запрос к БД не успел завершиться).
- Чтобы корректно закрыть ресурсы (например, соединения с БД или файловые дескрипторы).

Пример:

```go
package main

import (
	"context"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	server := &http.Server{Addr: ":8080"}

	// Запуск сервера в горутине
	go func() {
		if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("Server error: %v", err)
		}
	}()

	// Ожидание сигналов завершения (Ctrl+C, SIGTERM)
	stop := make(chan os.Signal, 1)
	signal.Notify(stop, os.Interrupt, syscall.SIGTERM)
	<-stop // Блокировка до получения сигнала

	// Graceful Shutdown с таймаутом
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	if err := server.Shutdown(ctx); err != nil {
		log.Fatalf("Shutdown error: %v", err)
	}
	log.Println("Server stopped gracefully")
}
```

### Как контекст реализованы?

`context` реализован как **иммутабельная (неизменяемая) цепочка контекстов**, где каждый новый контекст наследует или переопределяет поведение родителя.

**Базовый interface Context (из исходного кода Go):**

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)  // Возвращает дедлайн, если есть
    Done() <-chan struct{}                   // Канал для отмены
    Err() error                              // Причина отмены
    Value(key any) any                       // Получение значения (key, value)
}
```

**Основные реализации:**
1. **`emptyCtx`** — базовый контекст (`Background()`, `TODO()`).    
2. **`cancelCtx`** — контекст с отменой (`WithCancel`).
3. **`timerCtx`** — контекст с таймаутом/дедлайном (`WithTimeout`, `WithDeadline`).
4. **`valueCtx`** — контекст с ключ-значением (`WithValue`).

### Как бы вы реализовали `context.WithCancel` и `context.WithTimeout`?

#### Реализация `WithCancel`

```go
// TODO
```

#### Реализация `WithTimeout`

```go
// TODO
```

