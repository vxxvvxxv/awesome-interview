# HTTP / HTTPS / HTTP/2 / HTTP/3

Полезные ссылки:
- [RFC 9110 — HTTP Semantics](https://datatracker.ietf.org/doc/html/rfc9110)
- [HTTP/2 — RFC 9113](https://datatracker.ietf.org/doc/html/rfc9113)
- [MDN Web Docs — HTTP](https://developer.mozilla.org/ru/docs/Web/HTTP)

---

## Что такое HTTP?

**HTTP** (HyperText Transfer Protocol) — протокол прикладного уровня для передачи данных. Построен поверх **TCP** (HTTP/1.x, HTTP/2) или **QUIC/UDP** (HTTP/3).

**Ключевые характеристики:**
- **Stateless** — каждый запрос независим, сервер не хранит состояния.
- **Request-Response** — клиент инициирует, сервер отвечает.
- **Текстовый** (HTTP/1.x) / **Бинарный** (HTTP/2+).

## Структура HTTP-запроса

```
GET /api/users/42 HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{"filter": "active"}
```

**Части:** Request Line → Headers → (пустая строка) → Body.

## HTTP методы

| Метод | Семантика | Idempotent | Safe |
|-------|-----------|-----------|------|
| `GET` | Получить ресурс | Да | Да |
| `POST` | Создать / действие | Нет | Нет |
| `PUT` | Заменить полностью | Да | Нет |
| `PATCH` | Обновить частично | Нет* | Нет |
| `DELETE` | Удалить | Да | Нет |
| `HEAD` | Как GET, без body | Да | Да |
| `OPTIONS` | Доступные методы (CORS) | Да | Да |

**Idempotent** = N одинаковых запросов = тот же результат, что 1 раз.
**Safe** = не изменяет состояние сервера.

## Коды ответа

| Диапазон | Класс |
|----------|-------|
| **1xx** | Informational |
| **2xx** | Success |
| **3xx** | Redirection |
| **4xx** | Client Error |
| **5xx** | Server Error |

| Код | Описание |
|-----|---------|
| `200` | OK |
| `201` | Created (после POST) |
| `204` | No Content (после DELETE) |
| `301` | Moved Permanently |
| `304` | Not Modified (кэш актуален) |
| `400` | Bad Request |
| `401` | Unauthorized (нет аутентификации) |
| `403` | Forbidden (нет прав) |
| `404` | Not Found |
| `409` | Conflict |
| `422` | Unprocessable Entity (ошибка валидации) |
| `429` | Too Many Requests (rate limit) |
| `500` | Internal Server Error |
| `502` | Bad Gateway |
| `503` | Service Unavailable |
| `504` | Gateway Timeout |

## HTTP/1.0 → HTTP/1.1 → HTTP/2 → HTTP/3

### HTTP/1.1

- **Keep-Alive**: переиспользование TCP-соединения.
- **Pipelining**: несколько запросов без ожидания ответа (но ответы последовательны — **head-of-line blocking**).
- **Chunked transfer**: стриминг без Content-Length.
- **Host** заголовок: один IP → несколько доменов.

### HTTP/2 (2015) — полностью бинарный, поверх TCP

**Улучшения:**

1. **Мультиплексирование** — несколько параллельных запросов по одному TCP-соединению (нет head-of-line blocking на уровне HTTP).

2. **HPACK** — сжатие заголовков. Повторяющиеся заголовки хранятся в таблице — не дублируются.

3. **Server Push** — сервер может отправить ресурс до запроса клиента.

4. **Binary Framing** — данные разбиваются на фреймы.

```
HTTP/1.1:                     HTTP/2:
GET /a ─────────────►         ┌── Stream 1: GET /a ──►
        Response /a ◄─        │── Stream 3: GET /b ──►   параллельно
GET /b ─────────────►         └── Stream 5: GET /c ──►
        Response /b ◄─        ◄── Response 1, 3, 5 (в любом порядке)
```

**Ограничение HTTP/2**: head-of-line blocking на уровне **TCP** — потеря одного пакета блокирует все стримы.

### HTTP/3 (2022) — поверх QUIC (UDP)

- **Нет TCP head-of-line blocking**: каждый стрим в QUIC независим.
- **0-RTT**: соединение без дополнительного roundtrip (если ранее подключались).
- **TLS 1.3 встроен** в QUIC.
- Лучше при потере пакетов (мобильные сети).

## Чем HTTPS отличается от HTTP?

**HTTPS** = HTTP + **TLS** (Transport Layer Security).

**Что даёт TLS:**
- **Шифрование** — данные не читаемы посредником (MITM).
- **Аутентификация сервера** — клиент убеждён что общается с правильным сервером (через сертификат).
- **Целостность** — данные не изменены в пути (MAC).

### TLS Handshake (TLS 1.3, упрощённо)

```
Client                             Server
  │                                  │
  │──── ClientHello ────────────────►│  версии TLS, случайное число, cipher suites
  │◄─── ServerHello + Certificate ───│  выбранный cipher, сертификат
  │◄─── Finished ────────────────────│
  │──── Finished ────────────────────►│
  │═══════════ Encrypted Data ══════ │
```

TLS 1.3: **1 roundtrip** (TLS 1.2 — 2 roundtrip). С 0-RTT — 0 roundtrip.

## Кэширование

```http
# Сервер устанавливает
Cache-Control: max-age=3600       — кэш актуален 1 час
Cache-Control: no-cache           — всегда проверять (If-None-Match)
Cache-Control: no-store           — никогда не кэшировать
Cache-Control: private            — только браузер (не CDN)
ETag: "abc123"                    — тег версии ресурса

# Клиент проверяет
GET /data HTTP/1.1
If-None-Match: "abc123"

# Ответ если не изменилось
304 Not Modified  ← без тела, экономим трафик
```

## CORS

Механизм безопасности браузера: запросы к другому origin блокируются без явного разрешения сервера.

```http
# Preflight (OPTIONS)
OPTIONS /api/users HTTP/1.1
Origin: https://frontend.com
Access-Control-Request-Method: DELETE

# Ответ сервера
Access-Control-Allow-Origin: https://frontend.com
Access-Control-Allow-Methods: GET, POST, DELETE
Access-Control-Max-Age: 86400
```

## HTTP в Go

```go
// Клиент
client := &http.Client{
    Timeout: 10 * time.Second,
    Transport: &http.Transport{
        MaxIdleConnsPerHost: 10,
        IdleConnTimeout:     90 * time.Second,
    },
}

req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
req.Header.Set("Authorization", "Bearer "+token)

resp, err := client.Do(req)
if err != nil { return err }
defer resp.Body.Close()

// Сервер (Go 1.22+)
mux := http.NewServeMux()
mux.HandleFunc("GET /api/users/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{"id": id})
})

server := &http.Server{
    Addr:         ":8080",
    Handler:      mux,
    ReadTimeout:  10 * time.Second,
    WriteTimeout: 10 * time.Second,
}
```

## Что спрашивают на собеседованиях

1. **HTTP/1.1 vs HTTP/2?** → мультиплексирование, HPACK, Server Push, бинарный протокол.
2. **HTTP vs HTTPS?** → TLS: шифрование + аутентификация + целостность.
3. **Idempotent?** → повтор запроса = тот же эффект (GET, PUT, DELETE).
4. **401 vs 403?** → 401: нет токена; 403: токен есть, но прав нет.
5. **Как работает кэш с ETag?** → сервер выдаёт ETag; клиент присылает If-None-Match; 304 если не изменилось.
6. **Что такое CORS?** → браузер блокирует cross-origin запросы; сервер разрешает через заголовки.
7. **HTTP/3?** → поверх QUIC/UDP, нет TCP HoL blocking, встроенный TLS 1.3.
