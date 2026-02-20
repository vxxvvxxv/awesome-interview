# Тестирование в Go

Полезные ссылки:
- [testing — стандартная библиотека](https://pkg.go.dev/testing)
- [testify — GitHub](https://github.com/testify-go/testify)
- [gomock — GitHub](https://github.com/uber-go/mock)

---

## Философия тестирования

**Пирамида тестирования:**

```
           /\
          /  \    E2E тесты (мало, дорогие)
         /----\
        /      \  Integration тесты (средне)
       /--------\
      /          \ Unit тесты (много, быстрые)
     /____________\
```

- **Unit**: тестируем одну функцию/метод в изоляции (моки зависимостей).
- **Integration**: тестируем взаимодействие компонентов (реальная БД, реальные сервисы).
- **E2E**: тестируем весь сценарий как пользователь.

## Стандартный пакет `testing`

```go
// user_test.go
package user_test // _test суффикс — blackbox тестирование

import (
    "testing"
    "github.com/example/user"
)

func TestGetUser(t *testing.T) {
    got := user.Get(42)
    want := "Alice"

    if got.Name != want {
        t.Errorf("Get(42).Name = %q; want %q", got.Name, want)
    }
}
```

```bash
go test ./...           # все тесты
go test -v ./...        # verbose
go test -run TestGetUser # конкретный тест
go test -count=1 ./...  # без кэша
go test -race ./...     # детектор гонок данных
```

## Table-driven тесты

Стандартный Go-паттерн для тестирования множества сценариев:

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name string
        a, b int
        want int
    }{
        {"positive", 1, 2, 3},
        {"negative", -1, -2, -3},
        {"zero", 0, 0, 0},
        {"overflow", math.MaxInt, 1, math.MinInt}, // ожидаем overflow
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Add(tt.a, tt.b)
            if got != tt.want {
                t.Errorf("Add(%d, %d) = %d; want %d", tt.a, tt.b, got, tt.want)
            }
        })
    }
}
```

```bash
go test -run TestAdd/positive  # запустить конкретный подтест
```

## Хелперы тестирования

```go
func TestSomething(t *testing.T) {
    t.Helper() // помечает как helper — трассировка указывает на вызывающего

    // Пропуск теста
    t.Skip("пропускаем пока не исправим баг")
    t.Skipf("пропускаем: %s", reason)

    // Немедленный провал
    t.Fatal("критическая ошибка") // аналог t.Error + t.FailNow
    t.Fatalf("ошибка: %v", err)

    // Параллельный запуск
    t.Parallel() // тест запускается параллельно с другими Parallel-тестами
}
```

## testify — популярная библиотека

```go
import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestUser(t *testing.T) {
    user, err := GetUser(42)

    // require — останавливает тест при провале (Fatal)
    require.NoError(t, err)
    require.NotNil(t, user)

    // assert — продолжает тест при провале (Error)
    assert.Equal(t, int64(42), user.ID)
    assert.Equal(t, "Alice", user.Name)
    assert.True(t, user.IsActive)
    assert.Contains(t, user.Tags, "admin")
    assert.Len(t, user.Tags, 2)
    assert.Empty(t, user.DeletedAt)
    assert.WithinDuration(t, time.Now(), user.CreatedAt, time.Second)
}
```

### testify/suite

```go
import "github.com/stretchr/testify/suite"

type UserSuite struct {
    suite.Suite
    db *sql.DB
}

func (s *UserSuite) SetupSuite() {
    s.db = connectDB() // один раз на весь suite
}

func (s *UserSuite) TearDownSuite() {
    s.db.Close()
}

func (s *UserSuite) SetupTest() {
    // перед каждым тестом
    s.db.Exec("BEGIN")
}

func (s *UserSuite) TearDownTest() {
    s.db.Exec("ROLLBACK") // откат после каждого теста
}

func (s *UserSuite) TestGetUser() {
    user, err := GetUser(s.db, 42)
    s.Require().NoError(err)
    s.Equal("Alice", user.Name)
}

func TestUserSuite(t *testing.T) {
    suite.Run(t, new(UserSuite))
}
```

## Моки (Mocks)

Мок заменяет реальную зависимость в тестах — позволяет контролировать её поведение.

### Интерфейс для мокинга

```go
// Реальный интерфейс
type UserRepository interface {
    GetByID(ctx context.Context, id int64) (*User, error)
    Save(ctx context.Context, user *User) error
}

// Сервис зависит от интерфейса, а не от конкретной реализации
type UserService struct {
    repo UserRepository
}
```

### gomock (uber-go/mock)

```bash
go install go.uber.org/mock/mockgen@latest
mockgen -source=repository.go -destination=mocks/mock_repository.go
```

```go
import (
    "testing"
    "go.uber.org/mock/gomock"
    "github.com/example/mocks"
)

