# REST API

Полезные ссылки:
- [Roy Fielding — Architectural Styles (оригинальная диссертация)](https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm)
- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- [JSON:API](https://jsonapi.org/)

---

## Что такое REST?

**REST** (Representational State Transfer) — архитектурный стиль распределённых систем, описанный Roy Fielding в 2000 году в диссертации. REST — не протокол и не стандарт.

**RESTful API** — API, которое следует принципам REST.

## 6 ограничений REST (Fielding Constraints)

1. **Client-Server** — разделение UI и хранения данных.
2. **Stateless** — каждый запрос содержит всю необходимую информацию. Сессии на сервере — нарушение.
3. **Cacheable** — ответы помечаются как кэшируемые или нет.
4. **Uniform Interface** — единый интерфейс (ресурсы, представления, самоописание, HATEOAS).
5. **Layered System** — клиент не знает, общается ли он с конечным сервером или прокси.
6. **Code on Demand** *(опционально)* — сервер может передавать исполняемый код.

## Ресурсы и URL

```
Правильно:
GET    /users          # список пользователей
GET    /users/42       # пользователь с ID=42
POST   /users          # создать пользователя
PUT    /users/42       # заменить пользователя
PATCH  /users/42       # обновить частично
DELETE /users/42       # удалить

Вложенные ресурсы:
GET    /users/42/orders        # заказы пользователя
POST   /users/42/orders        # создать заказ для пользователя
GET    /users/42/orders/7      # конкретный заказ

Неправильно (RPC-стиль, не REST):
POST /getUser
GET  /deleteUser?id=42
POST /createUserOrder
```

**Правила именования:**
- Существительные, не глаголы: `/orders`, не `/getOrders`
- Множественное число: `/users`, не `/user`
- Lowercase с дефисами: `/order-items`, не `/orderItems`
- Без trailing slash: `/users`, не `/users/`

## HTTP методы

| Метод | Идемпотентен | Безопасен | Применение |
|-------|-------------|----------|-----------|
| GET | Да | Да | Получить ресурс |
| HEAD | Да | Да | Только заголовки |
| POST | **Нет** | Нет | Создать ресурс |
| PUT | Да | Нет | Заменить ресурс целиком |
| PATCH | **Нет** | Нет | Частичное обновление |
| DELETE | Да | Нет | Удалить ресурс |

> **Идемпотентный** = повторный вызов с теми же данными даёт тот же результат.  
> **Безопасный** = не изменяет состояние сервера.

## HTTP Status Codes

```
2xx — Успех
  200 OK              — стандартный успех (GET, PUT, PATCH)
  201 Created         — ресурс создан (POST)
  204 No Content      — успех без тела (DELETE)

3xx — Редирект
  301 Moved Permanently  — постоянный редирект
  304 Not Modified       — кэш актуален (ETag/If-None-Match)

4xx — Ошибка клиента
  400 Bad Request     — некорректный запрос
  401 Unauthorized    — нет аутентификации (нужен токен)
  403 Forbidden       — нет доступа (токен есть, прав нет)
  404 Not Found       — ресурс не найден
  409 Conflict        — конфликт (дублирование)
  422 Unprocessable   — валидация провалена
  429 Too Many Requests — rate limit

5xx — Ошибка сервера
  500 Internal Server Error — ошибка сервера
  502 Bad Gateway     — upstream не ответил
  503 Service Unavailable — сервис недоступен
  504 Gateway Timeout — timeout upstream
```

## Формат ответов

```json
// Успех (один ресурс)
{
  "id": 42,
  "name": "John Doe",
  "email": "john@example.com",
  "created_at": "2024-01-15T10:30:00Z"
}

// Список с пагинацией
{
  "data": [...],
  "meta": {
    "total": 150,
    "page": 1,
    "per_page": 20
  }
}

// Ошибка
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Email is invalid",
    "details": [
      {"field": "email", "message": "must be a valid email"}
    ]
  }
}
```

## Версионирование API

```bash
# 1. URL versioning (наиболее распространённое)
GET /api/v1/users
GET /api/v2/users

# 2. Header versioning
GET /users
Accept: application/vnd.myapp.v2+json

# 3. Query parameter (не рекомендуется)
GET /users?version=2
```

## Аутентификация

### JWT (JSON Web Token)

```
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

```
Header.Payload.Signature
{alg,typ} . {sub,iat,exp,roles} . Sign(header+payload, secret)
```

### API Key

```
X-API-Key: abc123xyz
# или
GET /resource?api_key=abc123xyz  # не рекомендуется (лог URL)
```

### OAuth 2.0

```
Authorization: Bearer <access_token>  # от OAuth provider
```

## Пагинация

```bash
# Cursor-based (рекомендуется для больших данных)
GET /users?after=user_abc123&limit=20
# Ответ: { data: [...], next_cursor: "user_xyz789" }

# Offset-based (удобнее, но проблемы при вставке)
GET /users?page=3&per_page=20
GET /users?offset=40&limit=20

# Keyset pagination (эффективно в БД)
GET /users?created_after=2024-01-15&limit=20
```

## Кэширование

```bash
# ETags — идентификатор версии ресурса
GET /users/42
Response: ETag: "abc123"

# Условный запрос
GET /users/42
If-None-Match: "abc123"
Response: 304 Not Modified (если не изменился)

# Cache-Control
Cache-Control: max-age=3600, must-revalidate  # кэшировать 1 час
Cache-Control: no-cache    # всегда проверять свежесть
Cache-Control: no-store    # не кэшировать вообще
Cache-Control: private     # только браузер (не CDN)
```

## HATEOAS (Hypermedia as the Engine of Application State)

Ответ содержит ссылки на возможные действия:

```json
{
  "id": 42,
  "status": "pending",
  "_links": {
    "self": { "href": "/orders/42" },
    "cancel": { "href": "/orders/42/cancel", "method": "DELETE" },
    "items": { "href": "/orders/42/items" }
  }
}
```

## REST API в Go

```go
// Простой REST сервер с net/http
package main

import (
    "encoding/json"
    "net/http"
    "strconv"
    "strings"
)

type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

func usersHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")

    parts := strings.Split(r.URL.Path, "/")
    
    switch r.Method {
    case http.MethodGet:
        if len(parts) == 3 && parts[2] != "" {
            // GET /users/:id
            id, _ := strconv.Atoi(parts[2])
            user := findUser(id)
            if user == nil {
                http.Error(w, `{"error": "not found"}`, http.StatusNotFound)
                return
            }
            json.NewEncoder(w).Encode(user)
        } else {
            // GET /users
            json.NewEncoder(w).Encode(listUsers())
        }
    case http.MethodPost:
        var user User
        if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
            http.Error(w, `{"error": "invalid json"}`, http.StatusBadRequest)
            return
        }
        created := createUser(user)
        w.WriteHeader(http.StatusCreated)
        json.NewEncoder(w).Encode(created)
    case http.MethodDelete:
        id, _ := strconv.Atoi(parts[2])
        deleteUser(id)
        w.WriteHeader(http.StatusNoContent)
    }
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/users", usersHandler)
    mux.HandleFunc("/users/", usersHandler)
    http.ListenAndServe(":8080", mux)
}
```

```go
// С chi router (популярный, легковесный)
import "github.com/go-chi/chi/v5"

