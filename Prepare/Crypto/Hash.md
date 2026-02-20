# Хеш-функции (Hash Functions)

Полезные ссылки:
- [crypto/sha256 — Go stdlib](https://pkg.go.dev/crypto/sha256)
- [crypto/sha512 — Go stdlib](https://pkg.go.dev/crypto/sha512)
- [NIST — Cryptographic Hash Functions](https://csrc.nist.gov/projects/hash-functions)

---

## Что такое хеш-функция?

**Криптографическая хеш-функция** — функция, которая преобразует данные произвольного размера в фиксированный "отпечаток" (хеш, дайджест).

**Свойства криптографической хеш-функции:**

1. **Детерминированность** — одинаковые входные данные → всегда одинаковый хеш.
2. **Быстрое вычисление** — легко вычислить хеш по данным.
3. **Устойчивость к прообразу** (preimage resistance) — по хешу нельзя восстановить данные.
4. **Устойчивость ко второму прообразу** — нельзя найти другие данные с тем же хешем.
5. **Устойчивость к коллизиям** — нельзя найти два разных сообщения с одинаковым хешем.
6. **Лавинный эффект** — малейшее изменение входа → кардинально другой хеш.

```
sha256("Hello")  = 185f8db32921bd46d35ccabbbbb06727...
sha256("Hello!") = 334d016f755cd6dc58c53a86e183882f... ← кардинально другой
```

## Алгоритмы

### MD5 (128 бит)

**Устарел и небезопасен** для криптографии. Коллизии найдены. Используется только для checksums файлов (некриптографическое использование).

```go
import "crypto/md5"
hash := md5.Sum([]byte("data"))
fmt.Printf("%x\n", hash)
```

### SHA-1 (160 бит)

**Устарел.** Google доказал collision attack в 2017 (SHAttered). Не использовать в новых системах.

### SHA-2 (256/384/512 бит)

Текущий стандарт. Семейство включает SHA-224, SHA-256, SHA-384, SHA-512.

```go
import (
    "crypto/sha256"
    "crypto/sha512"
    "fmt"
)

// SHA-256 (наиболее распространён)
hash := sha256.Sum256([]byte("hello"))
fmt.Printf("%x\n", hash) // 2cf24dba5fb0a30e...

// SHA-512 (быстрее на 64-битных CPU из-за 64-битных операций)
hash512 := sha512.Sum512([]byte("hello"))
fmt.Printf("%x\n", hash512)

// Для больших данных — через io.Writer
h := sha256.New()
h.Write([]byte("chunk1"))
h.Write([]byte("chunk2"))
result := h.Sum(nil) // []byte
fmt.Printf("%x\n", result)
```

### SHA-3 (Keccak, Go 1.24+)

Другой дизайн (sponge construction), не связан с SHA-2 уязвимостями:

```go
import "crypto/sha3"

hash := sha3.Sum256([]byte("hello"))
```

### BLAKE2b / BLAKE3

Быстрее SHA-2, при сравнимой безопасности. Используется в современных системах.

```go
import "golang.org/x/crypto/blake2b"

h, _ := blake2b.New256(nil) // без ключа
h.Write([]byte("data"))
hash := h.Sum(nil)
```

## Применение хеш-функций

### Проверка целостности файлов

```go
func fileHash(path string) (string, error) {
    f, err := os.Open(path)
    if err != nil {
        return "", err
    }
    defer f.Close()

    h := sha256.New()
    if _, err := io.Copy(h, f); err != nil {
        return "", err
    }
    return fmt.Sprintf("%x", h.Sum(nil)), nil
}
```

### HMAC (Hash-based Message Authentication Code)

HMAC = хеш с секретным ключом. Гарантирует **целостность и аутентичность**.

```go
import "crypto/hmac"

key := []byte("secret-key")
message := []byte("important data")

// Создать
mac := hmac.New(sha256.New, key)
mac.Write(message)
signature := mac.Sum(nil) // []byte

// Проверить (constant-time comparison — защита от timing attack)
mac2 := hmac.New(sha256.New, key)
mac2.Write(message)
valid := hmac.Equal(mac2.Sum(nil), signature) // true
```

**Применение HMAC:** JWT подписи, API webhooks, подписи cookies.

### Хранение паролей

**Нельзя** хранить пароли как SHA256(password) — уязвимо к rainbow tables и brute force.

**Нужно** использовать специализированные функции для паролей:

```go
import "golang.org/x/crypto/bcrypt"

// Хеширование пароля (cost = 10-12 рекомендуется)
hash, err := bcrypt.GenerateFromPassword([]byte("mypassword"), bcrypt.DefaultCost)
// $2a$10$... (включает соль и cost-фактор)

// Проверка
err = bcrypt.CompareHashAndPassword(hash, []byte("mypassword"))
if err == nil {
    // пароль верный
}
```

**Альтернативы bcrypt:**
- **Argon2** (рекомендуется OWASP) — `golang.org/x/crypto/argon2`.
- **scrypt** — `golang.org/x/crypto/scrypt`.
- **PBKDF2** — `crypto/pbkdf2` (Go 1.24+).

Почему они лучше SHA256 для паролей:
- Намеренно **медленные** (adjustable cost) — brute force дороже.
- Встроена **соль** (salt) — защита от rainbow tables.
- **Memory-hard** (Argon2, scrypt) — защита от GPU атак.

```go
import "golang.org/x/crypto/argon2"

func hashPassword(password string) (string, error) {
    salt := make([]byte, 16)
    rand.Read(salt)

    // argon2id — рекомендуется (защита от side-channel)
    hash := argon2.IDKey([]byte(password), salt,
        1,       // time (iterations)
        64*1024, // memory (KB) — 64 MB
        4,       // threads
        32,      // keyLen
    )

    // Хранить: base64(salt) + "$" + base64(hash)
    return encodeParams(salt, hash), nil
}
```

## Хеш-функции и структуры данных

### Bloom Filter

Вероятностная структура данных на основе хешей:
- "Элемент точно НЕ содержится" — всегда верно.
- "Элемент содержится" — с некоторой вероятностью ложноположительный результат.

```go
import "github.com/bits-and-blooms/bloom/v3"

filter := bloom.New(1000000, 5) // 1M элементов, 5 хеш-функций
filter.Add([]byte("user1@example.com"))

if filter.Test([]byte("user1@example.com")) {
    // вероятно, содержится
}
```

Применение: ускорение поиска в БД, фильтрация спама, кэши.

## Сравнение алгоритмов

| Алгоритм | Размер хеша | Безопасен? | Скорость | Применение |
|----------|-------------|-----------|---------|-----------|
| MD5 | 128 бит | **Нет** | Быстрый | Только checksums |
| SHA-1 | 160 бит | **Нет** | Быстрый | Устарел |
| SHA-256 | 256 бит | Да | Хороший | Общее назначение |
| SHA-512 | 512 бит | Да | Быстрее на 64-бит | Когда нужно > 256 бит |
| SHA-3-256 | 256 бит | Да | Медленнее | Дополнительная безопасность |
| BLAKE2b | 256/512 бит | Да | Быстрее SHA-2 | Современные системы |
| bcrypt | 184 бит | Да | Намеренно медленный | Пароли |
| Argon2id | конфигурируемый | Да | Намеренно медленный | Пароли (OWASP) |

## Атаки на хеши

| Атака | Описание | Защита |
|-------|---------|--------|
| Brute force | Перебор паролей | Медленные хеш-функции (bcrypt/argon2) |
| Rainbow tables | Предвычисленные хеши | Соль (salt) |
| Length extension | Для SHA-1/2 (Merkle-Damgård) | HMAC, SHA-3 |
| Collision | Два разных сообщения → один хеш | SHA-256+ |
| Timing attack | Сравнение хешей по времени | `crypto/hmac.Equal()` (constant time) |

## Что спрашивают на собеседованиях

1. **Что такое хеш-функция?** → Одностороннее преобразование данных в фиксированный отпечаток.
2. **Зачем соль (salt) в хешировании паролей?** → Защита от rainbow tables — одинаковые пароли дают разные хеши.
3. **Почему MD5/SHA-1 нельзя для паролей?** → Слишком быстрые → brute force реален; коллизии найдены.
4. **Что такое HMAC?** → Хеш с секретным ключом — гарантирует целостность и аутентичность.
5. **Bcrypt vs SHA-256 для паролей?** → Bcrypt намеренно медленный, встроена соль — bcrypt правильный выбор.
6. **Лавинный эффект?** → Малейшее изменение → кардинально другой хеш.
7. **Timing attack при проверке хеша?** → Использовать `hmac.Equal` вместо `==` — constant-time сравнение.
