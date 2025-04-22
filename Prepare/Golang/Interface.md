# Interface

## Что такое `interface` в Go?

Интерфейс в Go — это **набор методов**. Если тип реализует все методы интерфейса — он **автоматически** считается реализацией интерфейса.

Например:

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

Любой тип, у которого есть метод `Read([]byte) (int, error)`, реализует `Reader`.

## Как устроены интерфейсы внутри

Интерфейс — это **двойка**:

1. **Type (или type descriptor)** — информация о реальном типе, который под капотом.
2. **Value (или data)** — сам "объект".

```go
interface {
    tab  *itab  // типовая информация
    data unsafe.Pointer // данные
}
```

Это важно для понимания `nil interface`, см. ниже.

## Что такое duck typing

**Duck typing** (утинная типизация) — концепция из динамически типизированных языков (Python, Ruby и т.п.), но в Go она **реализована статически** через интерфейсы.

> "If it walks like a duck and quacks like a duck, it’s a duck."
> _"Если это ходит как утка и квакает как утка, значит это утка."_

**Go** применяет **структурную типизацию**, а не номинативную, то есть:

- Тип реализует интерфейс **автоматически**, если содержит нужные методы.

Это и есть **Duck Typing**, но со **строгой проверкой на этапе компиляции**.

### Пример

```go
type Duck interface {
	Quack()
	Walk()
}

type RobotDuck struct{}

func (r RobotDuck) Quack() { fmt.Println("Robot: Quack") }
func (r RobotDuck) Walk()  { fmt.Println("Robot: Walk") }

func letItBeADuck(d Duck) {
	d.Quack()
	d.Walk()
}

func main() {
	var d Duck = RobotDuck{} // НЕ надо явно указывать реализацию
	letItBeADuck(d)
}
```

## Приведение одного интерфейса к другому

Это работает **если реальный тип внутри** реализует оба интерфейса.

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}

type Closer interface {
	Close() error
}

type ReadCloser interface {
	Reader
	Closer
}

func DoSomething(i interface{}) {
	if rc, ok := i.(ReadCloser); ok {
		fmt.Println("i implements ReadCloser")
		// используем rc.Read() и rc.Close()
	}
}
```

### Проверка типа интерфейса — `type assertion`

```go
var i interface{} = "hello"

s := i.(string)      // ОК, если уверен, что там string
s, ok := i.(string)  // Без паники, безопасный вариант

fmt.Println(s, ok) // "hello true"
```

### `type switch` — проверка конкретного типа

```go
func checkType(i interface{}) {
	switch v := i.(type) {
	case string:
		fmt.Println("String:", v)
	case int:
		fmt.Println("Int:", v)
	default:
		fmt.Println("Unknown type")
	}
}
```

## Не очевидные кейсы с `nil interface`

### Интерфейс не `nil`, если тип есть, но значение nil

```go
type Cat struct {
	Name string
}

func (c *Cat) Speak() {
	fmt.Println("meow")
}

func GetCat() Speaker {
	var c *Cat = nil
	return c // ВНИМАНИЕ: интерфейс НЕ nil!
}

func main() {
	s := GetCat()
	fmt.Println(s == nil) // false (!)
	s.Speak()             // panic: runtime error: invalid memory address
}
```

Почему? Потому что интерфейс содержит информацию о типе (`*Cat`), но значение `nil`. То есть:

- `type != nil`
- `value == nil`

А `interface == nil` **только когда оба nil**.

## Негативные кейсы с `nil interface`

### Ошибка сравнения

```go
func check(s Speaker) {
	if s == nil {
		fmt.Println("nil speaker")
	} else {
		fmt.Println("not nil") // попадём сюда даже если s содержит nil pointer!
	}
}
```

### `panic` при вызове метода на `nil` внутри интерфейса

```go
var c *Cat = nil
var s Speaker = c

s.Speak() // panic!
```

#### Как избегать

Проверяй и тип, и значение:

```go
if s == nil || reflect.ValueOf(s).IsNil() {
	// точно nil
}
```

или делай `type assertion`, а потом проверяй:

```go
if cat, ok := s.(*Cat); ok && cat == nil {
	// nil внутри
}
```

