# SSH (Secure Shell)

Полезные ссылки:
- [RFC 4251 — SSH Architecture](https://datatracker.ietf.org/doc/html/rfc4251)
- [RFC 4252 — SSH Authentication](https://datatracker.ietf.org/doc/html/rfc4252)
- [golang.org/x/crypto/ssh](https://pkg.go.dev/golang.org/x/crypto/ssh)

---

## Что такое SSH?

**SSH** (Secure Shell) — криптографический протокол для безопасного удалённого управления системами и передачи данных. Заменил небезопасные Telnet и rsh. Работает поверх TCP, порт 22.

**Что обеспечивает:**
- **Аутентификация** — взаимная: клиент проверяет хост, сервер проверяет пользователя.
- **Конфиденциальность** — данные шифруются (AES-256-GCM, ChaCha20-Poly1305).
- **Целостность** — данные не изменены (HMAC/AEAD).

## SSH Handshake

```
Client                         Server
  │                              │
  │──── TCP connect (port 22) ──►│
  │◄─── Banner (SSH-2.0-...) ────│
  │                              │
  │──── Key Exchange Init ───────►│  предлагает алгоритмы
  │◄─── Key Exchange Init ────────│  выбирает алгоритм
  │                              │
  │──── ECDH public key ─────────►│
  │◄─── ECDH public key +         │
  │     Host Key + Signature ─────│  сервер подписывает host key
  │                              │
  │  (клиент проверяет host key   │
  │   в ~/.ssh/known_hosts)       │
  │                              │
  │──── New Keys ────────────────►│  включить шифрование
  │◄─── New Keys ─────────────────│
  │                              │
  │──── User Auth Request ───────►│  public key / password
  │◄─── Auth Success ─────────────│
  │                              │
  │══════════ Channel ════════════│  exec / shell / sftp / tunnel
```

### Верификация хоста (первое подключение)

При первом подключении клиент получает публичный ключ сервера и сохраняет в `~/.ssh/known_hosts`. При повторных подключениях — сверяет. Если ключ изменился → предупреждение о возможном MITM.

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@ WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED! @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
```

## Типы SSH-ключей

| Тип | Размер | Статус | Рекомендация |
|-----|--------|--------|--------------|
| **Ed25519** | 256 бит | Актуален | **Рекомендуется** |
| **ECDSA P-256** | 256 бит | Актуален | Да |
| **RSA** | ≥3072 бит | Устаревает | Только для совместимости |
| **DSA** | 1024 бит | **Сломан** | Никогда |

```bash
# Генерация ключей
ssh-keygen -t ed25519 -C "user@example.com"        # рекомендуется
ssh-keygen -t rsa -b 4096 -C "legacy@example.com"

# Ключи хранятся в:
~/.ssh/id_ed25519       # приватный (защитить chmod 600!)
~/.ssh/id_ed25519.pub   # публичный (можно раздавать)
```

## Методы аутентификации

### 1. Public Key (рекомендуется)

**Процесс:**
1. Клиент предъявляет публичный ключ.
2. Сервер проверяет наличие в `~/.ssh/authorized_keys`.
3. Сервер отправляет challenge (случайные данные).
4. Клиент подписывает challenge приватным ключом.
5. Сервер проверяет подпись публичным ключом → доступ.

```bash
# Скопировать ключ на сервер
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server.com

# Вручную
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### 2. Certificate-based (масштабируемое решение)

```bash
# CA подписывает публичный ключ пользователя
ssh-keygen -s ca_key -I "user@example.com" -n ubuntu -V +30d ~/.ssh/id_ed25519.pub
# → id_ed25519-cert.pub (сертификат)

# Сервер доверяет CA, а не конкретным ключам
# /etc/ssh/sshd_config:
TrustedUserCAKeys /etc/ssh/trusted_ca.pub
```

### 3. Password (не рекомендуется)

Передаётся зашифрованным, но уязвим к брутфорсу. Следует отключать:

```
# /etc/ssh/sshd_config
PasswordAuthentication no
```

## SSH Config (~/.ssh/config)

```
Host myserver
    HostName 192.168.1.100
    User deploy
    IdentityFile ~/.ssh/id_ed25519
    Port 22
    ServerAliveInterval 60
    ServerAliveCountMax 3

Host bastion
    HostName bastion.example.com
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519

Host internal-db
    HostName 10.0.1.50
    User postgres
    ProxyJump bastion    # прыгать через бастион (Go-style ProxyCommand)
```

## Port Forwarding (туннелирование)

```bash
# Local: localhost:8080 → remote-host:80 через SSH-сервер
ssh -L 8080:remote-host:80 user@ssh-server

# Remote: порт 8080 на SSH-сервере → localhost:3000
ssh -R 8080:localhost:3000 user@ssh-server

# Dynamic SOCKS-прокси на localhost:1080
ssh -D 1080 user@ssh-server

# Реальный пример: доступ к PostgreSQL через бастион
ssh -L 5432:prod-db.internal:5432 user@bastion.example.com
# Затем: psql -h localhost -p 5432 -U postgres mydb
```

## SSH Agent

Хранит расшифрованные приватные ключи в памяти — не нужно вводить passphrase каждый раз:

```bash
eval "$(ssh-agent -s)"     # запустить агент
ssh-add ~/.ssh/id_ed25519  # добавить ключ
ssh-add -l                 # список ключей в агенте
ssh-add -d ~/.ssh/id_ed25519  # удалить ключ
```

**Agent Forwarding** (`ssh -A`) — с осторожностью: при компрометации jump-сервера агент доступен атакующему.

## SSH в Go

```go
import "golang.org/x/crypto/ssh"

// Клиент: аутентификация по ключу
keyBytes, _ := os.ReadFile("/home/user/.ssh/id_ed25519")
signer, _ := ssh.ParsePrivateKey(keyBytes)

// Загрузить known_hosts
knownHosts, _ := knownhosts.New(filepath.Join(os.Getenv("HOME"), ".ssh/known_hosts"))

config := &ssh.ClientConfig{
    User: "ubuntu",
    Auth: []ssh.AuthMethod{
        ssh.PublicKeys(signer),
    },
    HostKeyCallback: knownHosts, // проверять known_hosts
    Timeout:         10 * time.Second,
}

conn, err := ssh.Dial("tcp", "server.example.com:22", config)
if err != nil {
    log.Fatal(err)
}
defer conn.Close()

// Выполнить команду
session, _ := conn.NewSession()
defer session.Close()
output, _ := session.Output("uptime")
fmt.Println(string(output))

// Передать файл через SFTP
sftpClient, _ := sftp.NewClient(conn) // golang.org/x/crypto/ssh/sftp
defer sftpClient.Close()
f, _ := sftpClient.Create("/tmp/file.txt")
f.Write([]byte("hello"))
```

```go
// SSH сервер в Go
serverConfig := &ssh.ServerConfig{
    PublicKeyCallback: func(conn ssh.ConnMetadata, key ssh.PublicKey) (*ssh.Permissions, error) {
        // загрузить authorized_keys и сверить
        authorizedKey := loadAuthorizedKey()
        if ssh.FingerprintSHA256(key) == ssh.FingerprintSHA256(authorizedKey) {
            return &ssh.Permissions{Extensions: map[string]string{"user": conn.User()}}, nil
        }
        return nil, fmt.Errorf("unknown key")
    },
}

hostKey, _ := ssh.ParsePrivateKey(hostKeyBytes)
serverConfig.AddHostKey(hostKey)

listener, _ := net.Listen("tcp", ":2222")
for {
    tcpConn, _ := listener.Accept()
    go func(c net.Conn) {
        sshConn, chans, reqs, err := ssh.NewServerConn(c, serverConfig)
        if err != nil {
            return
        }
        go ssh.DiscardRequests(reqs)
        for ch := range chans {
            if ch.ChannelType() == "session" {
                channel, _, _ := ch.Accept()
                go handleSession(channel)
            }
        }
    }(tcpConn)
}
```

## Hardening SSH сервера

```bash
# /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers deploy ubuntu
Port 2222                              # нестандартный порт (security by obscurity)
MaxAuthTries 3
LoginGraceTime 20
ClientAliveInterval 300
ClientAliveCountMax 2
X11Forwarding no
AllowTcpForwarding yes                 # или no

# Современные алгоритмы (OpenSSH 8.x+)
KexAlgorithms curve25519-sha256,ecdh-sha2-nistp256
Ciphers aes256-gcm@openssh.com,chacha20-poly1305@openssh.com
MACs hmac-sha2-256-etm@openssh.com

systemctl restart sshd
```

```bash
# Fail2ban: блокировать брутфорс
apt install fail2ban
# /etc/fail2ban/jail.local
[sshd]
enabled = true
maxretry = 3
bantime = 3600
```

## Что спрашивают на собеседованиях

1. **Как работает аутентификация по ключу?** → Клиент доказывает владение приватным ключом, подписывая challenge от сервера. Сервер верифицирует подпись публичным ключом.
2. **Чем Ed25519 лучше RSA?** → Короче ключ (256 бит), быстрее, основан на Edwards curve (более современная математика), нет risk от плохих случайных чисел (в отличие от ECDSA).
3. **Что такое known_hosts?** → Локальный кэш публичных ключей SSH-серверов для защиты от MITM при подключении.
4. **Как работает Port Forwarding?** → SSH создаёт туннель внутри зашифрованного соединения, перенаправляя TCP-трафик.
5. **Чем сертификаты лучше authorized_keys?** → Не нужно копировать ключи на каждый сервер; достаточно довериться CA; сертификаты имеют TTL.
6. **Agent Forwarding — риски?** → При компрометации промежуточного сервера атакующий может использовать ваш агент для подключения к другим серверам.
