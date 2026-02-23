# TCP и UDP

Полезные ссылки:
- [RFC 793 — TCP](https://datatracker.ietf.org/doc/html/rfc793)
- [RFC 768 — UDP](https://datatracker.ietf.org/doc/html/rfc768)
- [Computer Networking: A Top-Down Approach](https://gaia.cs.umass.edu/kurose_ross/)

---

## Транспортный уровень

TCP и UDP — протоколы **транспортного уровня** (L4) модели OSI. Работают поверх IP (L3), обеспечивают связь между **процессами** (через порты).

## Что такое TCP?

**TCP** (Transmission Control Protocol) — надёжный протокол с установлением соединения.

**Гарантии:**
- **Доставка** — подтверждения (ACK), повторная отправка при потере.
- **Порядок** — пакеты доставляются в правильном порядке (sequence numbers).
- **Без дублирования** — дубликаты отбрасываются.
- **Контроль потока** — receiver window (не перегружает получателя).
- **Контроль перегрузки** — slow start, AIMD (не перегружает сеть).

**Цена:** задержка (handshake, подтверждения), overhead.

## Что такое UDP?

**UDP** (User Datagram Protocol) — простой протокол без установления соединения.

**Характеристики:**
- **Нет гарантий** доставки и порядка.
- **Нет overhead** handshake.
- **Нет состояния** — connectionless.
- **Минимальный header** (8 байт vs 20+ у TCP).

**Применение:** DNS, DHCP, видеостриминг, игры, VoIP, QUIC (HTTP/3).

## Сравнение

| Характеристика | TCP | UDP |
|---------------|-----|-----|
| Надёжность | Гарантирована | Нет |
| Порядок | Гарантирован | Нет |
| Скорость | Медленнее | Быстрее |
| Overhead | 20+ байт header | 8 байт header |
| Соединение | Stateful | Stateless |
| Пример | HTTP, HTTPS, SSH | DNS, Video, QUIC |

---

## TCP 3-Way Handshake (установление соединения)

```
Client                    Server
  │                          │
  │──── SYN (seq=x) ────────►│   Клиент: хочу соединение
  │                          │   (SYN = synchronize)
  │◄─── SYN-ACK ─────────────│   Сервер: готов (seq=y, ack=x+1)
  │     (seq=y, ack=x+1)     │
  │                          │
  │──── ACK (ack=y+1) ───────►│   Клиент: подтверждаю
  │                          │
  │═══════ Data ═════════════│   Соединение установлено
```

**SYN flood атака** — атакующий засыпает сервер SYN-пакетами, не отвечая ACK. Защита: SYN cookies.

## TCP 4-Way Termination (закрытие соединения)

```
Client                    Server
  │                          │
  │──── FIN ────────────────►│   Клиент: заканчиваю отправку
  │◄─── ACK ─────────────────│   Сервер: принял FIN
  │◄─── FIN ─────────────────│   Сервер: тоже заканчиваю
  │──── ACK ────────────────►│   Клиент: принял
  │                          │
  │  (TIME_WAIT 2*MSL)       │   Клиент ждёт на случай повтора FIN
```

**Состояния:** `ESTABLISHED → FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED`

`TIME_WAIT` (2*MSL ≈ 60-120 сек) — защита от "призрачных" пакетов. Проблема при перезапуске сервера: `SO_REUSEADDR`.

## TCP Sequence Numbers и ACK

```
Sender:   seq=1000, len=500 → данные 1000-1499
Receiver: ack=1500           → "жду байт 1500"
Sender:   seq=1500, len=500 → данные 1500-1999
```

Если ACK не пришёл → ретрансмиссия (RTO = Retransmission Timeout).

## Контроль потока (Flow Control)

**Sliding Window** — получатель сообщает свой буфер (receiver window = rwnd):

```
Sender может иметь в полёте: min(cwnd, rwnd) байт
```

Если `rwnd = 0` → sender приостанавливает. Zero Window Probe — проверяет открылось ли окно.

## Контроль перегрузки (Congestion Control)

### Slow Start
```
cwnd = 1 MSS → 2 → 4 → 8 (удвоение на каждый RTT)
до ssthresh → переход в Congestion Avoidance
```

### Congestion Avoidance (AIMD)
```
cwnd += 1 MSS на каждый RTT (линейный рост)
При потере: cwnd /= 2, ssthresh = cwnd/2 (multiplicative decrease)
```

### TCP CUBIC (Linux default), BBR (Google) — современные алгоритмы

## Алгоритм Nagle

Буферизует маленькие пакеты, пока есть неподтверждённые данные:

```
Если данных мало И есть unACK'd данные → буферизовать
Иначе → отправить немедленно
```

Для интерактивных приложений (SSH, игры) — **отключать**:

```go
conn.(*net.TCPConn).SetNoDelay(true) // TCP_NODELAY
```

## TCP Keep-Alive

Обнаруживает "мёртвые" соединения:

```go
conn, _ := net.Dial("tcp", "server:80")
tcpConn := conn.(*net.TCPConn)
tcpConn.SetKeepAlive(true)
tcpConn.SetKeepAlivePeriod(30 * time.Second)
```

## TCP в Go

```go
// Server
listener, err := net.Listen("tcp", ":8080")
if err != nil {
    log.Fatal(err)
}
defer listener.Close()

for {
    conn, err := listener.Accept()
    if err != nil {
        continue
    }
    go handleConn(conn)
}

func handleConn(conn net.Conn) {
    defer conn.Close()
    conn.SetDeadline(time.Now().Add(30 * time.Second)) // таймаут

    buf := make([]byte, 4096)
    for {
        n, err := conn.Read(buf)
        if err != nil {
            return
        }
        conn.Write(buf[:n]) // echo
    }
}

// Client
conn, err := net.DialTimeout("tcp", "server:8080", 5*time.Second)
if err != nil {
    log.Fatal(err)
}
defer conn.Close()
conn.Write([]byte("hello"))
```

## UDP в Go

```go
// Server
addr, _ := net.ResolveUDPAddr("udp", ":8080")
conn, _ := net.ListenUDP("udp", addr)
defer conn.Close()

buf := make([]byte, 65535)
for {
    n, remoteAddr, _ := conn.ReadFromUDP(buf)
    conn.WriteToUDP(buf[:n], remoteAddr) // echo
}

// Client
conn, _ := net.Dial("udp", "server:8080")
conn.Write([]byte("ping"))
buf := make([]byte, 1024)
n, _ := conn.Read(buf)
fmt.Println(string(buf[:n]))
```

## TCP Socket Options

```go
// SO_REUSEADDR — переиспользование порта (после перезапуска)
lc := net.ListenConfig{
    Control: func(network, address string, c syscall.RawConn) error {
        return c.Control(func(fd uintptr) {
            syscall.SetsockoptInt(int(fd), syscall.SOL_SOCKET, syscall.SO_REUSEADDR, 1)
        })
    },
}
listener, _ := lc.Listen(context.Background(), "tcp", ":8080")

// SO_LINGER — ждать отправки данных при Close()
tcpConn.SetLinger(5) // 5 секунд

// Размеры буферов
tcpConn.SetReadBuffer(64 * 1024)
tcpConn.SetWriteBuffer(64 * 1024)
```

## QUIC (UDP + надёжность)

**QUIC** (Quick UDP Internet Connections) — протокол Google, стал основой **HTTP/3**:

- Работает поверх UDP.
- Реализует надёжность на уровне приложения (не ОС).
- **0-RTT** для повторных соединений.
- **Multiplexing без head-of-line blocking** (в отличие от HTTP/2 над TCP).
- Встроенный TLS 1.3.

```go
// Go HTTP/3 с quic-go
import "github.com/quic-go/quic-go/http3"

http3.ListenAndServeTLS(":443", "cert.pem", "key.pem", nil)
```

## Что спрашивают на собеседованиях

1. **Чем TCP отличается от UDP?** → TCP: надёжность, порядок, контроль потока. UDP: без гарантий, быстрее.
2. **Опишите TCP handshake.** → SYN → SYN-ACK → ACK. 3 шага установления соединения.
3. **Что такое TIME_WAIT?** → Состояние после закрытия соединения (2*MSL), защита от старых пакетов.
4. **Зачем нужен Sequence Number?** → Порядок доставки и дедупликация пакетов.
5. **Slow Start vs AIMD?** → Slow Start: экспоненциальный рост cwnd. AIMD: линейный рост + делить на 2 при потере.
6. **Когда использовать UDP?** → Когда задержка важнее надёжности: DNS, видео, игры, QUIC.
7. **Что такое Nagle Algorithm?** → Буферизация маленьких пакетов. Отключить (TCP_NODELAY) для интерактивных приложений.
8. **SYN flood атака?** → Атакующий отправляет SYN без ACK, переполняя очередь соединений. Защита: SYN cookies.
