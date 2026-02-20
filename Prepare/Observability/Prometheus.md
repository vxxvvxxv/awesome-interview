# Prometheus

Полезные ссылки:
- [Официальная документация](https://prometheus.io/docs/)
- [Grafana + Prometheus](https://grafana.com/docs/grafana/latest/datasources/prometheus/)
- [prometheus/client_golang](https://github.com/prometheus/client_golang)

---

## Что такое Prometheus?

**Prometheus** — система мониторинга и оповещения с открытым исходным кодом (CNCF). Работает по модели **pull**: сам опрашивает сервисы, не получает данные от них.

**Основные компоненты:**

```
┌─────────────────────────────────────────────────┐
│  Prometheus Server                              │
│  ┌──────────┐  ┌────────────┐  ┌───────────┐   │
│  │ Scrape   │  │ TSDB       │  │ Query     │   │
│  │ (Pull)   │  │ (хранилище)│  │ Engine    │   │
│  └──────────┘  └────────────┘  └───────────┘   │
└─────────────────────────────────────────────────┘
       │                                │
  /metrics endpoint              PromQL запросы
  (HTTP GET каждые N секунд)          │
       │                           Grafana
  ┌────▼────────────┐
  │  Ваш сервис     │
  │  GET /metrics   │
  └─────────────────┘
```

## Типы метрик

### Counter (счётчик)

Монотонно возрастающее число. Только растёт, при рестарте сбрасывается в 0.

```go
import "github.com/prometheus/client_golang/prometheus"

httpRequests := prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "Общее количество HTTP запросов",
    },
    []string{"method", "path", "status"},
)

// Использование
httpRequests.WithLabelValues("GET", "/api/users", "200").Inc()
httpRequests.WithLabelValues("POST", "/api/users", "500").Add(5)
```

**Использование:** количество запросов, ошибок, байт обработанных.
**PromQL:** `rate(http_requests_total[5m])` — скорость изменения.

### Gauge (датчик)

Значение которое может расти и убывать.

```go
activeConnections := prometheus.NewGauge(
    prometheus.GaugeOpts{
        Name: "active_connections",
        Help: "Количество активных соединений",
    },
)

activeConnections.Set(42)
activeConnections.Inc()
activeConnections.Dec()
activeConnections.Add(-10)
```

**Использование:** использование памяти, CPU, количество горутин, queue depth.

### Histogram (гистограмма)

Распределение значений по bucket'ам. Измеряет latency, размеры запросов.

```go
requestDuration := prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name:    "http_request_duration_seconds",
        Help:    "Latency HTTP запросов",
        Buckets: []float64{.001, .005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5},
        // или стандартные: prometheus.DefBuckets
    },
    []string{"method", "path"},
)

// Измерение
start := time.Now()
// ... обработка запроса ...
requestDuration.WithLabelValues("GET", "/api/users").
    Observe(time.Since(start).Seconds())
```

**Создаёт три метрики:**
- `http_request_duration_seconds_bucket{le="0.1"}` — запросы ≤ 100ms
- `http_request_duration_seconds_sum` — сумма всех значений
- `http_request_duration_seconds_count` — общее количество

**PromQL:** `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` — p99 latency.

### Summary (сводка)

Вычисляет квантили на стороне клиента. Менее гибко для агрегации чем Histogram.

```go
requestDuration := prometheus.NewSummaryVec(
    prometheus.SummaryOpts{
        Name:       "rpc_duration_seconds",
        Help:       "Latency RPC",
        Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001},
    },
    []string{"method"},
)
```

**Histogram vs Summary:**
- Histogram: buckets заданы заранее → можно агрегировать по серверам → выбирай его.
- Summary: квантили точные, но нельзя агрегировать; считается на стороне клиента.

## Регистрация и /metrics endpoint

```go
import (
    "net/http"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

// promauto — автоматически регистрирует в DefaultRegisterer
var (
    requestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "myapp_http_requests_total",
        },
        []string{"method", "status"},
    )
    requestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "myapp_http_request_duration_seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method"},
    )
)

func main() {
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":9090", nil)
}
```

## Middleware для Go HTTP сервера

```go
func metricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // Перехватываем status code
        rw := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
        next.ServeHTTP(rw, r)
        
        duration := time.Since(start).Seconds()
        status := strconv.Itoa(rw.statusCode)
        
        requestsTotal.WithLabelValues(r.Method, status).Inc()
        requestDuration.WithLabelValues(r.Method).Observe(duration)
    })
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}
```

## Конфигурация Prometheus

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'myapp'
    static_configs:
      - targets: ['myapp:9090']
    metrics_path: '/metrics'
    scrape_interval: 10s

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - "alerts.yml"
```

## PromQL — язык запросов

```promql
# Мгновенные значения
http_requests_total                          -- все серии
http_requests_total{method="GET"}            -- с фильтром
http_requests_total{status!="200"}           -- отрицание
http_requests_total{path=~"/api/.*"}         -- regex

# Функции
rate(http_requests_total[5m])                -- скорость/сек за 5 минут
irate(http_requests_total[5m])               -- мгновенная скорость
increase(http_requests_total[1h])            -- прирост за час

# Агрегация
sum(rate(http_requests_total[5m])) by (method)
avg(http_request_duration_seconds_sum / http_request_duration_seconds_count)

# Latency (p99)
histogram_quantile(0.99, 
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, method)
)

# Алёрт: error rate > 5%
sum(rate(http_requests_total{status=~"5.."}[5m])) /
sum(rate(http_requests_total[5m])) > 0.05
```

## Alerting

```yaml
# alerts.yml
groups:
  - name: myapp
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) /
          sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Высокий error rate {{ $value | humanizePercentage }}"
          
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, 
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
          ) > 1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "p99 latency превышает 1с: {{ $value }}s"
```

## Стандартные Go метрики (runtime)

```go
import "github.com/prometheus/client_golang/prometheus/collectors"

// Метрики Go runtime (горутины, GC, память)
prometheus.MustRegister(collectors.NewGoCollector())
// Метрики процесса (CPU, файловые дескрипторы)
prometheus.MustRegister(collectors.NewProcessCollector(collectors.ProcessCollectorOpts{}))
```

Даёт:
- `go_goroutines` — количество горутин
- `go_gc_duration_seconds` — длительность GC
- `go_memstats_heap_alloc_bytes` — выделено на heap
- `process_cpu_seconds_total` — CPU время процесса

## Лучшие практики

1. **Именование**: `namespace_subsystem_name_unit`. Например: `myapp_http_request_duration_seconds`.
2. **Labels**: не используй labels с высокой кардинальностью (user_id, IP → взрыв временных рядов).
3. **Histogram buckets**: подбирай под реальные значения latency.
4. **RED метрики**: Rate, Errors, Duration — минимальный набор для сервиса.
5. **USE метрики**: Utilization, Saturation, Errors — для ресурсов (CPU, память, диск).
