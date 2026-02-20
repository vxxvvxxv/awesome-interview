# TLS (Transport Layer Security)

Полезные ссылки:
- [RFC 8446 — TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [RFC 5246 — TLS 1.2](https://datatracker.ietf.org/doc/html/rfc5246)
- [The Illustrated TLS 1.3 Connection](https://tls13.xargs.org/)

---

## Что такое TLS?

**TLS** (Transport Layer Security) — криптографический протокол для защищённой передачи данных поверх TCP. Предшественник — SSL (устарел).

**Что обеспечивает:**
- **Конфиденциальность** — данные зашифрованы, посредник не прочитает.
- **Аутентификация** — клиент убедился что общается с нужным сервером (через сертификат).
- **Целостность** — данные не изменены в пути (HMAC/AEAD).

## Версии TLS

| Версия | Статус | Особенности |
|--------|--------|------------|
| SSL 2.0/3.0 | Устарел | Уязвимости (POODLE, BEAST) |
| TLS 1.0/1.1 | Устарел | Запрещён RFC 8996 (2021) |
| TLS 1.2 | Ещё используется | 2 roundtrip handshake |
| **TLS 1.3** | Рекомендуется | 1 roundtrip, удалены слабые алгоритмы |

## TLS Handshake (TLS 1.3)

```
Client                              Server
  │                                    │
  │──── ClientHello ──────────────────►│
  │     (версии TLS, cipher suites,    │
  │      key_share, случайное число)    │
  │                                    │
  │◄─── ServerHello ───────────────────│
  │     (выбранный cipher,             │
  │      key_share, случайное число)   │
  │◄─── {Certificate} ─────────────────│  } зашифровано
  │◄─── {CertificateVerify} ───────────│  } начиная
  │◄─── {Finished} ────────────────────│  } отсюда
  │                                    │
  │──── {Finished} ────────────────────►│
  │                                    │
  │══════════════ Data ════════════════│  Application data
```

**Ключевые моменты TLS 1.3:**
- **1 roundtrip** (RTT) для нового соединения.
- **0-RTT** (Early Data) для повторного (данные отправляются вместе с Finished, но без Perfect Forward Secrecy — с осторожностью).
- Убраны: RSA key exchange, MD5, SHA-1, DH, RC4, 3DES, экспортные шифры.
- Обязателен **ECDHE** (Ephemeral Elliptic Curve Diffie-Hellman).

## Ключевой обмен (Key Exchange)

### ECDHE (TLS 1.3)

```
Client: генерирует приватный ключ c, публикует g^c
Server: генерирует приватный ключ s, публикует g^s

Shared secret = (g^c)^s = (g^s)^c = g^(cs)
```

**Ephemeral** = каждое соединение — новые ключи → **Perfect Forward Secrecy** (PFS):
- Даже если долгосрочный приватный ключ сервера скомпрометирован, прошлые сессии расшифровать нельзя.

### Симметричное шифрование (после handshake)

Данные шифруются симметрично (быстро):
- **AES-256-GCM** — основной алгоритм.
- **ChaCha20-Poly1305** — для устройств без аппаратного AES.

## Сертификаты

### Структура X.509 сертификата

```
Certificate:
  Version: 3
  Serial Number: 01:23:45:67:...
  Signature Algorithm: SHA256withRSA
  Issuer: CN=Let's Encrypt Authority X3, O=Let's Encrypt, C=US
  Validity:
    Not Before: Jan 1 00:00:00 2025 GMT
    Not After:  Apr 1 00:00:00 2025 GMT
  Subject: CN=api.example.com
  Subject Alternative Names:
    DNS: api.example.com
    DNS: www.api.example.com
  Public Key: RSA 2048 bits (или EC P-256)
  Extensions:
    Key Usage: Digital Signature, Key Encipherment
    Extended Key Usage: TLS Web Server Authentication
  Signature: (подпись CA)
```

### Цепочка доверия (Chain of Trust)

```
Root CA (самоподписан)
└── Intermediate CA (подписан Root CA)
    └── Leaf Certificate (ваш домен, подписан Intermediate CA)
```

- Браузер доверяет Root CA (в системном хранилище).
- Root CA подписывает Intermediate CA → мы доверяем им.
- Intermediate CA подписывает ваш сертификат → мы доверяем ему.

### Виды сертификатов

| Тип | Проверяет | Описание |
|-----|----------|---------|
| **DV** (Domain Validation) | Владение доменом | Бесплатный (Let's Encrypt). Быстро. |
| **OV** (Organization Validation) | Домен + организация | Ручная проверка CA. |
| **EV** (Extended Validation) | Домен + организация (строго) | Самый строгий. |
| **Wildcard** | `*.example.com` | Все поддомены (один уровень) |
| **SAN** | Несколько доменов | `api.example.com`, `www.example.com`, ... |

## Mutual TLS (mTLS)

Обычный TLS: сервер доказывает себя клиенту.
**mTLS**: и сервер, и **клиент** доказывают себя друг другу.

Используется в микросервисах (service mesh: Istio, Linkerd), внутренних API.

```go
// Сервер с mTLS
cert, _ := tls.LoadX509KeyPair("server.crt", "server.key")
caCert, _ := os.ReadFile("ca.crt")

caCertPool := x509.NewCertPool()
caCertPool.AppendCertsFromPEM(caCert)

tlsConfig := &tls.Config{
    Certificates: []tls.Certificate{cert},
    ClientAuth:   tls.RequireAndVerifyClientCert, // требуем сертификат клиента
    ClientCAs:    caCertPool,
}

server := &http.Server{
    Addr:      ":443",
    TLSConfig: tlsConfig,
}
server.ListenAndServeTLS("", "")

// Клиент с сертификатом
clientCert, _ := tls.LoadX509KeyPair("client.crt", "client.key")
tlsConfig := &tls.Config{
    Certificates: []tls.Certificate{clientCert},
}
```

## TLS в Go

```go
// Базовый HTTPS сервер
http.ListenAndServeTLS(":443", "server.crt", "server.key", mux)

// Настроенный TLS
tlsConfig := &tls.Config{
    MinVersion:               tls.VersionTLS13,
    CurvePreferences:         []tls.CurveID{tls.X25519, tls.CurveP256},
    PreferServerCipherSuites: true,
}

server := &http.Server{
    Addr:      ":443",
    TLSConfig: tlsConfig,
}

// Клиент с проверкой конкретного CA
caCert, _ := os.ReadFile("ca.crt")
caCertPool := x509.NewCertPool()
caCertPool.AppendCertsFromPEM(caCert)

client := &http.Client{
    Transport: &http.Transport{
        TLSClientConfig: &tls.Config{
            RootCAs: caCertPool,
        },
    },
}

// Отключить проверку сертификата (ТОЛЬКО для тестов!)
client := &http.Client{
    Transport: &http.Transport{
        TLSClientConfig: &tls.Config{InsecureSkipVerify: true},
    },
}
```

## Атаки на TLS

| Атака | Описание | Защита |
|-------|---------|--------|
| MITM | Подмена сертификата | Certificate Pinning, HSTS |
| BEAST | Использует уязвимость TLS 1.0 CBC | TLS 1.2+ |
| POODLE | Атака на SSL 3.0 | Отключить SSL 3.0/TLS 1.0 |
| Heartbleed | Уязвимость OpenSSL | Обновление OpenSSL |
| CRIME/BREACH | Атака на сжатие | Отключить TLS compression |
| Downgrade | Принудить использовать слабый TLS | TLS_FALLBACK_SCSV, TLS 1.3 |

### HSTS (HTTP Strict Transport Security)

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

Браузер запомнит: этот домен — только HTTPS, даже без редиректа.

## Что спрашивают на собеседованиях

1. **Что такое TLS и зачем?** → Шифрование + аутентификация + целостность поверх TCP.
2. **TLS Handshake — как работает?** → ClientHello → ServerHello → сертификат → ключевой обмен → симметричное шифрование.
3. **Perfect Forward Secrecy?** → Ephemeral ключи (ECDHE) — компрометация долгосрочного ключа не раскроет прошлые сессии.
4. **Разница TLS 1.2 и TLS 1.3?** → 1.3: 1RTT, убраны слабые алгоритмы, только ECDHE, AEAD шифрование.
5. **Что такое mTLS?** → Взаимная аутентификация — клиент тоже предъявляет сертификат.
6. **Цепочка доверия?** → Root CA → Intermediate CA → Leaf cert. Браузер доверяет Root CA.
7. **Разница DV/OV/EV?** → Степень проверки владельца домена/организации.