r := chi.NewRouter()
r.Use(middleware.Logger)
r.Use(middleware.Recoverer)
r.Use(middleware.RequestID)

r.Get("/users", listUsers)
r.Post("/users", createUser)
r.Route("/users/{id}", func(r chi.Router) {
    r.Get("/", getUser)
    r.Put("/", updateUser)
    r.Delete("/", deleteUser)
    r.Get("/orders", getUserOrders)
})

// Получить URL параметр
id := chi.URLParam(r, "id")
```

```go
// Middleware: аутентификация JWT
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := strings.TrimPrefix(r.Header.Get("Authorization"), "Bearer ")
        if token == "" {
            http.Error(w, `{"error": "unauthorized"}`, http.StatusUnauthorized)
            return
        }
        claims, err := validateJWT(token)
        if err != nil {
            http.Error(w, `{"error": "invalid token"}`, http.StatusUnauthorized)
            return
        }
        ctx := context.WithValue(r.Context(), userKey, claims)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

## REST vs GraphQL vs gRPC

| Критерий | REST | GraphQL | gRPC |
|----------|------|---------|------|
| Протокол | HTTP | HTTP | HTTP/2 |
| Формат | JSON | JSON | Protobuf |
| Типизация | Нет (OpenAPI) | Схема | Proto schema |
| Over-fetching | Да | Нет | Нет |
| Браузер | Нативно | JS клиент | Нужен grpc-web |
| Стриминг | Нет | Subscriptions | Да |
| Скорость | Medium | Medium | **Высокая** |
| Зрелость | **Высокая** | Medium | Высокая |
| Use case | Public API | Frontend BFF | Микросервисы |

## OpenAPI (Swagger)

```yaml
openapi: 3.0.3
info:
  title: User API
  version: 1.0.0

paths:
  /users:
    get:
      summary: List users
      parameters:
        - name: page
          in: query
          schema: { type: integer }
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                type: array
                items: { $ref: '#/components/schemas/User' }

  /users/{id}:
    get:
      parameters:
        - name: id
          in: path
          required: true
          schema: { type: integer }
      responses:
        '200': { description: User found }
        '404': { description: Not found }

components:
  schemas:
    User:
      type: object
      properties:
        id: { type: integer }
        name: { type: string }
        email: { type: string, format: email }
```

## Что спрашивают на собеседованиях

1. **Чем REST отличается от RPC?** → REST — ресурс-ориентированный (существительные в URL), RPC — операция-ориентированный (глаголы).
2. **Что такое idempotent?** → Повторный вызов с теми же данными даёт тот же результат. PUT, DELETE — идемпотентны; POST — нет.
3. **401 vs 403?** → 401: нет аутентификации (нет токена). 403: есть токен, но нет прав.
4. **Как делать пагинацию?** → Cursor-based предпочтительнее offset для больших данных (нет проблемы пропуска при вставке).
5. **Как версионировать API?** → URL versioning `/v1/`, header, query param. URL — наиболее очевидно.
6. **Что такое HATEOAS?** → Ответ содержит ссылки на допустимые следующие действия. Редко реализуется полностью.
7. **Чем REST хуже gRPC?** → JSON медленнее Protobuf; нет типизации без OpenAPI; нет stremig; over-fetching.
