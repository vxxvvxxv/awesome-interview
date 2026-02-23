# JWKS — JSON Web Key Sets

Полезные ссылки:
- [RFC 7517 — JSON Web Key (JWK)](https://datatracker.ietf.org/doc/html/rfc7517)
- [RFC 7518 — JSON Web Algorithms (JWA)](https://datatracker.ietf.org/doc/html/rfc7518)
- [RFC 7519 — JSON Web Token (JWT)](https://datatracker.ietf.org/doc/html/rfc7519)
- [lestrrat-go/jwx](https://github.com/lestrrat-go/jwx)
- [golang-jwt/jwt](https://github.com/golang-jwt/jwt)
- [JWKS endpoint demo (Auth0)](https://auth0.com/docs/security/tokens/json-web-tokens/json-web-key-sets)

---

## Что такое JWKS?

**JWKS (JSON Web Key Set)** — это JSON-структура, описывающая набор **публичных криптографических ключей** (RFC 7517). Используется для верификации **JWT (JSON Web Token)** без передачи секрета.

### Связь JWT → JWKS

```
┌─────────────────────────────────────────────────┐
│                 Authorization Server            │
│  (Auth0, Keycloak, Google, ваш сервис)          │
│                                                 │
│   Приватный ключ ──► подписывает JWT            │
│   Публичный ключ ──► публикует в JWKS endpoint  │
└──────────────────────────┬──────────────────────┘
                           │ GET /.well-known/jwks.json
                           ▼
┌─────────────────────────────────────────────────┐
│                Resource Server (API)            │
│                                                 │
│   Получает JWT от клиента                       │
│   Скачивает публичный ключ из JWKS              │
│   Верифицирует подпись JWT                      │
└─────────────────────────────────────────────────┘
```

---

## Структура JWK и JWKS

### JWKS (набор ключей)

```json
{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "alg": "RS256",
      "kid": "2024-01-key",
      "n":   "0vx7agoebGcQSuuPiLJXZptN...",
      "e":   "AQAB"
    },
    {
      "kty": "EC",
      "use": "sig",
      "alg": "ES256",
      "kid": "2024-02-key",
      "crv": "P-256",
      "x":   "f83OJ3D2xF1Bg8vub9tLe1gH...",
      "y":   "x_FEzRu9m36HLN_tue659LNp..."
    }
  ]
}
```

### Поля JWK

| Поле  | Описание                                          | Пример                     |
|-------|---------------------------------------------------|----------------------------|
| `kty` | Key Type — тип ключа                              | `"RSA"`, `"EC"`, `"OKP"`  |
| `use` | Public Key Use — назначение                       | `"sig"` (подпись) или `"enc"` (шифрование) |
| `alg` | Algorithm — алгоритм                             | `"RS256"`, `"ES256"`, `"EdDSA"` |
| `kid` | Key ID — идентификатор ключа                     | `"2024-01"`, UUID          |
| `n`   | RSA modulus (base64url)                          | длинная строка             |
| `e`   | RSA exponent (base64url)                         | `"AQAB"` (65537)           |
| `crv` | EC curve                                          | `"P-256"`, `"P-384"`, `"Ed25519"` |
| `x`   | EC x-coordinate (base64url)                      |                             |
| `y`   | EC y-coordinate (base64url)                      |                             |

---

## Как работает верификация JWT через JWKS

### Поток верификации

```
Клиент                    API (Resource Server)          Auth Server
  │                              │                            │
  │── Authorization: Bearer JWT ─►│                            │
  │                              │                            │
  │                    JWT header decode                      │
  │                    { "alg": "RS256", "kid": "key-123" }   │
  │                              │                            │
  │                              │─── GET /.well-known/jwks.json ─►│
  │                              │◄── { "keys": [...] } ──────│
  │                              │                            │
  │                    Найти ключ по kid="key-123"            │
  │                    Верифицировать подпись                 │
  │                    Проверить claims (exp, iss, aud)       │
  │                              │                            │
  │◄── 200 OK / 401 Unauthorized ─│                            │
```

### JWT Header с kid

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "2024-01-key"
}
```

Поле `kid` указывает, **каким ключом** из JWKS нужно верифицировать этот токен.

---

## Преимущества JWKS

| Преимущество | Описание |
|---|---|
| **Нет общего секрета** | Публичный ключ можно раздавать публично, приватный — только у auth-сервера |
| **Ротация ключей** | JWKS содержит несколько ключей → плавная замена без даунтайма |
| **Алгоритмическая гибкость** | Разные ключи для разных алгоритмов (RS256, ES256, EdDSA) |
| **Стандарт** | Поддерживается всеми IdP: Auth0, Keycloak, Google, Okta, AWS Cognito |
| **Автоматическое обновление** | Клиенты сами перезагружают JWKS при появлении нового `kid` |
| **Zero-secret distribution** | Не нужно передавать секрет при деплое микросервисов |

### JWKS vs симметричный секрет

```
Симметричный (HS256):
  Auth Server ──[shared_secret]──► API 1
                                ──► API 2  (у каждого сервиса одинаковый секрет → утечка = катастрофа)
                                ──► API 3

JWKS (RS256/ES256):
  Auth Server ──[приватный ключ] (только у auth-server)
              ──publishes──► /.well-known/jwks.json
  API 1 ──fetch──► публичный ключ (безопасно раздавать)
  API 2 ──fetch──► публичный ключ
  API 3 ──fetch──► публичный ключ
```

---

## Ротация ключей

JWKS решает проблему **key rotation** без простоя:

```
1. Создать новый ключ (kid="2024-02")
2. Добавить его в JWKS рядом со старым:
   { "keys": [ {kid: "2024-01"}, {kid: "2024-02"} ] }
3. Начать выдавать новые JWT с kid="2024-02"
4. Старые JWT с kid="2024-01" ещё верифицируются (до истечения exp)
5. После истечения TTL старых токенов — удалить kid="2024-01"
```

---

## Реализация на Go

### 1. Сервер JWKS (Auth Server)

```go
package main

import (
    "crypto/ecdsa"
    "crypto/elliptic"
    "crypto/rand"
    "encoding/base64"
    "encoding/json"
    "math/big"
    "net/http"
)

type JWK struct {
    Kty string `json:"kty"`
    Use string `json:"use"`
    Alg string `json:"alg"`
    Kid string `json:"kid"`
    Crv string `json:"crv"`
    X   string `json:"x"`
    Y   string `json:"y"`
}

type JWKS struct {
    Keys []JWK `json:"keys"`
}

var privateKey *ecdsa.PrivateKey

func init() {
    var err error
    privateKey, err = ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
    if err != nil {
        panic(err)
    }
}

// base64url без padding
func b64url(b []byte) string {
    return base64.RawURLEncoding.EncodeToString(b)
}

// bigIntToBytes конвертирует big.Int в байты фиксированной длины
func bigIntToBytes(n *big.Int, size int) []byte {
    b := n.Bytes()
    if len(b) < size {
        padded := make([]byte, size)
        copy(padded[size-len(b):], b)
        return padded
    }
    return b
}

func jwksHandler(w http.ResponseWriter, r *http.Request) {
    pub := privateKey.PublicKey
    jwk := JWK{
        Kty: "EC",
        Use: "sig",
        Alg: "ES256",
        Kid: "2024-01",
        Crv: "P-256",
        X:   b64url(bigIntToBytes(pub.X, 32)),
        Y:   b64url(bigIntToBytes(pub.Y, 32)),
    }

    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("Cache-Control", "public, max-age=3600")
    json.NewEncoder(w).Encode(JWKS{Keys: []JWK{jwk}})
}

func main() {
    http.HandleFunc("/.well-known/jwks.json", jwksHandler)
    http.ListenAndServe(":8080", nil)
}
```

### 2. Выдача JWT (Auth Server) — через lestrrat-go/jwx

```bash
go get github.com/lestrrat-go/jwx/v2
```

```go
package auth

import (
    "context"
    "crypto/ecdsa"
    "time"

    "github.com/lestrrat-go/jwx/v2/jwa"
    "github.com/lestrrat-go/jwx/v2/jwk"
    "github.com/lestrrat-go/jwx/v2/jwt"
)

func IssueToken(privateKey *ecdsa.PrivateKey, userID string) (string, error) {
    // Создать JWK из приватного ключа
    key, err := jwk.FromRaw(privateKey)
    if err != nil {
        return "", err
    }
    key.Set(jwk.KeyIDKey, "2024-01")
    key.Set(jwk.AlgorithmKey, jwa.ES256)

    // Построить JWT
    tok, err := jwt.NewBuilder().
        Issuer("https://auth.example.com").
        Subject(userID).
        Audience([]string{"https://api.example.com"}).
        IssuedAt(time.Now()).
        Expiration(time.Now().Add(1 * time.Hour)).
        Claim("roles", []string{"admin"}).
        Build()
    if err != nil {
        return "", err
    }

    // Подписать
    signed, err := jwt.Sign(tok, jwt.WithKey(jwa.ES256, key))
    return string(signed), err
}
```

### 3. Верификация JWT (Resource Server) — с кешированием JWKS

```go
package middleware

import (
    "context"
    "net/http"
    "strings"
    "time"

    "github.com/lestrrat-go/jwx/v2/jwk"
    "github.com/lestrrat-go/jwx/v2/jwt"
)

type JWKSVerifier struct {
    cache *jwk.Cache
    url   string
}

func NewJWKSVerifier(jwksURL string) (*JWKSVerifier, error) {
    ctx := context.Background()

    // Автоматически обновляет JWKS каждые 15 минут
    cache := jwk.NewCache(ctx)
    err := cache.Register(jwksURL,
        jwk.WithMinRefreshInterval(15*time.Minute),
    )
    if err != nil {
        return nil, err
    }

    // Первоначальная загрузка
    _, err = cache.Refresh(ctx, jwksURL)
    if err != nil {
        return nil, err
    }

    return &JWKSVerifier{cache: cache, url: jwksURL}, nil
}

func (v *JWKSVerifier) VerifyToken(tokenStr string) (jwt.Token, error) {
    keySet, err := v.cache.Get(context.Background(), v.url)
    if err != nil {
        return nil, err
    }

    tok, err := jwt.Parse([]byte(tokenStr),
        jwt.WithKeySet(keySet),
        jwt.WithValidate(true),
        jwt.WithIssuer("https://auth.example.com"),
        jwt.WithAudience("https://api.example.com"),
    )
    return tok, err
}

// HTTP Middleware

type contextKey string
const claimsKey contextKey = "claims"

func (v *JWKSVerifier) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        authHeader := r.Header.Get("Authorization")
        if !strings.HasPrefix(authHeader, "Bearer ") {
            http.Error(w, "missing token", http.StatusUnauthorized)
            return
        }

        tokenStr := strings.TrimPrefix(authHeader, "Bearer ")
        tok, err := v.VerifyToken(tokenStr)
        if err != nil {
            http.Error(w, "invalid token: "+err.Error(), http.StatusUnauthorized)
            return
        }

        ctx := context.WithValue(r.Context(), claimsKey, tok)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Получить claims из контекста
func TokenFromContext(ctx context.Context) jwt.Token {
    tok, _ := ctx.Value(claimsKey).(jwt.Token)
    return tok
}
```

### 4. Использование Middleware

```go
package main

import (
    "encoding/json"
    "net/http"
)

func main() {
    verifier, err := NewJWKSVerifier("https://auth.example.com/.well-known/jwks.json")
    if err != nil {
        panic(err)
    }

    mux := http.NewServeMux()
    mux.HandleFunc("/profile", func(w http.ResponseWriter, r *http.Request) {
        tok := TokenFromContext(r.Context())

        resp := map[string]any{
            "sub":   tok.Subject(),
            "roles": tok.PrivateClaims()["roles"],
        }
        json.NewEncoder(w).Encode(resp)
    })

    // Применить middleware
    handler := verifier.Middleware(mux)
    http.ListenAndServe(":9090", handler)
}
```

### 5. Ручная верификация (без библиотек) — для понимания

```go
package main

import (
    "crypto/ecdsa"
    "crypto/sha256"
    "encoding/base64"
    "encoding/json"
    "fmt"
    "math/big"
    "strings"
)

// JWT состоит из трёх частей: header.payload.signature (base64url)
func verifyJWTManually(tokenStr string, pubKey *ecdsa.PublicKey) (map[string]any, error) {
    parts := strings.Split(tokenStr, ".")
    if len(parts) != 3 {
        return nil, fmt.Errorf("invalid JWT format")
    }

    // 1. Декодировать header
    headerJSON, _ := base64.RawURLEncoding.DecodeString(parts[0])
    var header map[string]string
    json.Unmarshal(headerJSON, &header)

    if header["alg"] != "ES256" {
        return nil, fmt.Errorf("unsupported algorithm: %s", header["alg"])
    }

    // 2. Верифицировать подпись
    // Подпись = ECDSA(SHA256(base64url(header) + "." + base64url(payload)))
    message := parts[0] + "." + parts[1]
    hash := sha256.Sum256([]byte(message))

    sigBytes, err := base64.RawURLEncoding.DecodeString(parts[2])
    if err != nil {
        return nil, err
    }

    // ECDSA подпись = r || s (каждый по 32 байта для P-256)
    if len(sigBytes) != 64 {
        return nil, fmt.Errorf("invalid signature length")
    }
    r := new(big.Int).SetBytes(sigBytes[:32])
    s := new(big.Int).SetBytes(sigBytes[32:])

    if !ecdsa.Verify(pubKey, hash[:], r, s) {
        return nil, fmt.Errorf("signature verification failed")
    }

    // 3. Декодировать payload
    payloadJSON, _ := base64.RawURLEncoding.DecodeString(parts[1])
    var claims map[string]any
    json.Unmarshal(payloadJSON, &claims)

    return claims, nil
}
```

---

## OpenID Connect Discovery

JWKS тесно связан с **OpenID Connect Discovery** (`.well-known/openid-configuration`):

```bash
# Получить конфигурацию
curl https://accounts.google.com/.well-known/openid-configuration

# Ответ содержит:
{
  "issuer": "https://accounts.google.com",
  "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",  # ← JWKS URL
  "authorization_endpoint": "...",
  "token_endpoint": "...",
  ...
}

# Получить публичные ключи
curl https://www.googleapis.com/oauth2/v3/certs
```

---

## Алгоритмы подписи JWT

| Алгоритм | Тип ключа | Безопасность | Рекомендация |
|----------|-----------|--------------|--------------|
| **HS256** | HMAC (симметричный) | Нужен общий секрет | Только для монолита |
| **RS256** | RSA (2048/4096 бит) | Хорошая | Широкая совместимость |
| **RS384/RS512** | RSA | Лучше | Если требует клиент |
| **ES256** | ECDSA P-256 | Отличная | **Рекомендуется** |
| **ES384** | ECDSA P-384 | Отличная | Для повышенных требований |
| **EdDSA** | Ed25519 | Отличная | **Современный стандарт** |
| **PS256** | RSA-PSS | Хорошая | Если требует стандарт |

**Правило:** В 2024+ предпочитай **ES256** или **EdDSA** вместо RS256.

---

## Кеширование JWKS

JWKS не нужно загружать при каждом запросе — это дорого. Стратегии кеширования:

```go
// Стратегия: Cache + refresh при неизвестном kid
type CachedJWKS struct {
    mu      sync.RWMutex
    keySet  jwk.Set
    url     string
    fetchAt time.Time
    ttl     time.Duration
}

func (c *CachedJWKS) GetKey(kid string) (jwk.Key, error) {
    c.mu.RLock()
    key, found := c.lookup(kid)
    c.mu.RUnlock()

    if found {
        return key, nil
    }

    // kid не найден → возможно ротация → принудительно обновить
    c.mu.Lock()
    defer c.mu.Unlock()

    if err := c.refresh(); err != nil {
        return nil, err
    }

    key, found = c.lookup(kid)
    if !found {
        return nil, fmt.Errorf("key %q not found in JWKS", kid)
    }
    return key, nil
}

func (c *CachedJWKS) lookup(kid string) (jwk.Key, bool) {
    if c.keySet == nil || time.Since(c.fetchAt) > c.ttl {
        return nil, false
    }
    return c.keySet.LookupKeyID(kid)
}
```

**Рекомендации:**
- TTL кеша: **5–15 минут** (`Cache-Control: max-age=...` от сервера)
- При неизвестном `kid`: **один** принудительный рефреш, потом ошибка
- Не рефрешить при каждом запросе с неизвестным `kid` — защита от DoS

---

## Безопасность JWKS

### Что проверять при валидации JWT

```go
jwt.Parse(tokenBytes,
    jwt.WithKeySet(keySet),
    jwt.WithValidate(true),           // проверить exp, nbf
    jwt.WithIssuer("expected-issuer"), // проверить iss
    jwt.WithAudience("my-service"),    // проверить aud (!)
    jwt.WithAcceptableSkew(30*time.Second), // drift часов
)
```

### Типичные уязвимости

| Уязвимость | Описание | Защита |
|---|---|---|
| **alg=none атака** | JWT без подписи | Запрет алгоритма `none` |
| **alg confusion** | RS256→HS256 подмена | Явно указывать ожидаемый алгоритм |
| **aud не проверяется** | Токен другого сервиса принят | Всегда проверять `aud` |
| **kid injection** | `kid: "../../etc/passwd"` | Только lookup по kid, никакого SQL/path |
| **JWKS poisoning** | Подмена JWKS URL | Hardcode URL или проверка issuer |
| **Expired token** | Использование истёкшего токена | Проверять `exp` (lestrrat делает это автоматически) |

```go
// Никогда так не делать (alg confusion):
jwt.Parse(token, jwt.WithKey(jwa.HS256, []byte(publicKeyPEM)))
// Злоумышленник может создать HS256 токен, используя публичный ключ как секрет
```

---

## Реализация JWKS-сервера с ротацией

```go
package jwks

import (
    "crypto/ecdsa"
    "crypto/elliptic"
    "crypto/rand"
    "encoding/json"
    "net/http"
    "sync"
    "time"
)

type KeyEntry struct {
    Kid       string
    CreatedAt time.Time
    Key       *ecdsa.PrivateKey
}

type JWKSServer struct {
    mu   sync.RWMutex
    keys []KeyEntry
}

func New() *JWKSServer {
    s := &JWKSServer{}
    s.RotateKey() // первый ключ
    return s
}

// RotateKey создаёт новый ключ (вызывать по расписанию)
func (s *JWKSServer) RotateKey() error {
    key, err := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
    if err != nil {
        return err
    }

    s.mu.Lock()
    defer s.mu.Unlock()

    s.keys = append(s.keys, KeyEntry{
        Kid:       time.Now().Format("2006-01-02"),
        CreatedAt: time.Now(),
        Key:       key,
    })

    // Удалить ключи старше 2 дней
    cutoff := time.Now().Add(-48 * time.Hour)
    var active []KeyEntry
    for _, k := range s.keys {
        if k.CreatedAt.After(cutoff) {
            active = append(active, k)
        }
    }
    s.keys = active
    return nil
}

// CurrentKey — последний ключ для подписи новых токенов
func (s *JWKSServer) CurrentKey() KeyEntry {
    s.mu.RLock()
    defer s.mu.RUnlock()
    return s.keys[len(s.keys)-1]
}

// ServeHTTP — JWKS endpoint
func (s *JWKSServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    s.mu.RLock()
    keys := make([]KeyEntry, len(s.keys))
    copy(keys, s.keys)
    s.mu.RUnlock()

    type JWK struct {
        Kty string `json:"kty"`
        Use string `json:"use"`
        Alg string `json:"alg"`
        Kid string `json:"kid"`
        Crv string `json:"crv"`
        X   string `json:"x"`
        Y   string `json:"y"`
    }

    var jwkList []JWK
    for _, entry := range keys {
        pub := entry.Key.PublicKey
        jwkList = append(jwkList, JWK{
            Kty: "EC", Use: "sig", Alg: "ES256",
            Kid: entry.Kid, Crv: "P-256",
            X: b64url(bigIntBytes(pub.X, 32)),
            Y: b64url(bigIntBytes(pub.Y, 32)),
        })
    }

    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("Cache-Control", "public, max-age=900") // 15 минут
    json.NewEncoder(w).Encode(map[string]any{"keys": jwkList})
}
```

---

## Что спрашивают на собеседованиях

1. **Что такое JWKS и зачем он нужен?**
   → Набор публичных ключей (RFC 7517) для верификации JWT. Позволяет не хранить shared secret на каждом сервисе. Auth-сервер публикует публичные ключи на `/.well-known/jwks.json`.

2. **Чем RS256/ES256 лучше HS256 в микросервисах?**
   → При HS256 каждый сервис знает секрет — если один скомпрометирован, все токены под угрозой. При RS256/ES256 только auth-сервер знает приватный ключ; остальные получают публичный ключ из JWKS.

3. **Как работает ротация ключей в JWKS?**
   → Новый ключ добавляется в JWKS рядом со старым (по `kid`). Новые JWT подписываются новым ключом. Старые JWT верифицируются старым ключом до истечения их `exp`. Потом старый ключ удаляется из JWKS.

4. **Как кешировать JWKS и когда рефрешить?**
   → Кешировать на 5–15 минут. Рефрешить при появлении неизвестного `kid` (один раз — защита от DoS). Использовать `Cache-Control` заголовок от сервера.

5. **Что такое alg confusion attack?**
   → Атака, при которой злоумышленник меняет `alg` в заголовке JWT с RS256 на HS256 и подписывает токен публичным ключом как HMAC-секретом. Защита: явно указывать ожидаемый алгоритм при верификации, никогда не доверять `alg` из заголовка.

6. **Что проверять при валидации JWT?**
   → `exp` (не истёк), `nbf` (не раньше), `iss` (правильный издатель), `aud` (правильный получатель), подпись через JWKS, `kid` для выбора ключа.
