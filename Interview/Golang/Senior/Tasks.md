# Go Senior Tasks — Разборы задач

Практические задачи на собеседованиях по Go.

---

## append

[Play](https://goplay.space/#oazR52NOiOr)

Что выведет программа?

```go
package main

import (
	"fmt"
)

func main() {
	var x []int
	x = append(x, 0)
	x = append(x, 1)
	x = append(x, 2)
	y := append(x, 3)
	z := append(x, 4)
	fmt.Println(y, z)
}
```

**Ответ:**
```
[0 1 2 4] [0 1 2 4]
```

**Объяснение:**

После `x = append(x, 2)` слайс `x` имеет: `len=3, cap=4` (Go удвоил: 2→4).

- `y := append(x, 3)` — помещает `3` в позицию `x[3]` (в существующем backing array). `y.len=4`.
- `z := append(x, 4)` — `x.len` всё ещё равен 3, cap=4. Помещает `4` в ту же позицию `x[3]`, перезаписывая `3`!
- Оба `y` и `z` указывают на **один backing array**, поэтому оба видят значение `4`.

```
backing array: [0, 1, 2, ?]
                            ↑
y → [0, 1, 2, 3]   z → [0, 1, 2, 4]  (тот же массив!)
                                           │
                                    z перезаписал x[3]
```

**Мораль:** никогда не используйте несколько слайсов, полученных через append от одного родительского слайса, если cap > len.

---

## Указатель на указатель

[Video](https://www.youtube.com/watch?v=rDZB4ueHxOY)

Что выведет программа?

```go
package main

import "fmt"

func increment(p *int) {
	*p++
}

func changePtr(pp **int) {
	x := 100
	*pp = &x
}

func main() {
	a := 1
	p := &a

	increment(p)
	fmt.Println(a, *p) // ?

	changePtr(&p)
	fmt.Println(*p)    // ?
}
```

**Ответ:**
```
2 2
100
```

**Объяснение:**

1. `increment(p)` — изменяет значение по адресу `p`, т.е. `a` становится `2`.
2. `changePtr(&p)` — `pp` = указатель на `p`. `*pp = &x` меняет `p` (не `a`) — теперь `p` указывает на `x = 100`.

```
До increment:   p → a(1)
После increment: p → a(2)
После changePtr: p → x(100)   (a всё ещё 2, но p уже не на неё)
```

**Применение:** linked lists, tree nodes — изменение самого указателя (не значения).

---

## defer

[Video](https://www.youtube.com/watch?v=_iyRjc9Cscw)

Задача 1: Что выведет?

```go
package main

import "fmt"

func a() int {
	i := 0
	defer func() {
		i++
		fmt.Println("defer:", i)
	}()
	return i
}

func main() {
	fmt.Println("return:", a())
}
```

**Ответ:**
```
defer: 1
return: 0
```

**Объяснение:** defer-функция выполняется после вычисления возвращаемого значения (return i → 0), но до фактического возврата. Замыкание изменяет `i`, но не возвращаемое значение (безымянный return).

---

Задача 2: А теперь с именованным возвращаемым значением?

```go
func b() (result int) {
	defer func() {
		result++
	}()
	return 0
}

func main() {
	fmt.Println(b()) // ?
}
```

**Ответ: 1**

**Объяснение:** `return 0` устанавливает `result = 0`, затем defer выполняет `result++` → `result = 1`. Именованный return позволяет defer изменить возвращаемое значение!

---

Задача 3: Порядок defer

```go
func c() {
	for i := 0; i < 3; i++ {
		defer fmt.Println(i)
	}
}
```

**Ответ:**
```
2
1
0
```

**LIFO:** defer'ы работают как стек — последний добавленный выполняется первым.

---

Задача 4: Аргументы defer вычисляются сразу

```go
func d() {
	x := 10
	defer fmt.Println("val:", x) // x вычислен СЕЙЧАС = 10
	defer func() { fmt.Println("closure:", x) }() // x при выполнении = 20
	x = 20
}
```

**Ответ:**
```
closure: 20
val: 10
```

---

## goroutine и каналы

[Video](https://www.youtube.com/watch?v=Cdoq5_kL8F0)

Задача 1: Что выведет?

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			fmt.Println(i) // ?
		}()
	}
	wg.Wait()
}
```

**Ответ (до Go 1.22):** скорее всего `5 5 5 5 5` (или любой порядок, все 5).

**Объяснение:** в Go < 1.22 переменная цикла `i` разделяется всеми горутинами. К моменту выполнения горутин `i` уже равно `5`.

**Go 1.22+:** каждая итерация создаёт новую переменную `i` → выведет `0 1 2 3 4` в произвольном порядке.

**Исправление для Go < 1.22:**

```go
for i := 0; i < 5; i++ {
	i := i // создать копию для каждой горутины
	wg.Add(1)
	go func() {
		defer wg.Done()
		fmt.Println(i)
	}()
}
// или передать как параметр:
go func(i int) {
    defer wg.Done()
    fmt.Println(i)
}(i)
```

---

Задача 2: Deadlock

```go
func main() {
	ch := make(chan int)
	ch <- 42 // ?
	fmt.Println(<-ch)
}
```

**Ответ:** `fatal error: all goroutines are asleep - deadlock!`

**Объяснение:** unbuffered channel — `ch <- 42` блокируется, ожидая получателя. Получателя нет (основная горутина заблокирована).

**Исправление:**
```go
// Вариант 1: buffered channel
ch := make(chan int, 1)
ch <- 42
fmt.Println(<-ch)

// Вариант 2: горутина
ch := make(chan int)
go func() { ch <- 42 }()
fmt.Println(<-ch)
```

---

## Fan-in (merge)

[Video](https://www.youtube.com/watch?v=dnGnKIB0_04)

Написать функцию, которая сливает несколько каналов в один:

```go
package main

import (
	"fmt"
	"sync"
)

// FanIn сливает несколько каналов в один
func FanIn(channels ...<-chan int) <-chan int {
	out := make(chan int)
	var wg sync.WaitGroup

	// Для каждого входного канала запускаем горутину
	for _, ch := range channels {
		wg.Add(1)
		go func(c <-chan int) {
			defer wg.Done()
			for v := range c {
				out <- v
			}
		}(ch)
	}

	// Закрыть out когда все горутины завершились
	go func() {
		wg.Wait()
		close(out)
	}()

	return out
}

func generateStream(nums ...int) <-chan int {
	ch := make(chan int)
	go func() {
		defer close(ch)
		for _, n := range nums {
			ch <- n
		}
	}()
	return ch
}

func main() {
	ch1 := generateStream(1, 2, 3)
	ch2 := generateStream(4, 5, 6)
	ch3 := generateStream(7, 8, 9)

	merged := FanIn(ch1, ch2, ch3)

	for v := range merged {
		fmt.Println(v) // 1-9 в произвольном порядке
	}
}
```

**Fan-in с select и context:**

```go
func FanInWithContext(ctx context.Context, channels ...<-chan int) <-chan int {
	out := make(chan int)
	var wg sync.WaitGroup

	for _, ch := range channels {
		wg.Add(1)
		go func(c <-chan int) {
			defer wg.Done()
			for {
				select {
				case v, ok := <-c:
					if !ok {
						return
					}
					select {
					case out <- v:
					case <-ctx.Done():
						return
					}
				case <-ctx.Done():
					return
				}
			}
		}(ch)
	}

	go func() {
		wg.Wait()
		close(out)
	}()

	return out
}
```

---

## Рандом с каналами

[Play](https://goplay.space/#rPiUUBYZEe8)

```go
package main

import (
	"fmt"
)

type C chan C

func main() {
	c := make(C, 1)
	c <- c

	for i := 0; i < 1000; i++ {
		select {
		case <-c:
		case <-c:
			c <- c
		default:
			fmt.Println(i)
			return
		}
	}
}
```

**Выводит случайное число:** 1, 2, 3, 4...

**Объяснение:**
- `c` — канал, который передаёт сам себя (C = chan C).
- В начале: `c <- c` помещает канал в буфер.
- На каждой итерации `select` выбирает случайный case:
  - `case <-c:` — читает из буфера (ничего не кладёт обратно)
  - `case <-c: c <- c` — читает И кладёт обратно (канал остаётся заполненным)
  - `default:` — если оба case недоступны (буфер пуст) → выводим i и выходим

Если первый case выбран первым → буфер пустеет → default срабатывает → i = 1.  
Если второй case выбирается несколько раз → буфер поддерживается → цикл продолжается.

---

## Работа с планировщиком и ссылками

[Play](https://go.dev/play/p/VmIEQOMRvl1)

```go
package main

import (
    "fmt"
    "runtime"
)

func main() {
    var holder [][]byte

    for i := 0; i < 10; i++ {
        chunk := make([]byte, 100_000_000) // 100 МБ
        holder = append(holder, chunk[:1]) // держим только 1 байт
        runtime.GC()
        var m runtime.MemStats
        runtime.ReadMemStats(&m)
        fmt.Printf("After iteration %d: HeapAlloc = %d MB\n", i, m.HeapAlloc/1024/1024)
    }
}

// Output:
// After iteration 0: HeapAlloc = 95 MB
// After iteration 1: HeapAlloc = 191 MB
// ...
// After iteration 9: HeapAlloc = 954 MB
```

**Проблема:** `chunk[:1]` создаёт слайс с тем же backing array что и `chunk`. GC не может освободить 100 МБ, пока жив хотя бы один слайс этого массива.

**Решение:** создать копию только нужного байта:

```go
holder = append(holder, append([]byte{}, chunk[:1]...)) // копируем 1 байт в новый массив

// Output:
// After iteration 0: HeapAlloc = 0 MB
// After iteration 1: HeapAlloc = 0 MB
// ...
// После GC большие chunks освобождаются
```

---

## GOMAXPROCS(1) — понимание планировщика

[Video](https://www.youtube.com/watch?v=_iyRjc9Cscw)

Что выведет при `GOMAXPROCS=1`?

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	runtime.GOMAXPROCS(1) // только 1 OS thread

	go func() {
		for {
			fmt.Println("goroutine")
		}
	}()

	for {
		fmt.Println("main")
		runtime.Gosched() // явно отдаём управление планировщику
	}
}
```

**Ответ:** чередование "main" и "goroutine".

**Без `runtime.Gosched()`** — бесконечно "main" (горутина не получает управление при GOMAXPROCS=1, если main не блокируется).

**Объяснение:**
- `GOMAXPROCS(1)` — только один P (логический CPU).
- Горутины переключаются кооперативно (и preemptively с Go 1.14+).
- `runtime.Gosched()` — явно уступает управление планировщику.
- Без Gosched и блокировки, main горутина держит P.

---

## Написать TCP сервер

```go
package main

import (
	"bufio"
	"fmt"
	"net"
	"strings"
)

func main() {
	listener, err := net.Listen("tcp", ":8080")
	if err != nil {
		panic(err)
	}
	defer listener.Close()
	fmt.Println("TCP server listening on :8080")

	for {
		conn, err := listener.Accept()
		if err != nil {
			fmt.Println("Accept error:", err)
			continue
		}
		go handleConnection(conn)
	}
}

func handleConnection(conn net.Conn) {
	defer conn.Close()
	fmt.Println("New connection from:", conn.RemoteAddr())

	scanner := bufio.NewScanner(conn)
	for scanner.Scan() {
		line := scanner.Text()
		fmt.Println("Received:", line)

		// Echo with transformation
		response := strings.ToUpper(line) + "\n"
		conn.Write([]byte(response))
	}
}
```

**Тест:**
```bash
# Запустить сервер
go run main.go

# Подключиться
nc localhost 8080
hello world
# → HELLO WORLD
```

**С graceful shutdown:**

```go
func main() {
	listener, _ := net.Listen("tcp", ":8080")

	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
	defer stop()

	var wg sync.WaitGroup

	go func() {
		for {
			conn, err := listener.Accept()
			if err != nil {
				select {
				case <-ctx.Done():
					return
				default:
					continue
				}
			}
			wg.Add(1)
			go func() {
				defer wg.Done()
				handleConnection(conn)
			}()
		}
	}()

	<-ctx.Done()          // ждём SIGTERM/SIGINT
	listener.Close()      // прекратить принимать новые соединения
	wg.Wait()             // дождаться завершения активных соединений
	fmt.Println("Server stopped gracefully")
}
```

---

## Worker Pool

Классический паттерн для ограничения параллелизма:

```go
package main

import (
	"fmt"
	"sync"
)

func workerPool(jobs <-chan int, results chan<- int, numWorkers int) {
	var wg sync.WaitGroup
	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			for job := range jobs {
				result := job * job // работа
				fmt.Printf("Worker %d processed job %d → %d\n", id, job, result)
				results <- result
			}
		}(i)
	}
	go func() {
		wg.Wait()
		close(results)
	}()
}

func main() {
	jobs := make(chan int, 100)
	results := make(chan int, 100)

	// Запустить пул из 3 воркеров
	workerPool(jobs, results, 3)

	// Отправить задачи
	for i := 1; i <= 10; i++ {
		jobs <- i
	}
	close(jobs)

	// Собрать результаты
	for r := range results {
		fmt.Println("Result:", r)
	}
}
```

---

## Pipeline pattern

```go
// Генератор → обработчик → принтер
func generator(nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for _, n := range nums {
			out <- n
		}
	}()
	return out
}

func square(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			out <- n * n
		}
	}()
	return out
}

func filter(in <-chan int, pred func(int) bool) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			if pred(n) {
				out <- n
			}
		}
	}()
	return out
}

func main() {
	// 1, 2, 3, 4, 5 → squares → filter even → print
	nums := generator(1, 2, 3, 4, 5)
	squares := square(nums)
	evens := filter(squares, func(n int) bool { return n%2 == 0 })

	for n := range evens {
		fmt.Println(n) // 4, 16
	}
}
```
