# DNS (Domain Name System)

Полезные ссылки:
- [RFC 1034/1035 — DNS](https://datatracker.ietf.org/doc/html/rfc1035)
- [How DNS Works](https://howdns.works/)

---

## Что такое DNS?

**DNS** (Domain Name System) — распределённая иерархическая система для преобразования доменных имён в IP-адреса (и обратно).

```
example.com → 93.184.216.34
```

DNS — это "телефонный справочник" интернета.

## Иерархия DNS

```
.                           ← Root (корень)
├── com.                    ← TLD (Top Level Domain)
│   ├── example.com.        ← Second Level Domain
│   │   ├── www.example.com ← Subdomain
│   │   └── api.example.com
├── org.
└── ru.
```

## Типы DNS-записей

| Тип | Описание | Пример |
|-----|---------|--------|
| `A` | Домен → IPv4 | `example.com → 93.184.216.34` |
| `AAAA` | Домен → IPv6 | `example.com → 2606:2800::1` |
| `CNAME` | Алиас на другой домен | `www → example.com` |
| `MX` | Почтовый сервер | `example.com → mail.example.com` |
| `TXT` | Произвольный текст | SPF, DKIM, верификация |
| `NS` | Name Server зоны | `example.com → ns1.dnshosting.com` |
| `SOA` | Start of Authority | Метаданные зоны |
| `PTR` | IP → домен (reverse DNS) | `34.216.184.93.in-addr.arpa → example.com` |
| `SRV` | Сервис + порт | `_grpc._tcp.example.com → server:443` |
| `CAA` | Авторизованные CA | |

## Процесс разрешения (DNS Resolution)

```
Browser → OS Cache → /etc/hosts → Local Resolver
    → Root Nameserver → TLD Nameserver → Authoritative NS → IP
```

**Полный путь для `api.example.com`:**

1. **Браузер** проверяет кэш.
2. **OS** проверяет `/etc/hosts`.
3. **Recursive Resolver** (ваш провайдер / 8.8.8.8):
   - Запрашивает **Root NS**: "кто ведёт .com?"
   - Root NS отвечает: адрес **TLD NS** для `.com`.
   - Запрашивает TLD NS: "кто ведёт `example.com`?"
   - TLD NS отвечает: адрес **Authoritative NS** для `example.com`.
   - Запрашивает Authoritative NS: "IP для `api.example.com`?"
   - Получает A-запись: `93.184.216.34`.
4. Recursive Resolver кэширует ответ (TTL).
5. Возвращает IP браузеру.

## Recursive vs Iterative Resolver

- **Recursive**: клиент делает один запрос → resolver делает всю цепочку сам.
- **Iterative**: клиент делает запросы самостоятельно на каждом уровне.

На практике клиенты используют recursive resolver (провайдер, 8.8.8.8, 1.1.1.1).

## TTL (Time To Live)

Время кэширования DNS-ответа в секундах.

- Малый TTL (60–300с) — быстрое распространение изменений, больше нагрузки на NS.
- Большой TTL (3600–86400с) — медленное обновление, меньше нагрузки.

**Перед сменой IP:** сначала уменьши TTL, подожди, смени IP, потом верни TTL.

## DNS транспорт

| Протокол | Описание |
|----------|---------|
| DNS over UDP (53) | Стандарт, UDP, ≤ 512 байт |
| DNS over TCP (53) | Для больших ответов (>512 байт, AXFR) |
| DoT (853) | DNS over TLS — шифрование |
| DoH (443) | DNS over HTTPS — шифрование + прокси |
| DoQ | DNS over QUIC (RFC 9250) |

## Зоны и делегирование

**Зона** — часть DNS-пространства имён, за которую отвечает конкретный NS.

```
example.com  зона:  NS1.example.com, NS2.example.com
  ├── api.example.com   → в зоне example.com
  └── shop.example.com → делегировано → NS3.shop.example.com
```

**Делегирование**: NS-запись указывает другой сервер отвечать за поддомен.

## DNS в микросервисах (Service Discovery)

```
# SRV запись для service discovery
_grpc._tcp.user-service.svc.cluster.local → server:50051

# В Kubernetes
user-service.production.svc.cluster.local → ClusterIP
```

Go пример:
```go
// Lookup A
addrs, err := net.LookupHost("api.example.com")

// Lookup SRV (service discovery)
_, addrs, err := net.LookupSRV("grpc", "tcp", "user-service.svc.cluster.local")
for _, addr := range addrs {
    fmt.Printf("%s:%d\n", addr.Target, addr.Port)
}
```

## Атаки и защита

| Атака | Описание | Защита |
|-------|---------|--------|
| DNS Spoofing | Подмена DNS-ответа | DNSSEC |
| DNS Cache Poisoning | Отравление кэша resolver | DNSSEC, рандомизация портов |
| DNS Amplification (DDoS) | Усиление через открытые resolver | Rate limiting, BCP38 |
| DNS Hijacking | Перехват запросов | DoH, DoT |

**DNSSEC** — цифровая подпись DNS-записей. Клиент проверяет подпись цепочкой до Root NS.

## Что спрашивают на собеседованиях

1. **Что такое DNS?** → Распределённая система для преобразования имён в IP.
2. **Как работает DNS-запрос?** → Кэш → /etc/hosts → recursive resolver → root → TLD → authoritative.
3. **Что такое TTL в DNS?** → Время кэширования ответа.
4. **Разница A и CNAME?** → A: домен → IP; CNAME: алиас на другой домен.
5. **Что такое DNS Cache Poisoning?** → Злоумышленник подменяет запись в кэше resolver.
6. **Как DNS используется в микросервисах?** → Service discovery через SRV-записи или DNS-имена (Kubernetes).
