## append

[Play](https://goplay.space/#oazR52NOiOr)

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

```go
[0 1 2 4] [0 1 2 4]
```

## Указатель на указатель

TODO
https://www.youtube.com/watch?v=rDZB4ueHxOY

## defer

TODO
https://www.youtube.com/watch?v=_iyRjc9Cscw


## goroutine и каналы

TODO

https://www.youtube.com/watch?v=Cdoq5_kL8F0

## Fan-in (merge)

TODO
https://www.youtube.com/watch?v=dnGnKIB0_04

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

Выводить будет рандомно, то 1, то 2, то 3, то 4 и т.д.

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
        chunk := make([]byte, 100_000_000)
        holder = append(holder, chunk[:1])
        runtime.GC()
        var m runtime.MemStats
        runtime.ReadMemStats(&m)
        fmt.Printf("After iteration %d: HeapAlloc = %d MB\n", i, m.HeapAlloc/1024/1024)
    }
}

// Output:
// After iteration 0: HeapAlloc = 95 MB
// After iteration 1: HeapAlloc = 191 MB
// After iteration 2: HeapAlloc = 286 MB
// After iteration 3: HeapAlloc = 381 MB
// After iteration 4: HeapAlloc = 477 MB
// After iteration 5: HeapAlloc = 572 MB
// After iteration 6: HeapAlloc = 667 MB
// After iteration 7: HeapAlloc = 763 MB
// After iteration 8: HeapAlloc = 858 MB
// After iteration 9: HeapAlloc = 954 MB
```

Решение

```go
holder = append(holder, append([]byte{}, chunk[:1]...)) // создаём копию 1 байта

// Output:
// After iteration 0: HeapAlloc = 0 MB
// After iteration 1: HeapAlloc = 0 MB
// After iteration 2: HeapAlloc = 0 MB
// After iteration 3: HeapAlloc = 0 MB
// After iteration 4: HeapAlloc = 0 MB
// After iteration 5: HeapAlloc = 0 MB
// After iteration 6: HeapAlloc = 0 MB
// After iteration 7: HeapAlloc = 0 MB
// After iteration 8: HeapAlloc = 0 MB
// After iteration 9: HeapAlloc = 0 MB
```

## GOMAXPROC(1)

TODO
https://www.youtube.com/watch?v=_iyRjc9Cscw

Написать TCP сервер
```go
package main

import "net"

func main() {
	net.Listen()

}
```