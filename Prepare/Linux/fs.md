# Linux: Файловая система, процессы, сигналы

Полезные ссылки:
- [The Linux Command Line](https://linuxcommand.org/tlcl.php)
- [Linux Kernel Documentation](https://www.kernel.org/doc/html/latest/)

---

## Как устроена ext4: inode

**Inode** (index node) — структура данных, хранящая метаданные файла:
- Права доступа (permissions).
- Владелец (UID/GID).
- Размер файла.
- Временные метки (atime, mtime, ctime).
- Указатели на блоки данных на диске.

**Чего НЕТ в inode:** имени файла. Имя хранится в **директории**.

```
Директория:
  "file.txt" → inode 12345
  "image.png" → inode 67890

Inode 12345:
  mode: -rw-r--r--
  uid: 1001
  size: 4096
  blocks: [block_1, block_2, ...]
```

```bash
ls -i file.txt      # показать inode номер
stat file.txt       # полная информация
df -i               # использование inodes на ФС
```

## Hardlink vs Symlink

**Hardlink** — ещё одно имя для того же inode:
- Указывает на те же данные.
- Удаление одного hardlink не удаляет файл (пока есть другие).
- Нельзя между разными ФС.
- Нельзя для директорий.

```bash
ln file.txt hardlink.txt
ls -i file.txt hardlink.txt
# 12345 file.txt
# 12345 hardlink.txt  ← тот же inode!
```

**Symlink** (Soft link) — ссылка на путь (имя файла):
- Хранит путь как строку.
- Если оригинал удалён → symlink "сломан".
- Работает между ФС и для директорий.
- Имеет свой inode.

```bash
ln -s /path/to/file.txt symlink.txt
ls -la symlink.txt
# lrwxrwxrwx 1 user group 9 Jan 21 symlink.txt -> file.txt
```

## Где хранятся названия файлов

В **директории**. Директория — особый файл-таблица `(имя) → (inode)`. Inode не знает своего имени.

## Флаги доступа

```
-rwxr-xr-x  1  user  group  4096  Jan 21  file
│└┬┘└┬┘└┬┘
│ │  │  └── others: r-x = 5
│ │  └───── group:  r-x = 5
│ └──────── owner:  rwx = 7
└────────── тип: - файл, d директория, l symlink

r=4, w=2, x=1
chmod 755 file   # rwxr-xr-x
chmod 644 file   # rw-r--r--
chmod +x file
```

**Специальные биты:**
- `SUID` (4xxx): выполняется с правами владельца (`-rwsr-xr-x`). Пример: `/usr/bin/passwd`.
- `SGID` (2xxx): выполняется с правами группы.
- `Sticky bit` (1xxx): удалить файл в директории может только его владелец (`/tmp`: `drwxrwxrwt`).

## Что делает команда kill?

`kill` **не убивает** — отправляет **сигнал** процессу:

```bash
kill -SIGTERM 1234   # вежливое завершение (15)
kill -SIGKILL 1234   # принудительное (9) — нельзя перехватить
kill -SIGHUP 1234    # перезагрузить конфиг
kill -0 1234         # проверить существование без сигнала
pkill nginx          # по имени
```

| Сигнал | Номер | Описание |
|--------|-------|---------|
| `SIGTERM` | 15 | Вежливое завершение (можно перехватить) |
| `SIGKILL` | 9 | Принудительное (нельзя перехватить!) |
| `SIGINT` | 2 | Ctrl+C |
| `SIGHUP` | 1 | Перезагрузить конфиг (в демонах) |
| `SIGUSR1/2` | 10/12 | Пользовательские |
| `SIGPIPE` | 13 | Запись в закрытый pipe |

**SIGTERM vs SIGKILL:**
- `SIGTERM` — процесс может завершиться gracefully (закрыть соединения, flush данных).
- `SIGKILL` — ядро убивает немедленно, без шансов.

### Graceful shutdown в Go

```go
func main() {
    stop := make(chan os.Signal, 1)
    signal.Notify(stop, syscall.SIGTERM, syscall.SIGINT)

    // ... запускаем сервер ...

    <-stop
    log.Println("shutting down...")

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    server.Shutdown(ctx)
    db.Close()
}
```

## procfs и sysfs

### /proc — виртуальная ФС

`/proc` — интерфейс к данным ядра. Создаётся при загрузке, не хранится на диске.

```bash
/proc/1/              # процесс PID 1
/proc/1/cmdline       # командная строка
/proc/1/status        # VmRSS, threads, etc.
/proc/1/fd/           # открытые файловые дескрипторы
/proc/1/maps          # маппинг памяти

/proc/cpuinfo         # информация о CPU
/proc/meminfo         # информация о памяти
/proc/net/tcp         # активные TCP соединения
/proc/loadavg         # средняя нагрузка (1, 5, 15 минут)

cat /proc/meminfo | grep MemAvailable
cat /proc/$(pgrep myapp)/status | grep VmRSS
```

### /sys — sysfs

Интерфейс к устройствам и параметрам ядра:

```bash
/sys/class/net/        # сетевые интерфейсы
/sys/block/            # блочные устройства
/sys/kernel/mm/        # параметры управления памятью

sysctl -w vm.swappiness=0      # изменить параметр ядра
sysctl -a | grep tcp_keepalive # все параметры ядра
```

## Что такое git?

**Git** — распределённая система контроля версий. Хранит историю изменений файлов через граф коммитов.

### Commit, Tag, Branch

| Понятие | Описание |
|---------|---------|
| **Commit** | Снимок состояния + метаданные (SHA-1 хеш, автор, дата). Неизменяем. |
| **Branch** | Подвижная ссылка на коммит. При новом commit — автоматически перемещается. |
| **Tag** | Статичная ссылка на конкретный коммит (для версий: `v1.0.0`). |
| **HEAD** | Указатель на текущую позицию. |

```bash
# Ветки
git branch feature/api
git checkout -b feature/api  # создать + переключиться

# Теги
git tag -a v1.0.0 -m "Release 1.0.0"
git push origin v1.0.0

# Базовые команды
git status
git add -p                    # интерактивное добавление
git commit -m "message"
git log --oneline --graph --all

# Откаты
git reset HEAD~1              # отменить коммит (оставить изменения)
git reset --hard HEAD~1       # отменить + удалить изменения
git revert abc123             # безопасная отмена (новый коммит)
git restore file.go           # восстановить файл из HEAD
```

## Полезные команды для Go разработчика

```bash
# Открытые файловые дескрипторы
ls -la /proc/$(pgrep myapp)/fd | wc -l

# Использование памяти Go процесса
cat /proc/$(pgrep myapp)/status | grep -E "VmRSS|VmPeak"

# Лимиты дескрипторов
ulimit -n           # текущий лимит
ulimit -Sn 65536    # увеличить мягкий лимит

# Порты
ss -tulnp | grep 8080
lsof -i :8080

# Мониторинг I/O
iostat -x 1
iotop -o            # только активные процессы
```
