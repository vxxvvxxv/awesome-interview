# Protocol Buffers (Protobuf)

Полезные ссылки:
- [Официальная документация](https://protobuf.dev/)
- [proto3 Language Guide](https://protobuf.dev/programming-guides/proto3/)
- [Encoding Reference](https://protobuf.dev/programming-guides/encoding/)

---

## Что такое Protocol Buffers?

**Protocol Buffers** (Protobuf) — бинарный формат сериализации данных от Google. Используется как основной формат в gRPC, но может применяться независимо.

**Преимущества перед JSON:**
- **Размер**: в 2–10 раз меньше.
- **Скорость**: в 5–10 раз быстрее парсинг.
- **Типизация**: строгая схема, ошибки на этапе компиляции.
- **Эволюция схемы**: обратная и прямая совместимость.

**Недостатки:**
- Не читается человеком (бинарный).
- Нужна схема (.proto) для десериализации.
- Хуже для публичных API (нет self-describing).

## Синтаксис proto3

```protobuf
syntax = "proto3";

package example;
option go_package = "github.com/example/gen;examplepb";

// Импорт
import "google/protobuf/timestamp.proto";
import "google/protobuf/any.proto";

message User {
    int64  id        = 1;          // поле с тегом 1
    string name      = 2;
    string email     = 3;
    bool   is_active = 4;
    repeated string tags = 5;      // список
    Address address  = 6;          // вложенное сообщение
    google.protobuf.Timestamp created_at = 7;
}

message Address {
    string city    = 1;
    string country = 2;
}

// Enum
enum Status {
    STATUS_UNKNOWN  = 0;  // 0 — обязательное нулевое значение
    STATUS_ACTIVE   = 1;
    STATUS_INACTIVE = 2;
}

// Сервис (для gRPC)
service UserService {
    rpc GetUser(GetUserRequest) returns (User);
}

message GetUserRequest { int64 id = 1; }
```

## Типы данных

| Protobuf тип | Go тип | Описание |
|-------------|--------|---------|
| `int32`, `int64` | `int32`, `int64` | Числа со знаком |
| `uint32`, `uint64` | `uint32`, `uint64` | Без знака |
| `sint32`, `sint64` | `int32`, `int64` | Эффективнее для отрицательных |
| `fixed32`, `fixed64` | `uint32`, `uint64` | Фиксированный размер (быстрее для больших чисел) |
| `float`, `double` | `float32`, `float64` | Числа с плавающей точкой |
| `bool` | `bool` | |
| `string` | `string` | UTF-8 |
| `bytes` | `[]byte` | Произвольные байты |
| `message` | struct | Вложенное сообщение |
| `repeated T` | `[]T` | Список |
| `map<K, V>` | `map[K]V` | Словарь |

## Кодирование (wire format)

Protobuf сериализует каждое поле как `(field_number << 3) | wire_type` + данные.

### Wire Types

| Wire Type | Значение | Используется для |
|-----------|----------|-----------------|
| Varint | 0 | int32, int64, uint32, uint64, sint32, sint64, bool, enum |
| 64-bit | 1 | fixed64, sfixed64, double |
| Length-delimited | 2 | string, bytes, embedded messages, repeated |
| 32-bit | 5 | fixed32, sfixed32, float |

### Varint (переменная длина)

Числа кодируются компактно — маленькие числа занимают меньше байт:

```
0   → 1 байт (0x00)
127 → 1 байт (0x7F)
128 → 2 байта (0x80 0x01)
300 → 2 байта (0xAC 0x02)
```

**Принцип**: 7 бит на данные, 1 бит (MSB) — признак продолжения.

### Почему int32 плох для отрицательных чисел?

```
-1 (int32) → кодируется как uint64 → 10 байт всегда
-1 (sint32) → zigzag encoding → 1 байт
```

**ZigZag encoding**: `0 → 0`, `-1 → 1`, `1 → 2`, `-2 → 3`, `2 → 4` ...

Используй `sint32`/`sint64` если значения часто отрицательные.

## Пример бинарного кодирования

```
message User {
    int64  id   = 1;   // field 1
    string name = 2;   // field 2
}

User{Id: 150, Name: "Alice"}
```

```
08 96 01      // field=1, type=varint, value=150
12 05 416c696365  // field=2, type=length-delimited, len=5, "Alice"
```

## Эволюция схемы (совместимость)

### Обратная совместимость (backward)

Старый код читает новые данные (с новыми полями):
- Неизвестные поля **игнорируются**.
- Новые обязательные поля → нельзя добавить в proto3 (все поля опциональны).

### Прямая совместимость (forward)

Новый код читает старые данные:
- Отсутствующие поля → нулевые значения (`0`, `""`, `false`, `nil`).

### Правила безопасного изменения

✅ **Можно:**
- Добавить новое поле с новым номером.
- Удалить поле (зарезервировать номер через `reserved`).
- Переименовать поле (имя не кодируется, только номер).
- Изменить `int32` ↔ `int64` (совместимы по wire type).

❌ **Нельзя:**
- Изменить номер существующего поля.
- Изменить тип поля несовместимо (`string` → `int32`).
- Использовать номера удалённых полей для новых полей.

```protobuf
message User {
    reserved 4, 5;         // зарезервированные номера (нельзя использовать)
    reserved "old_field";  // зарезервированные имена

    int64  id   = 1;
    string name = 2;
    // поле 3 удалено
    string email = 6;  // новое поле — новый номер
}
```

## oneof (union типы)

```protobuf
message Notification {
    string user_id = 1;
    oneof content {
        EmailNotification email  = 2;
        SMSNotification   sms    = 3;
        PushNotification  push   = 4;
    }
}
```

```go
// Go
switch n := notif.Content.(type) {
case *pb.Notification_Email:
    sendEmail(n.Email)
case *pb.Notification_Sms:
    sendSMS(n.Sms)
}
```

## map в Protobuf

```protobuf
message Config {
    map<string, string> labels    = 1;
    map<string, int32>  counters  = 2;
}
```

Ограничения map:
- Ключ: любой скалярный тип (кроме float, double, bytes).
- Порядок итерации не гарантирован.
- Нельзя использовать `repeated`.

## Well-Known Types

```protobuf
import "google/protobuf/timestamp.proto";
import "google/protobuf/duration.proto";
import "google/protobuf/any.proto";
import "google/protobuf/struct.proto";
import "google/protobuf/wrappers.proto";
import "google/protobuf/empty.proto";
```

```go
// Timestamp
ts := timestamppb.Now()
ts := timestamppb.New(time.Now())
t := ts.AsTime() // time.Time

// Duration
d := durationpb.New(5 * time.Second)
dur := d.AsDuration() // time.Duration

// Any (произвольное сообщение)
anyVal, _ := anypb.New(&pb.User{Id: 1})
var user pb.User
anyVal.UnmarshalTo(&user)

// Empty (нет параметров / нет возврата)
rpc DeleteUser(DeleteUserRequest) returns (google.protobuf.Empty);

// Nullable обёртки (wrappers)
import "google/protobuf/wrappers.proto";
message Profile {
    google.protobuf.StringValue nickname = 1; // nullable string
}
```

## Использование в Go

```go
package main

import (
    "google.golang.org/protobuf/proto"
    pb "github.com/example/gen"
)

func main() {
    user := &pb.User{
        Id:   42,
        Name: "Alice",
        Tags: []string{"admin", "user"},
    }

    // Сериализация
    data, err := proto.Marshal(user)
    if err != nil {
        panic(err)
    }

    // Десериализация
    var decoded pb.User
    if err := proto.Unmarshal(data, &decoded); err != nil {
        panic(err)
    }

    // Сравнение
    proto.Equal(user, &decoded) // true

    // Клонирование
    clone := proto.Clone(user).(*pb.User)
}
```

## JSON ↔ Protobuf

```go
import "google.golang.org/protobuf/encoding/protojson"

// Proto → JSON
jsonBytes, _ := protojson.Marshal(user)
// {"id":"42","name":"Alice","tags":["admin","user"]}

// JSON → Proto
var user pb.User
protojson.Unmarshal(jsonBytes, &user)

// Настройки
marshaler := protojson.MarshalOptions{
    EmitUnpopulated: true,     // включить нулевые поля
    UseProtoNames:   true,     // snake_case вместо camelCase
    Indent:          "  ",
}
```

## Protobuf vs JSON vs MessagePack

| Характеристика | JSON | MessagePack | Protobuf |
|----------------|------|-------------|---------|
| Читаемость | Да | Нет | Нет |
| Self-describing | Да | Частично | Нет (нужна схема) |
| Размер | Большой | Средний | Маленький |
| Скорость парсинга | Медленная | Быстрая | Очень быстрая |
| Типизация | Слабая | Слабая | Строгая |
| Совместимость схем | Нет | Нет | Встроена |
| Генерация кода | Нет | Нет | Да |
