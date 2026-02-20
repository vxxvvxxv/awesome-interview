# Docker

Полезные ссылки:
- [Официальная документация](https://docs.docker.com/)
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [Docker Compose reference](https://docs.docker.com/compose/compose-file/)

---

## Что такое Docker?

**Docker** — платформа контейнеризации. Контейнер — изолированный процесс с собственной файловой системой и сетью, но **использующий ядро хост-системы**.

**Контейнер vs VM:**

| | Контейнер | VM |
|--|-----------|-----|
| Изоляция | Namespace + cgroup | Гипервизор |
| Ядро ОС | Общее с хостом | Своё |
| Старт | Миллисекунды | Минуты |
| Размер | МБ | ГБ |
| Overhead | Минимальный | Высокий |

## Как Docker изолирует контейнеры

Механизмы ядра Linux:

- **Namespaces** — изоляция ресурсов:
  - `pid` — процессы (контейнер видит только свои)
  - `net` — своя сетевая карта и IP
  - `mnt` — файловая система
  - `uts` — hostname
  - `ipc` — межпроцессное взаимодействие
  - `user` — пользователи и права

- **cgroups** — ограничение CPU, RAM, I/O.

- **OverlayFS** — слоистая файловая система для образов.

## Ключевые концепции

- **Image** — read-only шаблон (набор слоёв OverlayFS).
- **Container** — запущенный экземпляр image.
- **Layer** — слой файловой системы (копируется при изменении — copy-on-write).
- **Registry** — хранилище образов (Docker Hub, GHCR).
- **Dockerfile** — инструкции для сборки image.

## Dockerfile для Go-приложения

### Многоэтапная сборка (Multi-stage Build)

```dockerfile
# Этап 1: сборка
FROM golang:1.26-alpine AS builder

WORKDIR /app

# Зависимости отдельным слоем (кэширование)
COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-w -s" \
    -o /app/server \
    ./cmd/server

# Этап 2: минимальный образ
FROM scratch
# или FROM gcr.io/distroless/static-debian12

COPY --from=builder /app/server /server
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

EXPOSE 8080
USER 65534:65534
ENTRYPOINT ["/server"]
```

**Преимущества multi-stage:**
- Финальный образ `scratch` ≈ 10 МБ вместо 300+ МБ с Go runtime.
- Нет лишних инструментов в production-образе — меньше attack surface.

### Флаги сборки

```bash
CGO_ENABLED=0    # статическая сборка без libc
GOOS=linux       # целевая ОС
-ldflags="-w -s" # -w: без DWARF, -s: без symbol table → меньший бинарник
```

## Слои и кэширование

```dockerfile
# ПЛОХО: любое изменение кода сбрасывает кэш go mod download
COPY . .
RUN go mod download
RUN go build .

# ХОРОШО: go.mod меняется редко → кэш устойчив
COPY go.mod go.sum ./
RUN go mod download    # кэшируется
COPY . .
RUN go build .
```

## .dockerignore

```
.git
*.md
tmp/
vendor/
*.test
.env
```

## Основные команды

```bash
# Сборка
docker build -t myapp:latest .
docker build --no-cache -t myapp:latest .

# Запуск
docker run -p 8080:8080 myapp:latest
docker run -d --name myapp -p 8080:8080 myapp:latest    # в фоне
docker run -e DATABASE_URL=postgres://... myapp:latest
docker run --env-file .env myapp:latest
docker run -v /host/data:/app/data myapp:latest         # монтирование

# Управление
docker ps                  # запущенные
docker ps -a               # все
docker logs -f myapp       # логи (follow)
docker exec -it myapp sh   # войти в контейнер
docker stop myapp && docker rm myapp

# Очистка
docker system prune -f     # неиспользуемые ресурсы
```

## Docker Compose

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

```bash
docker compose up -d           # запуск в фоне
docker compose down            # остановка (тома сохраняются)
docker compose down -v         # остановка + удаление томов
docker compose logs -f app
docker compose exec app sh
```

## Сети в Docker

```
bridge  — виртуальная сеть (контейнеры общаются по имени сервиса)
host    — контейнер использует сеть хоста напрямую
none    — нет сети
overlay — для Docker Swarm (между нодами)
```

## Управление ресурсами

```bash
docker run \
  --memory="512m" \      # лимит RAM
  --memory-swap="512m" \ # swap = 0 (= memory → нет swap)
  --cpus="1.5" \         # 1.5 CPU ядра
  myapp
```

## Health Check

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD wget -qO- http://localhost:8080/health || exit 1
```

## Безопасность

```dockerfile
# Не запускать от root
RUN adduser -D -u 1001 appuser
USER appuser

# Capabilities
docker run --cap-drop=ALL --security-opt=no-new-privileges myapp
```

## Что спрашивают на собеседованиях

1. **Чем контейнер отличается от VM?** → namespace + cgroup vs гипервизор; общее ядро.
2. **Зачем multi-stage build?** → минимальный образ, нет исходников и инструментов.
3. **Как работает кэш слоёв?** → каждая инструкция = слой; ставь COPY go.mod до COPY . .
4. **Что такое OverlayFS?** → слои: нижние readonly, верхний writable (copy-on-write).
5. **Как контейнеры общаются между собой?** → bridge-сеть, по имени сервиса.
6. **Volume vs bind mount?** → volume управляется Docker (данные в /var/lib/docker/volumes), bind mount — директория хоста.
