# WebAuthn / FIDO2 (Passwordless Authentication)

Полезные ссылки:
- [webauthn.guide](https://webauthn.guide/)
- [W3C — Web Authentication API](https://w3c.github.io/webauthn/)
- [FIDO2 Spec](https://fidoalliance.org/fido2/)
- [go-webauthn/webauthn](https://github.com/go-webauthn/webauthn)

---

## Что такое WebAuthn?

**WebAuthn** (Web Authentication API) — стандарт W3C для беспарольной и многофакторной аутентификации на основе публичных ключей. Часть спецификации **FIDO2**.

**Проблема паролей:**
- Пользователи повторно используют пароли.
- Уязвимы к фишингу, утечкам БД.
- Сложно запоминать.

**WebAuthn решает:**
- Аутентификатор генерирует пару ключей **на устройстве**.
- Приватный ключ **никогда не передаётся** серверу.
- Защита от фишинга через привязку к `origin` (URL).

## Ключевые понятия

| Термин | Описание |
|--------|---------|
| **Relying Party (RP)** | Веб-сервис (ваш backend). Идентифицируется через `rpId` (домен). |
| **Authenticator** | Устройство/ПО, хранящее приватный ключ. |
| **Credential** | Пара ключей, созданная для конкретного RP. |
| **Challenge** | Случайные данные от сервера (nonce), защита от replay. |
| **Attestation** | Доказательство того, что аутентификатор легитимный. |
| **Assertion** | Доказательство наличия приватного ключа (при входе). |

## Типы аутентификаторов

### Platform Authenticator (встроенный)
- Touch ID, Face ID (Apple).
- Windows Hello.
- Android Fingerprint/PIN.

**Плюсы:** всегда под рукой. **Минусы:** привязан к устройству.

### Cross-platform Authenticator (внешний)
- YubiKey, Google Titan Key.
- Смартфон как ключ (Passkeys в iOS/Android).

**Плюсы:** переносной. **Минусы:** можно потерять.

## Как работает: Registration Flow

```
Browser                    RP Server                  Authenticator
  │                            │                            │
  │──── GET /register ────────►│                            │
  │◄─── challenge + options ───│                            │
  │     {                      │                            │
  │       challenge: random,   │                            │
  │       rpId: "example.com", │                            │
  │       userId: "user123"    │                            │
  │     }                      │                            │
  │                            │                            │
  │──── navigator.credentials                               │
  │     .create(options) ─────────────────────────────────►│
  │                            │          [биометрия/PIN]   │
  │                            │      генерирует ключевую   │
  │                            │      пару, сохраняет priv  │
  │◄─── credential ────────────────────────────────────────│
  │     {publicKey, attestation, clientDataJSON}            │
  │                            │                            │
  │──── POST /register ───────►│                            │
  │     {credential}           │                            │
  │                       верифицирует                      │
  │                       сохраняет publicKey               │
  │◄─── 200 OK ────────────────│                            │
```

## Как работает: Authentication Flow

```
Browser                    RP Server                  Authenticator
  │                            │                            │
  │──── POST /login ──────────►│                            │
  │◄─── challenge + credIds ───│                            │
  │                            │                            │
  │──── navigator.credentials                               │
  │     .get(options) ────────────────────────────────────►│
  │                            │          [биометрия/PIN]   │
  │                            │      подписывает challenge  │
  │                            │      приватным ключом       │
  │◄─── assertion ─────────────────────────────────────────│
  │     {signature, clientDataJSON, authenticatorData}      │
  │                            │                            │
  │──── POST /verify ─────────►│                            │
  │     {assertion}            │                            │
  │                       верифицирует подпись              │
  │                       сохранённым publicKey             │
  │◄─── 200 OK / token ────────│                            │
```

## WebAuthn в Go

```go
import "github.com/go-webauthn/webauthn/webauthn"

// Инициализация
wau, _ := webauthn.New(&webauthn.Config{
    RPDisplayName: "My App",
    RPID:          "example.com",
    RPOrigins:     []string{"https://example.com"},
})

// Реализовать интерфейс webauthn.User
type User struct {
    ID          []byte
    Name        string
    DisplayName string
    Credentials []webauthn.Credential
}

func (u User) WebAuthnID() []byte                         { return u.ID }
func (u User) WebAuthnName() string                       { return u.Name }
func (u User) WebAuthnDisplayName() string                { return u.DisplayName }
func (u User) WebAuthnCredentials() []webauthn.Credential { return u.Credentials }

// === REGISTRATION ===

// Шаг 1: сервер генерирует options
func beginRegistration(w http.ResponseWriter, r *http.Request) {
    user := getUser(r)
    options, sessionData, _ := wau.BeginRegistration(user)
    session.Set("registration", sessionData)
    json.NewEncoder(w).Encode(options)
}

// Шаг 2: проверяем ответ от браузера
func finishRegistration(w http.ResponseWriter, r *http.Request) {
    user := getUser(r)
    sessionData := session.Get("registration").(webauthn.SessionData)

    credential, err := wau.FinishRegistration(user, sessionData, r)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    user.Credentials = append(user.Credentials, *credential)
    saveUser(user)
}

// === AUTHENTICATION ===

func beginLogin(w http.ResponseWriter, r *http.Request) {
    user := getUser(r)
    options, sessionData, _ := wau.BeginLogin(user)
    session.Set("auth", sessionData)
    json.NewEncoder(w).Encode(options)
}

func finishLogin(w http.ResponseWriter, r *http.Request) {
    user := getUser(r)
    sessionData := session.Get("auth").(webauthn.SessionData)

    credential, err := wau.FinishLogin(user, sessionData, r)
    if err != nil {
        http.Error(w, "Unauthorized", http.StatusUnauthorized)
        return
    }
    // Обновить счётчик (защита от replay)
    updateCredentialCounter(credential)
    issueJWT(w, user)
}
```

## JavaScript (Browser API)

```javascript
// Registration
const options = await fetch('/register/begin').then(r => r.json())
// options.challenge — base64url encoded bytes
options.challenge = base64urlDecode(options.challenge)
options.user.id = base64urlDecode(options.user.id)

const credential = await navigator.credentials.create({ publicKey: options })

await fetch('/register/finish', {
    method: 'POST',
    body: JSON.stringify({
        id: credential.id,
        rawId: base64urlEncode(credential.rawId),
        response: {
            clientDataJSON: base64urlEncode(credential.response.clientDataJSON),
            attestationObject: base64urlEncode(credential.response.attestationObject),
        },
        type: credential.type,
    }),
})

// Authentication
const options = await fetch('/login/begin').then(r => r.json())
options.challenge = base64urlDecode(options.challenge)

const assertion = await navigator.credentials.get({ publicKey: options })
// отправить assertion на сервер для верификации
```

## Passkeys

**Passkeys** — эволюция WebAuthn:
- Синхронизируются между устройствами через облако (Apple iCloud Keychain, Google Password Manager).
- Один ключ работает на всех устройствах пользователя.
- Поддержка: Safari 16+, Chrome 108+, Firefox 122+.

```
Resident Credential (Discoverable Credential):
- Хранится на аутентификаторе/облаке
- Не нужен список credentialIds от сервера
- Пользователь выбирает аккаунт сам (user handle)
```

## Attestation (для корпоративных требований)

**Attestation** — аутентификатор доказывает свою модель/производителя:

| Тип | Описание |
|-----|---------|
| `none` | Нет аттестации (passkeys, большинство сценариев) |
| `packed` | Aутентификатор сам подписывает |
| `fido-u2f` | Для аппаратных FIDO1 токенов |
| `android-key` | Android Keystore |
| `tpm` | Trusted Platform Module |

```go
// Только TPM и platform authenticators (Enterprise)
options, sessionData, _ := wau.BeginRegistration(user,
    webauthn.WithAuthenticatorSelection(protocol.AuthenticatorSelection{
        AuthenticatorAttachment: protocol.Platform,
        UserVerification:        protocol.VerificationRequired,
        ResidentKey:             protocol.ResidentKeyRequired,
    }),
    webauthn.WithConveyancePreference(protocol.PreferDirectAttestation),
)
```

## Сравнение с другими методами аутентификации

| Метод | Фишинг | Брутфорс | Утечки БД | UX |
|-------|--------|----------|-----------|-----|
| Пароль | Уязвим | Уязвим | Уязвим | Плохо |
| TOTP (Google Auth) | Частично | Защита | Уязвим | Средне |
| SMS OTP | Уязвим | Защита | Защита | Средне |
| **WebAuthn** | **Защита** | **Защита** | **Защита** | Хорошо |
| **Passkeys** | **Защита** | **Защита** | **Защита** | **Отлично** |

## Что спрашивают на собеседованиях

1. **Что такое WebAuthn?** → Стандарт беспарольной аутентификации на публичных ключах; приватный ключ никогда не покидает устройство.
2. **Как защищает от фишинга?** → Credential привязан к `rpId` (домену); на фишинговом сайте браузер не предоставит подходящий ключ.
3. **Что такое challenge?** → Случайный nonce от сервера, который аутентификатор подписывает — защита от replay-атак.
4. **Passkeys vs WebAuthn?** → Passkeys = WebAuthn + синхронизация через облако (Apple/Google); одни ключи на всех устройствах.
5. **Зачем counter в credential?** → Монотонно возрастающий счётчик подписей; защита от клонирования аутентификатора (replay).
6. **Что такое Attestation?** → Доказательство легитимности аутентификатора (производитель, модель); нужно в корпоративных средах.
