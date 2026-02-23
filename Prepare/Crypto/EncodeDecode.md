# Кодирование и ключи: PEM, DER, форматы криптографических данных

Полезные ссылки:
- [crypto/x509 — Go stdlib](https://pkg.go.dev/crypto/x509)
- [encoding/pem — Go stdlib](https://pkg.go.dev/encoding/pem)
- [RFC 5958 — PKCS#8](https://datatracker.ietf.org/doc/html/rfc5958)
- [RFC 7468 — PEM Format](https://datatracker.ietf.org/doc/html/rfc7468)

---

## Открытые ключи (Public Keys)

**Публичный ключ** — математически связан с приватным, но не позволяет его восстановить. Можно свободно публиковать.

**Применение:**
- Шифрование данных (только владелец приватного ключа расшифрует).
- Проверка подписи (созданной приватным ключом).
- Аутентификация в SSH, TLS, JWT.

### Форматы публичных ключей

```
-----BEGIN PUBLIC KEY-----          ← PKCS#8 SubjectPublicKeyInfo (универсальный)
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcD
QgAEbEwHmBg3NIpF0M/IbozTw2M/TQDB
ZH8j2MXB9w==
-----END PUBLIC KEY-----

-----BEGIN RSA PUBLIC KEY-----      ← PKCS#1 (только RSA, устаревший формат)
MIIBCgKCAQEA2a2r...
-----END RSA PUBLIC KEY-----
```

### Структура публичного ключа (ASN.1/DER)

```
SubjectPublicKeyInfo:
  algorithm: AlgorithmIdentifier
    algorithm: OID (1.2.840.10045.2.1 для EC, 1.2.840.113549.1.1.1 для RSA)
    parameters: OID кривой (для EC)
  subjectPublicKey: BIT STRING (сам ключ)
```

```go
import (
    "crypto/ecdsa"
    "crypto/elliptic"
    "crypto/rand"
    "crypto/x509"
    "encoding/pem"
)

// Генерация EC ключевой пары
privateKey, _ := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
publicKey := &privateKey.PublicKey

// Сериализация публичного ключа → DER (бинарный)
pubDER, _ := x509.MarshalPKIXPublicKey(publicKey) // PKCS#8 формат

// DER → PEM (base64 с заголовком)
pubPEM := pem.EncodeToMemory(&pem.Block{
    Type:  "PUBLIC KEY",
    Bytes: pubDER,
})
fmt.Println(string(pubPEM))

// PEM → DER → публичный ключ
block, _ := pem.Decode(pubPEM)
pub, _ := x509.ParsePKIXPublicKey(block.Bytes) // возвращает interface{}
ecPub := pub.(*ecdsa.PublicKey)
```

---

## Закрытые ключи (Private Keys)

**Приватный ключ** — секретный. Должен храниться зашифрованным. Из него математически выводится публичный ключ.

### Форматы приватных ключей

```
-----BEGIN PRIVATE KEY-----         ← PKCS#8 (универсальный, рекомендуется)
MIGHAgEAMBMGByqGSM49AgEGCCqGSM49
AwEHBG0wawIBAQQgr+OGTxY...
-----END PRIVATE KEY-----

-----BEGIN ENCRYPTED PRIVATE KEY-----   ← PKCS#8 зашифрованный (passphrase)
MIIFHDBOBgkqhkiG9w0BBQ0w...
-----END ENCRYPTED PRIVATE KEY-----

-----BEGIN EC PRIVATE KEY-----      ← SEC1 (только EC, устаревший)
MHQCAQEEIBt06cg...
-----END EC PRIVATE KEY-----

-----BEGIN RSA PRIVATE KEY-----     ← PKCS#1 (только RSA, устаревший)
MIIEowIBAAKCAQEA2a2r...
-----END RSA PRIVATE KEY-----
```

### Хранение приватных ключей

```go
// Сохранить приватный ключ (PKCS#8)
privDER, _ := x509.MarshalPKCS8PrivateKey(privateKey)
privPEM := pem.EncodeToMemory(&pem.Block{
    Type:  "PRIVATE KEY",
    Bytes: privDER,
})
os.WriteFile("private.pem", privPEM, 0600) // chmod 600 — только владелец!

// Загрузить приватный ключ
pemData, _ := os.ReadFile("private.pem")
block, _ := pem.Decode(pemData)
key, _ := x509.ParsePKCS8PrivateKey(block.Bytes) // interface{}
privKey := key.(*ecdsa.PrivateKey)

// Зашифровать ключ паролем (PEM с шифрованием)
encBlock, _ := x509.EncryptPEMBlock(
    rand.Reader, "ENCRYPTED PRIVATE KEY",
    privDER, []byte("passphrase"), x509.PEMCipherAES256,
)
encPEM := pem.EncodeToMemory(encBlock)
```

---

## Подпись (Digital Signature)

**Цифровая подпись** — математическое доказательство того, что данные подписаны владельцем приватного ключа.

### Как работает

```
Подписание:
  hash = SHA256(message)
  signature = Sign(privateKey, hash)

Проверка:
  hash = SHA256(message)
  valid = Verify(publicKey, hash, signature)
```

### ECDSA (Elliptic Curve Digital Signature Algorithm)

```go
import (
    "crypto/ecdsa"
    "crypto/sha256"
    "math/big"
)

message := []byte("важное сообщение")
hash := sha256.Sum256(message)

// Подписание
r, s, err := ecdsa.Sign(rand.Reader, privateKey, hash[:])
if err != nil {
    log.Fatal(err)
}

// Подпись = (r, s) — два больших числа
// Сериализация в DER формат (ASN.1)
sigDER, _ := asn1.Marshal(struct{ R, S *big.Int }{r, s})

// Проверка
valid := ecdsa.Verify(publicKey, hash[:], r, s)
fmt.Println("Valid:", valid) // true
```

### Ed25519 (рекомендуется)

```go
import "crypto/ed25519"

// Генерация
pubKey, privKey, _ := ed25519.GenerateKey(rand.Reader)

// Подпись — детерминирована (нет нужды в rand)
signature := ed25519.Sign(privKey, message)

// Проверка
valid := ed25519.Verify(pubKey, message, signature)
```

**Преимущества Ed25519:**
- Детерминированная подпись (не требует хорошего случайного числа).
- Быстрее ECDSA.
- Короткие подписи (64 байта).
- Устойчив к timing-атакам.

### RSA подпись (PSS)

```go
import (
    "crypto/rsa"
    "crypto/x509"
)

// PKCS1v15 (устаревший)
sig, _ := rsa.SignPKCS1v15(rand.Reader, rsaKey, crypto.SHA256, hash[:])
err = rsa.VerifyPKCS1v15(&rsaKey.PublicKey, crypto.SHA256, hash[:], sig)

// PSS (рекомендуется, рандомизирован)
sig, _ = rsa.SignPSS(rand.Reader, rsaKey, crypto.SHA256, hash[:], nil)
err = rsa.VerifyPSS(&rsaKey.PublicKey, crypto.SHA256, hash[:], sig, nil)
```

---

## Форматы кодирования

### DER (Distinguished Encoding Rules)

**Бинарный** формат. Базовый формат для хранения ASN.1 структур.

### PEM (Privacy-Enhanced Mail)

**Base64 + заголовки**. Текстовый wrapper над DER:

```
-----BEGIN <TYPE>-----
<base64(DER)>
-----END <TYPE>-----
```

```go
// Произвольные данные → PEM
block := &pem.Block{
    Type:  "CERTIFICATE",
    Bytes: certDER,
}
pemData := pem.EncodeToMemory(block)

// PEM → DER
block, rest := pem.Decode(pemData)
// rest — остаток (несколько блоков в одном файле)
```

### Base64 и Base64URL

```go
import "encoding/base64"

// Стандартный Base64 (использует + и /)
encoded := base64.StdEncoding.EncodeToString(data)

// URL-safe Base64 (использует - и _, без =)
// Используется в JWT, URL токенах
encoded = base64.RawURLEncoding.EncodeToString(data)

// Декодирование
decoded, err := base64.RawURLEncoding.DecodeString(encoded)
```

### Hex (шестнадцатеричный)

```go
import "encoding/hex"

hexStr := hex.EncodeToString(hashBytes) // "2cf24dba..."
decoded, _ := hex.DecodeString(hexStr)
```

---

## PKCS#11 и HSM

**HSM** (Hardware Security Module) — физическое устройство для хранения ключей. Ключ никогда не покидает устройство.

**PKCS#11** — стандартный интерфейс (Cryptoki) для работы с HSM, смарт-картами, токенами.

```go
// Пример с miekg/pkcs11
import "github.com/miekg/pkcs11"

p := pkcs11.New("/usr/lib/softhsm/libsofthsm2.so")
p.Initialize()
session, _ := p.OpenSession(0, pkcs11.CKF_SERIAL_SESSION)
p.Login(session, pkcs11.CKU_USER, "1234")

// Подпись внутри HSM (ключ не покидает устройство)
p.SignInit(session, []*pkcs11.Mechanism{pkcs11.NewMechanism(pkcs11.CKM_SHA256_RSA_PKCS, nil)}, keyHandle)
signature, _ := p.Sign(session, data)
```

---

## Что спрашивают на собеседованиях

1. **Чем PEM отличается от DER?** → DER — бинарный ASN.1, PEM — base64(DER) с текстовыми заголовками.
2. **Что такое PKCS#8?** → Стандарт хранения приватных ключей (универсальный, для любого алгоритма).
3. **Почему Ed25519 лучше ECDSA для подписи?** → Детерминирована (не нужен хороший rand), быстрее, не уязвима к timing-атакам.
4. **Что такое цифровая подпись?** → hash(message) + Sign(privKey) → signature. Верификация: Verify(pubKey, hash, signature).
5. **Почему нельзя хранить приватный ключ без passphrase?** → При компрометации файла атакующий сразу имеет ключ. С passphrase нужен ещё пароль.
6. **Что такое HSM?** → Аппаратный модуль безопасности; ключи никогда не покидают устройство, операции выполняются внутри.