func TestUserService_GetUser(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    mockRepo := mocks.NewMockUserRepository(ctrl)

    // Настройка ожиданий
    mockRepo.EXPECT().
        GetByID(gomock.Any(), int64(42)).
        Return(&User{ID: 42, Name: "Alice"}, nil).
        Times(1)

    svc := &UserService{repo: mockRepo}
    user, err := svc.GetUser(context.Background(), 42)

    assert.NoError(t, err)
    assert.Equal(t, "Alice", user.Name)
}
```

### testify/mock

```go
import "github.com/stretchr/testify/mock"

type MockUserRepo struct {
    mock.Mock
}

func (m *MockUserRepo) GetByID(ctx context.Context, id int64) (*User, error) {
    args := m.Called(ctx, id)
    return args.Get(0).(*User), args.Error(1)
}

// Использование
mockRepo := new(MockUserRepo)
mockRepo.On("GetByID", mock.Anything, int64(42)).Return(&User{Name: "Alice"}, nil)

// После теста
mockRepo.AssertExpectations(t)
```

## Бенчмарки

```go
func BenchmarkAdd(b *testing.B) {
    // Go 1.24+: предпочтительный способ
    b.Loop(func() {
        Add(1, 2)
    })
}

// Старый способ (до 1.24)
func BenchmarkAddOld(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(1, 2)
    }
}

// С подготовкой
func BenchmarkJSON(b *testing.B) {
    data := generateBigStruct() // не включается в замер
    b.ResetTimer()

    b.Loop(func() {
        json.Marshal(data)
    })
}
```

```bash
go test -bench=. -benchmem ./...
# BenchmarkAdd-8   1000000000   0.25 ns/op   0 B/op   0 allocs/op
```

## Fuzzing (Go 1.18+)

```go
func FuzzParseURL(f *testing.F) {
    // Seed corpus — начальные входные данные
    f.Add("https://example.com/path?q=1")
    f.Add("http://localhost:8080")

    f.Fuzz(func(t *testing.T, input string) {
        url, err := ParseURL(input)
        if err != nil {
            return // ошибка допустима
        }
        // Инварианты — что всегда должно быть верно
        if url.Host == "" {
            t.Error("parsed URL has empty host")
        }
    })
}
```

```bash
go test -fuzz=FuzzParseURL -fuzztime=30s
```

## Тестирование HTTP-обработчиков

```go
import "net/http/httptest"

func TestGetUserHandler(t *testing.T) {
    // Создаём тестовый запрос
    req := httptest.NewRequest(http.MethodGet, "/users/42", nil)
    w := httptest.NewRecorder()

    // Вызываем хендлер напрямую
    handler := NewUserHandler(mockRepo)
    handler.ServeHTTP(w, req)

    // Проверяем ответ
    resp := w.Result()
    assert.Equal(t, http.StatusOK, resp.StatusCode)

    var user User
    json.NewDecoder(resp.Body).Decode(&user)
    assert.Equal(t, int64(42), user.ID)
}
```

## Тестирование с реальной БД (testcontainers)

```go
import "github.com/testcontainers/testcontainers-go"

func TestWithPostgres(t *testing.T) {
    ctx := context.Background()

    container, err := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:16"),
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
    )
    require.NoError(t, err)
    defer container.Terminate(ctx)

    dsn, _ := container.ConnectionString(ctx, "sslmode=disable")
    db, _ := sql.Open("postgres", dsn)

    // Мигрируем схему
    runMigrations(db)

    // Тесты с реальной БД
    repo := NewPostgresUserRepo(db)
    user, err := repo.GetByID(ctx, 1)
    // ...
}
```

## Покрытие кода (Coverage)

```bash
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out  # HTML отчёт в браузере
go tool cover -func=coverage.out  # по функциям в терминале

# % покрытия конкретного пакета
go test -cover ./internal/user/...
```

## Race Detector

```bash
go test -race ./...
go run -race main.go
```

```go
// Пример гонки данных, которую поймает -race
var counter int

func TestRace(t *testing.T) {
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++ // DATA RACE! Нужен atomic или mutex
        }()
    }
    wg.Wait()
}
```

## TestMain — инициализация пакета

```go
func TestMain(m *testing.M) {
    // Настройка перед всеми тестами
    setupDB()
    setupLogger()

    code := m.Run() // запустить все тесты

    // Очистка после всех тестов
    teardownDB()

    os.Exit(code)
}
```

## Лучшие практики

1. **Называй тесты** по паттерну: `TestFunctionName_Scenario_ExpectedResult`.
2. **Table-driven tests** для множества кейсов.
3. **Тестируй поведение, не реализацию** — если можешь изменить внутреннюю реализацию не ломая тесты, тесты хорошие.
4. **Один assert на тест** (или несколько связанных) — легче находить провал.
5. **`t.Parallel()`** для независимых тестов — ускоряет CI.
6. **Всегда проверяй ошибки через `require`** перед дальнейшими assert.
7. **`-race` в CI** — ловит гонки которые сложно воспроизвести.
8. **Моки только для внешних зависимостей** (БД, HTTP-сервисы), не для внутренней логики.
