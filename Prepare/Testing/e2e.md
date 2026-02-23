# End-to-End (E2E) тестирование

Полезные ссылки:
- [testcontainers-go](https://golang.testcontainers.org/)
- [chromedp (браузерная автоматизация)](https://github.com/chromedp/chromedp)
- [httptest](https://pkg.go.dev/net/http/httptest)
- [playwright-go](https://github.com/playwright-community/playwright-go)

---

## Что такое E2E тестирование?

**E2E (End-to-End)** — тестирование всей системы от начала до конца, как это делал бы реальный пользователь. Тестирует интеграцию всех компонентов: API, БД, кеш, брокер.

### Пирамида тестирования

```
         /\
        /E2E\           (мало, медленные, дорогие)
       /------\
      /Integr. \        (среднее количество)
     /------------\
    / Unit Tests   \    (много, быстрые, дешёвые)
   /__________________\
```

**Unit тесты:** тестируют функции изолированно (mock зависимости).  
**Интеграционные:** тестируют взаимодействие 2-3 компонентов.  
**E2E:** тестируют весь поток от пользователя до БД.

## E2E API тестирование

### С реальным HTTP сервером

```go
// main_test.go
package main_test

import (
    "net/http"
    "net/http/httptest"
    "testing"
    "encoding/json"
    "bytes"
)

// Запускаем реальный сервер на случайном порту
func TestCreateUser_E2E(t *testing.T) {
    srv := httptest.NewServer(setupRouter()) // роутер с реальными хендлерами
    defer srv.Close()

    // Создать пользователя
    body := `{"name": "Alice", "email": "alice@example.com"}`
    resp, err := http.Post(srv.URL+"/users", "application/json",
        bytes.NewBufferString(body))
    if err != nil {
        t.Fatal(err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusCreated {
        t.Errorf("expected 201, got %d", resp.StatusCode)
    }

    var user map[string]any
    json.NewDecoder(resp.Body).Decode(&user)
    if user["email"] != "alice@example.com" {
        t.Errorf("unexpected email: %v", user["email"])
    }
}
```

## Testcontainers-go: реальная БД в тестах

```go
import (
    "testing"
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/jackc/pgx/v5"
)

func TestUserRepo_E2E(t *testing.T) {
    ctx := context.Background()

    // Поднять PostgreSQL в Docker
    container, err := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:16-alpine"),
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
        testcontainers.WithWaitStrategy(
            wait.ForListeningPort("5432/tcp").WithStartupTimeout(30*time.Second),
        ),
    )
    if err != nil {
        t.Fatal(err)
    }
    defer container.Terminate(ctx)

    // Получить DSN
    dsn, _ := container.ConnectionString(ctx, "sslmode=disable")

    // Применить миграции
    pool, _ := pgxpool.New(ctx, dsn)
    pool.Exec(ctx, `CREATE TABLE users (id SERIAL PRIMARY KEY, name TEXT, email TEXT UNIQUE)`)

    // Создать репозиторий
    repo := NewUserRepo(pool)

    // Тест
    user, err := repo.Create(ctx, CreateUserInput{
        Name:  "Alice",
        Email: "alice@example.com",
    })
    if err != nil {
        t.Fatal(err)
    }
    if user.ID == 0 {
        t.Error("expected non-zero ID")
    }

    // Получить
    found, err := repo.FindByEmail(ctx, "alice@example.com")
    if err != nil {
        t.Fatal(err)
    }
    if found.Name != "Alice" {
        t.Errorf("expected Alice, got %s", found.Name)
    }
}
```

### Testcontainers с Redis и Kafka

```go
func setupTestEnv(t *testing.T) (pgDSN, redisAddr, kafkaBroker string) {
    t.Helper()
    ctx := context.Background()

    // PostgreSQL
    pgC, _ := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:16-alpine"),
        postgres.WithDatabase("testdb"),
    )
    t.Cleanup(func() { pgC.Terminate(ctx) })
    pgDSN, _ = pgC.ConnectionString(ctx, "sslmode=disable")

    // Redis
    redisC, _ := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: testcontainers.ContainerRequest{
            Image:        "redis:7-alpine",
            ExposedPorts: []string{"6379/tcp"},
            WaitingFor:   wait.ForListeningPort("6379/tcp"),
        },
        Started: true,
    })
    t.Cleanup(func() { redisC.Terminate(ctx) })
    redisHost, _ := redisC.Host(ctx)
    redisPort, _ := redisC.MappedPort(ctx, "6379")
    redisAddr = fmt.Sprintf("%s:%s", redisHost, redisPort.Port())

    return pgDSN, redisAddr, ""
}

func TestOrderService_E2E(t *testing.T) {
    pgDSN, redisAddr, _ := setupTestEnv(t)

    // Запустить приложение с реальными зависимостями
    app := NewApp(pgDSN, redisAddr)
    srv := httptest.NewServer(app.Handler())
    defer srv.Close()

    // Тест полного flow: создать заказ
    resp, _ := http.Post(srv.URL+"/orders", "application/json",
        strings.NewReader(`{"product_id": 1, "quantity": 2}`))
    // ...
}
```

## Полный интеграционный тест с Docker Compose

```go
// docker-compose.test.yml + запуск через go test
// Или TestMain для setup/teardown

func TestMain(m *testing.M) {
    // Setup: docker compose up
    cmd := exec.Command("docker", "compose", "-f", "docker-compose.test.yml", "up", "-d")
    cmd.Run()

    // Ждать готовности
    waitForService("localhost:5432")
    waitForService("localhost:6379")

    // Запустить тесты
    code := m.Run()

    // Teardown
    exec.Command("docker", "compose", "-f", "docker-compose.test.yml", "down").Run()
    os.Exit(code)
}
```

## Браузерная автоматизация (chromedp)

```go
import (
    "github.com/chromedp/chromedp"
    "testing"
)

func TestLoginFlow_Browser(t *testing.T) {
    ctx, cancel := chromedp.NewContext(context.Background())
    defer cancel()

    // Запустить реальный сервер
    srv := httptest.NewServer(app.Handler())
    defer srv.Close()

    var title string
    err := chromedp.Run(ctx,
        chromedp.Navigate(srv.URL+"/login"),
        chromedp.WaitVisible(`input[name="email"]`),
        chromedp.SendKeys(`input[name="email"]`, "alice@example.com"),
        chromedp.SendKeys(`input[name="password"]`, "secret"),
        chromedp.Click(`button[type="submit"]`),
        chromedp.WaitVisible(`#dashboard`),
        chromedp.Title(&title),
    )
    if err != nil {
        t.Fatal(err)
    }
    if !strings.Contains(title, "Dashboard") {
        t.Errorf("expected Dashboard, got %s", title)
    }
}
```

## Table-driven E2E тесты

```go
func TestUserCRUD_E2E(t *testing.T) {
    srv, db := setupTestServer(t) // testcontainers + httptest
    client := &http.Client{}

    tests := []struct {
        name       string
        method     string
        path       string
        body       string
        wantStatus int
        wantBody   string
    }{
        {"create user", "POST", "/users", `{"name":"Alice","email":"a@b.com"}`, 201, `"email":"a@b.com"`},
        {"get user",    "GET",  "/users/1", "", 200, `"name":"Alice"`},
        {"duplicate",  "POST", "/users", `{"name":"Bob","email":"a@b.com"}`, 409, `"CONFLICT"`},
        {"delete user", "DELETE", "/users/1", "", 204, ""},
        {"not found",  "GET",  "/users/1", "", 404, `"NOT_FOUND"`},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            req, _ := http.NewRequest(tt.method, srv.URL+tt.path,
                strings.NewReader(tt.body))
            req.Header.Set("Content-Type", "application/json")
            resp, err := client.Do(req)
            if err != nil {
                t.Fatal(err)
            }
            body, _ := io.ReadAll(resp.Body)
            resp.Body.Close()

            if resp.StatusCode != tt.wantStatus {
                t.Errorf("status: want %d, got %d", tt.wantStatus, resp.StatusCode)
            }
            if tt.wantBody != "" && !strings.Contains(string(body), tt.wantBody) {
                t.Errorf("body: want %q in %q", tt.wantBody, body)
            }
        })
    }
}
```

## Параллельные E2E тесты

```go
// Каждый тест создаёт свою БД (изоляция)
func TestOrder_E2E(t *testing.T) {
    t.Parallel() // можно запускать параллельно

    // Создать уникальную схему для этого теста
    schema := fmt.Sprintf("test_%s", strings.ReplaceAll(t.Name(), "/", "_"))
    db.Exec("CREATE SCHEMA " + schema)
    t.Cleanup(func() { db.Exec("DROP SCHEMA " + schema + " CASCADE") })

    // Запустить тесты в своей схеме
    // ...
}

