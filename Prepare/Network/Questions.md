# Сетевые вопросы: что происходит при вводе URL

## Что происходит, когда ввёл ссылку в браузере и получил страницу?

Полный цикл от нажатия Enter до отображения страницы — классический вопрос на собеседованиях.

---

### Шаг 1: Парсинг URL

```
https://www.example.com:443/path/to/page?query=value#section

scheme:   https
host:     www.example.com
port:     443 (для https по умолчанию)
path:     /path/to/page
query:    ?query=value
fragment: #section (только в браузере, не отправляется на сервер)
```

### Шаг 2: DNS резолюция (www.example.com → IP)

```
1. Кэш браузера       → уже есть IP?
2. Кэш ОС             → /etc/hosts, systemd-resolved?
3. Recursive resolver  → обычно провайдера или 8.8.8.8
4. Root сервер         → спрашивает .com
5. TLD сервер (.com)   → отдаёт NS записи example.com
6. Authoritative NS    → возвращает A/AAAA запись (IP)
7. IP сохраняется с TTL
```

Подробнее: [DNS.md](DNS.md)

### Шаг 3: TCP соединение (TCP 3-Way Handshake)

```
Browser                   Server (IP:443)
  │──── SYN ──────────────►│
  │◄─── SYN-ACK ───────────│
  │──── ACK ──────────────►│
```

- Если HTTPS (порт 443) → после TCP сразу TLS Handshake.
- Если HTTP (порт 80) → сразу данные.

### Шаг 4: TLS Handshake (для HTTPS)

```
Browser                        Server
  │──── ClientHello ──────────►│  TLS версии, cipher suites, random
  │◄─── ServerHello ───────────│  выбранный cipher, сертификат
  │◄─── Certificate ───────────│  X.509 сертификат
  │                            │
  │  (браузер проверяет сертификат:
  │   - не истёк ли?
  │   - подписан доверенным CA?
  │   - CN/SAN совпадает с hostname?)
  │                            │
  │──── Key Exchange ─────────►│  ECDHE shared secret
  │◄─── Finished ──────────────│
  │──── Finished ─────────────►│
  │════════ Encrypted ═════════│
```

TLS 1.3: только **1 RTT** (вместо 2 в TLS 1.2). Подробнее: [TLS.md](../Crypto/TLS.md)

### Шаг 5: HTTP Request

```
GET /path/to/page?query=value HTTP/2
Host: www.example.com
Accept: text/html,application/xhtml+xml
Accept-Language: ru,en
Accept-Encoding: gzip, deflate, br
Cookie: session_id=abc123
User-Agent: Mozilla/5.0 ...
Connection: keep-alive
```

### Шаг 6: Обработка на сервере

```
Browser → Load Balancer → Web Server (nginx) → App Server (Go) → Database/Cache

1. Load Balancer выбирает backend (round-robin / least connections)
2. nginx/Caddy: TLS termination, статика, rate limiting
3. Go сервер: роутинг, middleware, бизнес-логика
4. БД: PostgreSQL / Redis / etc.
5. Формирование ответа
```

### Шаг 7: HTTP Response

```
HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
Content-Encoding: gzip
Cache-Control: max-age=300
ETag: "abc123"
Set-Cookie: session_id=xyz789; HttpOnly; Secure

<html>...
```

### Шаг 8: Браузерный рендеринг

```
1. Parsing HTML → DOM Tree
2. Parsing CSS → CSSOM
3. DOM + CSSOM → Render Tree
4. Layout (Reflow): вычисление позиций/размеров
5. Painting: пикселизация
6. Compositing: отрисовка слоёв GPU

Параллельно:
- Загрузка CSS → блокирует рендеринг (render-blocking)
- Загрузка JS → блокирует парсинг (если без async/defer)
- Загрузка изображений → асинхронно
```

### Шаг 9: JS выполнение

```
1. Download JS файлы (async/defer)
2. Parse → AST
3. Compile (JIT)
4. Execute
5. Event Loop: обработка событий, fetch, setTimeout
6. React/Vue гидратация (если SPA)
```

### Шаг 10: Закрытие соединения (TCP FIN)

HTTP/2 поддерживает multiplexing — несколько запросов по одному соединению. Соединение закрывается по таймауту или явно.

```
Browser                   Server
  │──── FIN ─────────────►│
  │◄─── ACK ───────────────│
  │◄─── FIN ───────────────│
  │──── ACK ─────────────►│
```

---

## Сводная таблица

| Шаг | Протокол | Время |
|-----|----------|-------|
| DNS | UDP/53 → TCP/53 | 1-100ms |
| TCP Handshake | TCP | 0.5 RTT |
| TLS 1.3 Handshake | TLS over TCP | 1 RTT |
| HTTP Request/Response | HTTP/2 | зависит от сервера |
| Рендеринг | — | 10-100ms |

**Full page load: 0.5–3 сек** (для типичного сайта)

---

## HTTP/2 vs HTTP/1.1 в браузере

- **HTTP/1.1**: браузер открывает 6 параллельных соединений к одному домену.
- **HTTP/2**: одно соединение, multiplexing (все ресурсы параллельно).
- **HTTP/3**: QUIC вместо TCP (0-RTT reconnect, нет HOL blocking).

---

## Оптимизации (что ускоряет загрузку)

```
DNS Prefetch:    <link rel="dns-prefetch" href="//cdn.example.com">
Preconnect:      <link rel="preconnect" href="https://api.example.com">
Preload:         <link rel="preload" href="font.woff2" as="font">
CDN:             статика ближе к пользователю
Gzip/Brotli:     сжатие (Brotli ~20% лучше Gzip)
HTTP/2 Push:     отдать ресурсы до запроса браузера
Service Worker:  кэш в браузере (PWA)
```

## Что спрашивают на собеседованиях (от простого к сложному)

1. **DNS запрос — UDP или TCP?** → UDP/53 для запросов (быстро), TCP/53 для zone transfer и больших ответов.
2. **HTTPS — зачем TLS?** → Конфиденциальность (шифрование), целостность (HMAC), аутентичность (сертификат).
3. **Что такое HSTS?** → Сервер говорит браузеру "всегда использовать HTTPS" (`Strict-Transport-Security` header).
4. **Что такое CDN?** → Сеть серверов по миру; статика отдаётся с ближайшего узла.
5. **Что такое HTTP/2 Multiplexing?** → Несколько запросов параллельно по одному TCP соединению без HOL blocking.
6. **Что блокирует рендеринг?** → CSS (render-blocking), синхронный JS (parser-blocking). Решения: `async`, `defer`, критический CSS inline.
