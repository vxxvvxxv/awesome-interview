# Grafana

Полезные ссылки:
- [Grafana Documentation](https://grafana.com/docs/)
- [Grafana Dashboards (community)](https://grafana.com/grafana/dashboards/)
- [LGTM Stack](https://grafana.com/go/webinar/lgtm-grafana-loki-grafana-tempo-grafana-mimir/)
- [Grafana Go App Plugin](https://grafana.com/developers/plugin-tools/)

---

## Что такое Grafana?

**Grafana** — open-source платформа для визуализации и мониторинга. Подключается к различным источникам данных и строит дашборды, алерты.

**Применение:**
- Дашборды для метрик (Prometheus, InfluxDB, CloudWatch).
- Логи (Loki).
- Трейсы (Jaeger, Tempo).
- Алертинг.
- Business аналитика (PostgreSQL, MySQL).

## Источники данных (Data Sources)

| Источник | Тип данных | Применение |
|----------|-----------|-----------|
| **Prometheus** | Метрики (time series) | Инфраструктура, приложения |
| **Loki** | Логи | Поиск по логам |
| **Jaeger/Tempo** | Трейсы | Distributed tracing |
| **InfluxDB** | Time series | IoT, метрики |
| **PostgreSQL/MySQL** | Реляционные | Бизнес данные |
| **Elasticsearch** | Документы + логи | Поиск, логи |
| **CloudWatch** | AWS метрики | AWS мониторинг |
| **Graphite** | Метрики | Legacy |

```bash
# Добавить Data Source через Grafana UI:
# Configuration → Data Sources → Add data source

# Или через provisioning (Infrastructure as Code):
# /etc/grafana/provisioning/datasources/prometheus.yaml
```

## LGTM Stack (рекомендуемый стек)

```
┌─────────────────────────────────────────────┐
│                  Grafana                    │
│      (Visualize all observability data)     │
└────────┬────────┬──────────┬────────────────┘
         │        │          │
       Loki     Grafana   Grafana
      (Logs)   Tempo     Mimir
               (Traces)  (Metrics)
         │        │          │
    Promtail  OpenTelemetry  Prometheus
    (agents)  Collector      scrape
```

- **L**oki — логи (индексирует только labels, не текст → дёшево).
- **G**rafana — дашборды.
- **T**empo — трейсы.
- **M**imir (или Prometheus) — метрики.

## Типы панелей (Panels)

| Панель | Применение |
|--------|-----------|
| **Time series** | Метрики во времени (основная) |
| **Stat** | Одно число с тенденцией |
| **Gauge** | Прогресс-бар (0-100%) |
| **Bar gauge** | Горизонтальный bar chart |
| **Table** | Табличные данные |
| **Pie chart** | Доли |
| **Heatmap** | Плотность распределения (latency percentiles) |
| **Logs** | Поток логов (Loki) |
| **Node graph** | Граф зависимостей |
| **Traces** | Waterfall трейсы |

## Grafana + Prometheus

```yaml
# /etc/grafana/provisioning/datasources/prometheus.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    access: proxy
    isDefault: true
    jsonData:
      timeInterval: "15s"
      httpMethod: POST
```

### PromQL в Grafana

```promql
# HTTP RPS
rate(http_requests_total{job="myapp"}[$__rate_interval])

# P99 latency
histogram_quantile(0.99,
  rate(http_request_duration_seconds_bucket{job="myapp"}[$__rate_interval]))

# Error rate
rate(http_requests_total{status=~"5.."}[$__rate_interval])
  /
rate(http_requests_total[$__rate_interval])

# Go goroutines
go_goroutines{job="myapp"}

# Memory usage
process_resident_memory_bytes{job="myapp"} / 1024 / 1024
```

**Важно:** `$__rate_interval` — магическая переменная Grafana, автоматически выбирает оптимальный range для `rate()`.

## Grafana + Loki (логи)

```yaml
# /etc/grafana/provisioning/datasources/loki.yml
apiVersion: 1
datasources:
  - name: Loki
    type: loki
    url: http://loki:3100
    access: proxy
```

### LogQL запросы

```logql
# Все логи из сервиса
{app="myapp"}

# Логи с ошибками
{app="myapp"} |= "ERROR"

# Regex фильтрация
{app="myapp"} |~ "error|panic|fatal"

# Парсинг JSON логов
{app="myapp"} | json | level = "error"

# Метрики из логов (log rate)
rate({app="myapp"}[5m])

# Подсчёт ошибок по типу
sum by (error_type) (
  count_over_time({app="myapp"} | json | level="error" [5m])
)
```

### Promtail (агент сбора логов)

```yaml
# promtail-config.yml
clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: myapp
    static_configs:
      - targets: [localhost]
        labels:
          app: myapp
          __path__: /var/log/myapp/*.log

    pipeline_stages:
      - json:
          expressions:
            level: level
            trace_id: trace_id
      - labels:
          level:
          trace_id:
```

### Логи из Go в Loki

```go
// Структурированные логи (JSON) + Promtail читает файл
import "log/slog"

logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))

logger.Info("request processed",
    slog.String("method", "GET"),
    slog.String("path", "/users"),
    slog.Int("status", 200),
    slog.Duration("duration", 52*time.Millisecond),
    slog.String("trace_id", traceID),
)

// Output: {"time":"2024-01-15T10:00:00Z","level":"INFO","msg":"request processed","method":"GET",...}
```

## Grafana + Tempo (трейсы)

```yaml
# datasource: tempo
datasources:
  - name: Tempo
    type: tempo
    url: http://tempo:3200
    jsonData:
      tracesToLogsV2:
        datasourceUid: loki-uid  # связать трейсы с логами!
        tags: [{key: "service.name", value: "app"}]
      serviceMap:
        datasourceUid: prometheus-uid
```

**Correlation:** нажав на span в трейсе → Grafana откроет соответствующие логи в Loki.

## Дашборды как код

### Provisioning

```yaml
# /etc/grafana/provisioning/dashboards/myapp.yml
apiVersion: 1
providers:
  - name: myapp-dashboards
    type: file
    options:
      path: /etc/grafana/dashboards
      # watchChanges: true — авто-перезагрузка
```

```json
// /etc/grafana/dashboards/myapp.json (экспорт из UI)
{
  "title": "My App",
  "panels": [...],
  "templating": {"list": [...]},
  "time": {"from": "now-1h", "to": "now"},
  "refresh": "30s"
}
```

### Grafonnet (Jsonnet библиотека)

```jsonnet
// dashboard.jsonnet
local grafana = import 'grafonnet/grafana.libsonnet';
local dashboard = grafana.dashboard;
local row = grafana.row;
local prometheus = grafana.prometheus;
local graphPanel = grafana.graphPanel;

dashboard.new('My App Dashboard')
.addPanel(
  graphPanel.new('HTTP RPS')
  .addTarget(
    prometheus.target('rate(http_requests_total[5m])', legendFormat='{{method}} {{path}}')
  ),
  gridPos={x: 0, y: 0, w: 12, h: 8}
)
```

```bash
jsonnet -J vendor dashboard.jsonnet -o dashboard.json
```

## Алертинг в Grafana

```yaml
# Grafana Unified Alerting (8.0+)
# Создаётся через UI или provisioning

# provisioning/alerting/rules.yml
groups:
  - name: myapp
    interval: 1m
    rules:
      - uid: high-error-rate
        title: "High Error Rate"
        condition: C
        data:
          - refId: A
            queryType: ''
            relativeTimeRange:
              from: 600
              to: 0
            model:
              expr: |
                rate(http_requests_total{status=~"5.."}[5m])
                  /
                rate(http_requests_total[5m]) > 0.05
          - refId: C
            queryType: ''
            relativeTimeRange:
              from: 10
              to: 0
            model:
              type: threshold
              conditions:
                - evaluator:
                    type: gt
                    params: [0]
        noDataState: NoData
        execErrState: Error
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Error rate > 5%"
```

### Notification channels

```yaml
# provisioning/alerting/contact-points.yml
contactPoints:
  - name: slack-ops
    receivers:
      - uid: slack-ops-uid
        type: slack
        settings:
          url: "$SLACK_WEBHOOK_URL"
          channel: "#ops-alerts"
          title: "{{ template \"slack.default.title\" . }}"
          text: "{{ template \"slack.default.text\" . }}"
```

## Дашборд для Go приложения

```
Рекомендуемые панели для Go микросервиса:
1. HTTP RPS (rate(http_requests_total[5m]))
2. P99 Latency (histogram_quantile(0.99, ...))
3. Error Rate (5xx / total)
4. Goroutines (go_goroutines)
5. Memory (go_memstats_heap_alloc_bytes)
6. GC pause (go_gc_duration_seconds)
7. CPU Usage (rate(process_cpu_seconds_total[5m]))
8. Active Connections
9. DB connection pool stats
10. Logs panel (Loki: {app="myapp"} |= "ERROR")
```

**Готовые дашборды:**
- [Go Processes](https://grafana.com/grafana/dashboards/6671) — стандартный Go дашборд
- [Node Exporter Full](https://grafana.com/grafana/dashboards/1860) — система
- [Kubernetes](https://grafana.com/grafana/dashboards/315) — k8s метрики

## Переменные в дашбордах (Variables/Templating)

```
# В UI: Dashboard Settings → Variables

Variable: namespace
Type: Query
Query: label_values(up, namespace)

Variable: pod
Type: Query  
Query: label_values(up{namespace="$namespace"}, pod)

# Использование в панели:
rate(http_requests_total{namespace="$namespace", pod="$pod"}[5m])
```

## Что спрашивают на собеседованиях

1. **Что такое Grafana?** → Платформа для визуализации данных мониторинга. Подключается к Prometheus, Loki, Tempo и другим источникам.
2. **LGTM стек — что это?** → Loki (логи) + Grafana (визуализация) + Tempo (трейсы) + Mimir/Prometheus (метрики). Полный observability стек от Grafana Labs.
3. **Как строить дашборды как код?** → Provisioning через YAML файлы (datasources, дашборды JSON, alerting rules).
4. **Что такое LogQL?** → Query language для Loki (как PromQL для Prometheus). `{app="myapp"} |= "ERROR"`.
5. **Как связать трейсы и логи в Grafana?** → Correlation через `trace_id` в логах; настройка в datasource Tempo → tracesToLogsV2.
6. **Для чего нужны переменные в дашборде?** → Фильтрация по namespace, сервису, pod без дублирования панелей.