// Запуск: go test -parallel=4 ./...
```

## Best Practices

```bash
# Запустить только E2E тесты (с тегом)
go test -tags=e2e ./...

# Или по имени
go test -run ".*E2E.*" ./...

# С таймаутом
go test -timeout 5m -run ".*E2E.*" ./...

# race detector (обязательно)
go test -race -timeout 5m ./...
```

```go
// Тег для разделения unit и e2e тестов
//go:build e2e

// e2e_test.go (запускается только с -tags=e2e)
```

### Состояние БД между тестами

```go
func cleanDB(t *testing.T, db *pgxpool.Pool) {
    t.Helper()
    tables := []string{"orders", "products", "users"}
    for _, table := range tables {
        db.Exec(context.Background(),
            fmt.Sprintf("TRUNCATE TABLE %s RESTART IDENTITY CASCADE", table))
    }
}

func TestSomething_E2E(t *testing.T) {
    db := getTestDB(t)
    cleanDB(t, db) // всегда начинаем с чистой БД
    defer cleanDB(t, db) // или defer

    // тест
}
```

## Flaky тесты — как бороться

```go
// Ретрай для нестабильных операций (не рекомендуется — лучше исправить причину)
func retryable(t *testing.T, maxTries int, fn func() error) {
    t.Helper()
    var err error
    for i := 0; i < maxTries; i++ {
        if err = fn(); err == nil {
            return
        }
        time.Sleep(100 * time.Millisecond * time.Duration(i+1))
    }
    t.Fatalf("failed after %d attempts: %v", maxTries, err)
}

// Ожидать асинхронные операции
func waitForCondition(t *testing.T, timeout time.Duration, fn func() bool) {
    t.Helper()
    deadline := time.Now().Add(timeout)
    for time.Now().Before(deadline) {
        if fn() {
            return
        }
        time.Sleep(50 * time.Millisecond)
    }
    t.Fatal("condition not met within timeout")
}
```

## Что спрашивают на собеседованиях

1. **Чем E2E отличается от интеграционных тестов?** → E2E тестирует весь стек от запроса до ответа (как пользователь). Интеграционные — взаимодействие конкретных компонентов.
2. **Testcontainers — зачем?** → Поднять реальную БД/Redis в Docker для тестов. Не нужно мокать; тесты реалистичны.
3. **Как изолировать E2E тесты в параллельном запуске?** → Уникальные схемы БД, уникальные namespace/таблицы для каждого теста.
4. **Flaky тесты — причины и решения?** → Гонки данных, timing issues, зависимость от внешних сервисов. Решение: детерминированный setup, ожидание условий вместо sleep.
5. **Как разделить unit и e2e тесты?** → Build tags (`//go:build e2e`), naming convention (`_e2e_test.go`), отдельные директории.
