# Криптография

Материалы по криптографии для подготовки к собеседованиям Senior Go Developer.

## Содержание

- [Hash](Hash.md) — хеш-функции (SHA-256, bcrypt, Argon2, HMAC)
- [RSA](RSA.md) — асимметричное шифрование, RSA, ECDSA, Ed25519, пост-квантум
- [TLS](TLS.md) — TLS 1.3, mTLS, сертификаты, цепочка доверия
- [SSH](SSH.md) — SSH протокол, ключи, туннелирование, Go SSH клиент/сервер
- [EncodeDecode](EncodeDecode.md) — PEM/DER форматы, PKCS#8, цифровые подписи
- [Webauthn](Webauthn.md) — WebAuthn/FIDO2, passkeys, беспарольная аутентификация
- [JWKS](JWKS.md) — JSON Web Key Sets, ротация ключей, верификация JWT

## Быстрая шпаргалка

### Что использовать для...

| Задача | Решение |
|--------|---------|
| Хеш общего назначения | SHA-256 (`crypto/sha256`) |
| Хеш паролей | Argon2id (`golang.org/x/crypto/argon2`) |
| Подпись сообщений | Ed25519 (`crypto/ed25519`) |
| Шифрование данных | AES-256-GCM (`crypto/aes` + `crypto/cipher`) |
| HMAC (аутентификация) | HMAC-SHA256 (`crypto/hmac`) |
| TLS клиент | `crypto/tls` с `tls.VersionTLS13` |
| SSH клиент/сервер | `golang.org/x/crypto/ssh` |
| Беспарольный вход | WebAuthn `github.com/go-webauthn/webauthn` |
| Ключевой обмен | X25519 через `crypto/ecdh` |
| JWT верификация (микросервисы) | JWKS + ES256 `github.com/lestrrat-go/jwx/v2` |

### Что НЕ использовать

| Нельзя | Почему |
|--------|--------|
| MD5 для безопасности | Коллизии найдены |
| SHA-1 | Google SHAttered (2017) |
| SHA-256 для паролей | Слишком быстрый → brute force |
| ECB режим AES | Паттерны видны |
| RSA без OAEP/PSS padding | Уязвимости |
| `InsecureSkipVerify: true` в prod | MITM атаки |
| Самописная криптография | Ошибки неизбежны |

### AES-256-GCM — симметричное шифрование

```go
import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
)

func encrypt(key, plaintext []byte) ([]byte, error) {
    block, err := aes.NewCipher(key) // key должен быть 32 байта для AES-256
    if err != nil {
        return nil, err
    }

    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return nil, err
    }

    nonce := make([]byte, gcm.NonceSize()) // 12 байт
    rand.Read(nonce)

    // Seal: nonce + зашифрованные данные + authentication tag
    ciphertext := gcm.Seal(nonce, nonce, plaintext, nil)
    return ciphertext, nil
}

func decrypt(key, ciphertext []byte) ([]byte, error) {
    block, _ := aes.NewCipher(key)
    gcm, _ := cipher.NewGCM(block)

    nonceSize := gcm.NonceSize()
    nonce, ciphertext := ciphertext[:nonceSize], ciphertext[nonceSize:]
    return gcm.Open(nil, nonce, ciphertext, nil) // проверяет аутентичность
}
```

### Генерация безопасных случайных данных

```go
import "crypto/rand"

// Безопасный случайный ключ
key := make([]byte, 32)
rand.Read(key)

// Безопасный токен
token := make([]byte, 32)
rand.Read(token)
tokenStr := base64.URLEncoding.EncodeToString(token)
```
