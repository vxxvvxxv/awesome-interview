# RSA и асимметричная криптография

Полезные ссылки:
- [RFC 8017 — PKCS#1: RSA Cryptography Specifications](https://datatracker.ietf.org/doc/html/rfc8017)
- [RFC 8032 — Ed25519](https://datatracker.ietf.org/doc/html/rfc8032)
- [NIST Post-Quantum Cryptography](https://csrc.nist.gov/projects/post-quantum-cryptography)
- [crypto/rsa — Go stdlib](https://pkg.go.dev/crypto/rsa)
- [crypto/ecdsa — Go stdlib](https://pkg.go.dev/crypto/ecdsa)

---

## Симметричная vs Асимметричная криптография

### Симметричная

Один и тот же ключ для шифрования и расшифровки:

```
Alice                              Bob
  │                                  │
  │── зашифровано ключом K ─────────►│
  │                                  │ расшифровывает ключом K
```

**Проблема**: как безопасно передать ключ K по небезопасному каналу?

**Алгоритмы**: AES-256-GCM, ChaCha20-Poly1305.

### Асимметричная (Public Key Cryptography)

Два связанных ключа:
- **Public key** (открытый) — можно публиковать.
- **Private key** (закрытый) — только у владельца.

```
Свойство: то, что зашифровано public key → расшифруется только private key
          то, что подписано private key → проверяется public key
```

**Алгоритмы**: RSA, ECDSA, Ed25519, X25519.

**Применение**: TLS Handshake, SSH, GPG, JWT-подписи, сертификаты.

---

## RSA (Rivest–Shamir–Adleman, 1977)

### Математическая основа

RSA основан на **сложности факторизации большого числа**:
- Перемножить два больших простых числа — легко.
- Разложить произведение обратно — вычислительно невозможно при достаточном размере.

```
p = 61, q = 53       ← два случайных простых числа
n = p * q = 3233     ← модуль (публикуется)
```

### Генерация ключей RSA

```
1. Выбрать два больших простых числа: p, q
2. n = p * q                           (модуль, часть public и private key)
3. φ(n) = (p-1) * (q-1)              (функция Эйлера)
4. Выбрать e: 1 < e < φ(n), gcd(e, φ(n)) = 1   (обычно e = 65537)
5. d = e⁻¹ mod φ(n)                  (мультипликативный обратный)

Public key  = (n, e)
Private key = (n, d)   ← d хранится в секрете
```

Упрощённый пример:
```
p = 61, q = 53
n = 3233
φ(n) = 60 * 52 = 3120
e = 17   (взаимно просто с 3120)
d = 2753 (17 * 2753 ≡ 1 mod 3120)

Public key:  (3233, 17)
Private key: (3233, 2753)
```

### Шифрование и расшифровка

```
Шифрование:   C = M^e mod n
Расшифровка:  M = C^d mod n

Пример (M = 65):
  C = 65^17 mod 3233 = 2790
  M = 2790^2753 mod 3233 = 65 ✓
```

### Цифровая подпись RSA

```
Подпись:      S = hash(M)^d mod n      (подписываем hash, не сами данные)
Проверка:     hash(M) == S^e mod n
```

**Почему подписывают хеш, а не данные?**
- Данные могут быть большими — RSA работает только с числами < n.
- Хеш фиксированного размера (SHA-256 = 256 бит).

### Размеры ключей RSA

| Размер ключа | Безопасность | Применение |
|-------------|-------------|-----------|
| 1024 бит | **Небезопасен** (взломан) | Устарел |
| 2048 бит | ~112 бит безопасности | Минимум сегодня |
| 3072 бит | ~128 бит безопасности | Рекомендуется до 2030 |
| 4096 бит | ~140 бит безопасности | Долгосрочное хранение |

**Медленно!** RSA в 100–1000 раз медленнее симметричного AES. Поэтому RSA используется только для:
- Обмена ключами (шифруем небольшой симметричный ключ).
- Цифровых подписей.
- НЕ для шифрования больших данных.

---

## RSA в Go

```go
import (
    "crypto/rand"
    "crypto/rsa"
    "crypto/sha256"
    "crypto/x509"
    "encoding/pem"
)

// Генерация ключей
privateKey, err := rsa.GenerateKey(rand.Reader, 4096)
publicKey := &privateKey.PublicKey

// Сериализация в PEM
privDER := x509.MarshalPKCS8PrivateKey(privateKey)
privPEM := pem.EncodeToMemory(&pem.Block{
    Type:  "PRIVATE KEY",
    Bytes: privDER,
})

pubDER, _ := x509.MarshalPKIXPublicKey(publicKey)
pubPEM := pem.EncodeToMemory(&pem.Block{
    Type:  "PUBLIC KEY",
    Bytes: pubDER,
})

// Шифрование (OAEP — современный, безопасный padding)
ciphertext, err := rsa.EncryptOAEP(
    sha256.New(),
    rand.Reader,
    publicKey,
    []byte("secret data"),
    nil, // label
)

// Расшифровка
plaintext, err := rsa.DecryptOAEP(
    sha256.New(),
    rand.Reader,
    privateKey,
    ciphertext,
    nil,
)

// Подпись
hash := sha256.Sum256([]byte("document"))
signature, err := rsa.SignPSS(
    rand.Reader,
    privateKey,
    crypto.SHA256,
    hash[:],
    nil,
)

// Проверка подписи
err = rsa.VerifyPSS(
    publicKey,
    crypto.SHA256,
    hash[:],
    signature,
    nil,
)
```

### RSA Padding

| Padding | Описание | Статус |
|---------|---------|--------|
| PKCS#1 v1.5 | Устаревший | Уязвим к Bleichenbacher attack — **избегать** |
| OAEP | Optimal Asymmetric Encryption Padding | Использовать для шифрования |
| PSS | Probabilistic Signature Scheme | Использовать для подписей |

---

## Эллиптические кривые (ECC) — современная альтернатива RSA

**Проблема RSA**: ключи большого размера (4096 бит), медленная операция.

**Решение ECC**: математика на эллиптических кривых.

```
Уравнение эллиптической кривой: y² = x³ + ax + b (mod p)
```

Безопасность основана на **проблеме дискретного логарифма на эллиптической кривой (ECDLP)** — значительно сложнее, чем факторизация RSA.

### Сравнение RSA vs ECC

| Безопасность | RSA | ECC |
|-------------|-----|-----|
| 80 бит | 1024 бит | 160 бит |
| 112 бит | 2048 бит | 224 бит |
| 128 бит | 3072 бит | **256 бит** |
| 192 бит | 7680 бит | 384 бит |
| 256 бит | 15360 бит | 521 бит |

ECC при той же безопасности: **ключи в 10+ раз меньше, операции быстрее**.

---

## ECDSA — подписи на эллиптических кривых

```go
import "crypto/ecdsa"
import "crypto/elliptic"

// Генерация ключа
privateKey, err := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
publicKey := &privateKey.PublicKey

// Подпись
hash := sha256.Sum256([]byte("data"))
r, s, err := ecdsa.Sign(rand.Reader, privateKey, hash[:])

// Верификация
valid := ecdsa.Verify(publicKey, hash[:], r, s)

// Или через crypto.Signer интерфейс
signature, err := privateKey.Sign(rand.Reader, hash[:], crypto.SHA256)
```

**Популярные кривые:**
- **P-256** (secp256r1, prime256v1) — NIST, TLS, JWT ES256.
- **P-384** — для более высокой безопасности.
- **secp256k1** — Bitcoin, Ethereum.

---

## Ed25519 — самый современный стандарт подписей

**Ed25519** (Edwards-curve Digital Signature Algorithm) — подписи на кривой Curve25519.

**Преимущества перед ECDSA:**
- **Детерминированный**: не требует случайности при подписи (нет утечки ключа при плохом RNG).
- **Быстрее** ECDSA в ~2 раза.
- **Безопаснее** по дизайну (нет уязвимостей Sony PS3-типа).
- Маленькие ключи: **64 байта** (32 private + 32 public).

```go
import "crypto/ed25519"

// Генерация
publicKey, privateKey, err := ed25519.GenerateKey(rand.Reader)

// Подпись
message := []byte("important document")
signature := ed25519.Sign(privateKey, message) // 64 байта

// Верификация
valid := ed25519.Verify(publicKey, message, signature)
```

**Применение Ed25519:**
- SSH (тип `ssh-ed25519`) — самый рекомендуемый сегодня.
- JWT (алгоритм EdDSA).
- TLS сертификаты (будущее).
- Signal Protocol, WireGuard, OpenSSH.

---

## X25519 — ECDH для обмена ключами

**X25519** — алгоритм Диффи-Хеллмана на кривой Curve25519. Используется для **обмена ключами** (не для подписей).

```
Alice: генерирует a, отправляет A = g^a
Bob: генерирует b, отправляет B = g^b

Shared secret: Alice: B^a = g^(ab)
               Bob:   A^b = g^(ab)   ← одинаково!
```

```go
import "golang.org/x/crypto/curve25519"

// Генерация ключей
privateKey := make([]byte, 32)
rand.Read(privateKey)
privateKey[0] &= 248
privateKey[31] &= 127
privateKey[31] |= 64

publicKey, _ := curve25519.X25519(privateKey, curve25519.Basepoint)

// Вычисление shared secret
sharedSecret, _ := curve25519.X25519(myPrivateKey, theirPublicKey)
```

**Применение**: TLS 1.3 (основной key exchange), WireGuard, Signal.

---

## Алгоритм Диффи-Хеллмана (DH / ECDH)

Позволяет двум сторонам получить **общий секрет** по открытому каналу, не передавая его напрямую.

```
Public: g (base), p (prime)

Alice               Channel             Bob
a (secret) ──────────────────────────── b (secret)
A = g^a mod p ──►     A, B ◄──── B = g^b mod p

S = B^a mod p = g^(ab) mod p
                        S = A^b mod p = g^(ab) mod p
                        ↑ одинаковый shared secret!
```

Перехватчик видит только A и B, но не может вычислить a или b (задача дискретного логарифма).

---

## Post-Quantum Cryptography (PQC) — новые стандарты

**Проблема**: квантовый компьютер с алгоритмом Шора может взломать RSA, ECDSA, DH за полиномиальное время.

NIST в 2024 году стандартизировал первые постквантовые алгоритмы:

| Алгоритм | Назначение | Основа | Размер ключа |
|---------|-----------|--------|------------|
| **ML-KEM** (FIPS 203) | Обмен ключами | Module Lattice | ~1 КБ |
| **ML-DSA** (FIPS 204) | Цифровые подписи | Module Lattice | ~2.5 КБ |
| **SLH-DSA** (FIPS 205) | Цифровые подписи | Hash-based | ~50 байт ключ, ~50 КБ подпись |

**В Go:**
```go
// ML-KEM (Go 1.24+) — бывший Kyber
import "crypto/mlkem"

// Генерация ключей
dk, _ := mlkem.GenerateKey768() // 768 — уровень безопасности
encapsulationKey := dk.EncapsulationKey()

// Encapsulation (клиент)
ciphertext, sharedSecret, _ := mlkem.Encapsulate768(encapsulationKey)

// Decapsulation (сервер)
sharedSecret2, _ := dk.Decapsulate(ciphertext)
// sharedSecret == sharedSecret2
```

**Hybrid approach** (рекомендуется сегодня): X25519 + ML-KEM вместе:
- Классический алгоритм: взламывается квантовым компьютером.
- Постквантовый: на случай будущей угрозы.
- Оба сразу: атакующий должен взломать оба.

TLS 1.3 в Go поддерживает `X25519Kyber768Draft00` начиная с Go 1.23.

---

## Форматы ключей

### PEM (Privacy Enhanced Mail)

```
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...
-----END PUBLIC KEY-----
```

```
-----BEGIN PRIVATE KEY-----     ← PKCS#8 (универсальный)
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBK...
-----END PRIVATE KEY-----

-----BEGIN RSA PRIVATE KEY-----  ← PKCS#1 (только RSA, устаревший)
MIIEowIBAAKCAQEA...
-----END RSA PRIVATE KEY-----

-----BEGIN EC PRIVATE KEY-----   ← EC specific
...
-----END EC PRIVATE KEY-----
```

### Чтение ключей в Go

```go
// Читаем PEM
pemData, _ := os.ReadFile("private.key")
block, _ := pem.Decode(pemData)

// Парсим PKCS#8 (универсальный формат)
key, err := x509.ParsePKCS8PrivateKey(block.Bytes)
switch k := key.(type) {
case *rsa.PrivateKey:
    // RSA
case *ecdsa.PrivateKey:
    // ECDSA
case ed25519.PrivateKey:
    // Ed25519
}
```

---

## SSH ключи

SSH использует асимметричную криптографию для аутентификации.

```bash
# Генерация ключей (рекомендуется Ed25519)
ssh-keygen -t ed25519 -C "email@example.com"
# → ~/.ssh/id_ed25519      (приватный)
# → ~/.ssh/id_ed25519.pub  (публичный — копируем на сервер)

# RSA (если нужна совместимость)
ssh-keygen -t rsa -b 4096 -C "email@example.com"

# Копирование публичного ключа на сервер
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server

# ~/.ssh/authorized_keys на сервере:
# ssh-ed25519 AAAA... email@example.com
```

**Как работает SSH аутентификация:**
1. Сервер отправляет challenge (случайные данные).
2. Клиент подписывает challenge своим приватным ключом.
3. Сервер проверяет подпись публичным ключом из `authorized_keys`.

---

## Итог: что использовать в 2026

| Задача | Рекомендация |
|-------|-------------|
| Шифрование данных | AES-256-GCM (симметричное) |
| Обмен ключами | X25519 (+ ML-KEM для PQC) |
| Цифровые подписи | Ed25519 |
| TLS сертификаты | ECDSA P-256 (или RSA 2048 если нужна совместимость) |
| SSH | Ed25519 |
| Хранение паролей | Argon2id |
| Избегать | RSA < 2048, ECDSA без deterministic nonce, MD5/SHA-1 |

## Что спрашивают на собеседованиях

1. **Чем отличается симметричное от асимметричного шифрования?** → 1 ключ vs пара ключей; симметричное быстрее.
2. **Как работает RSA?** → Факторизация: n=p*q; шифруем public key, расшифруем private.
3. **Почему RSA медленный?** → Операции на больших числах (1024–4096 бит); используем для ключевого обмена, не для данных.
4. **Что такое Perfect Forward Secrecy?** → Ephemeral ключи (ECDHE): даже при компрометации долгосрочного ключа прошлые сессии безопасны.
5. **RSA vs ECDSA vs Ed25519?** → Ed25519 — детерминированный, быстрый, маленькие ключи, рекомендуется.
6. **Что такое постквантовая криптография?** → Алгоритмы устойчивые к квантовым компьютерам (ML-KEM, ML-DSA).
7. **Как работает алгоритм Диффи-Хеллмана?** → Обмен ключами по открытому каналу через задачу дискретного логарифма.
