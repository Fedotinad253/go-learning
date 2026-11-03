# 16. Observability: логи, метрики, трейсинг, профилирование

В продакшене сервис нельзя отладить через `fmt.Println`. Нужны три «столпа»: **логи** (что произошло), **метрики** (сколько и как быстро), **трейсы** (путь запроса через сервисы).

## Логи
- Структурированные (JSON), а не строки: `log/slog` из стандартной библиотеки, популярны также `zap` и `zerolog`.
- Уровни: debug, info, warn, error.
- В каждом логе запроса: `request_id` или `trace_id`, чтобы найти все логи одного запроса.
- Не логируй пароли, токены и персональные данные.
- Собирают в ELK / OpenSearch / Loki.

```go
log := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))
log.Info("task created", "task_id", t.ID, "user_id", uid)
reqLog := log.With("request_id", rid) // логгер с постоянными полями
```

## Метрики (Prometheus + Grafana)
- Prometheus сам **забирает (scrape)** метрики с эндпоинта `/metrics` сервиса.
- Типы: **Counter** (только растёт: число запросов), **Gauge** (растёт и падает: активные соединения), **Histogram** (распределение: время ответа → перцентили p95, p99), Summary.
- Методы выбора метрик: **RED** для сервисов (Rate, Errors, Duration), **USE** для ресурсов (Utilization, Saturation, Errors).
- Язык запросов PromQL: `rate(http_requests_total[5m])`, `histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))`.
- Алерты: Alertmanager.

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequests = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "Total HTTP requests",
    }, []string{"method", "route", "status"})

    httpDuration = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Name:    "http_request_duration_seconds",
        Help:    "HTTP request duration",
        Buckets: prometheus.DefBuckets,
    }, []string{"method", "route"})
)

mux.Handle("GET /metrics", promhttp.Handler())
```
> В label `route` пиши шаблон (`/tasks/{id}`), а не реальный путь (`/tasks/123`), иначе метрик станет бесконечно много (high cardinality).

## Трейсинг (OpenTelemetry)
- **Trace** — весь путь запроса, **span** — одна операция (HTTP-хендлер, SQL-запрос, вызов gRPC).
- Контекст трейса передаётся между сервисами в заголовках (W3C `traceparent`) и через `context.Context` внутри сервиса.
- SDK: `go.opentelemetry.io/otel`, готовые обёртки `otelhttp`, `otelgrpc`, для pgx есть `otelpgx`.
- Бэкенды для просмотра: **Jaeger**, Tempo, Zipkin.

## Профилирование (pprof)
```go
import _ "net/http/pprof" // регистрирует /debug/pprof/ на DefaultServeMux
go http.ListenAndServe("localhost:6060", nil)
```
```bash
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30   # CPU
go tool pprof http://localhost:6060/debug/pprof/heap                 # память
go tool pprof -http=:8081 cpu.out                                    # веб-интерфейс с flame graph
```
Также полезны: `/debug/pprof/goroutine` (поиск утечек горутин) и `go tool trace`.

## Health checks
- `/healthz` (liveness): процесс жив.
- `/readyz` (readiness): зависимости (БД, Redis) доступны.

## Ресурсы
- prometheus.io/docs/introduction/overview
- opentelemetry.io/docs/languages/go/getting-started
- go.dev/doc/diagnostics, go.dev/blog/pprof

## Задачи
- [ ] ⭐ **Структурные логи.** Переведи Tasks API на `slog` (JSON), добавь `request_id` во все логи запроса через context.
- [ ] ⭐⭐ **Метрики.** Middleware для `http_requests_total` и `http_request_duration_seconds`, плюс бизнес-метрика `tasks_created_total`. Эндпоинт `/metrics`.
- [ ] ⭐⭐ **Prometheus + Grafana.** Добавь их в docker-compose (`prom/prometheus`, `grafana/grafana`), настрой scrape. Нагрузи сервис (`brew install hey` → `hey -n 10000 -c 50 http://localhost:8080/tasks`) и собери дашборд: RPS, доля ошибок, p95 и p99 времени ответа.
- [ ] ⭐⭐ **Алерт.** Правило Prometheus: доля 5xx больше 5% в течение минуты. Вызови ошибки и посмотри, как алерт переходит в firing.
- [ ] ⭐⭐⭐ **Распределённый трейсинг.** OpenTelemetry в Tasks API и Users service, экспорт в Jaeger (`jaegertracing/all-in-one` в compose). В Jaeger должен быть виден один трейс: HTTP-запрос → gRPC-вызов → SQL-запрос.
- [ ] ⭐⭐ **pprof: CPU.** Напиши намеренно медленный хендлер (например, неэффективная сортировка или регулярка, компилируемая в цикле). Найди узкое место через CPU-профиль и flame graph, исправь, сравни.
- [ ] ⭐⭐⭐ **pprof: утечка.** Сделай хендлер, который запускает горутину, навсегда блокирующуюся на канале. Нагрузи сервис, найди утечку через `/debug/pprof/goroutine` и метрику `go_goroutines`, исправь.

## Вопросы для самопроверки
1. Какие типы метрик в Prometheus? Чем Histogram лучше среднего значения?
2. Что такое high cardinality и почему это плохо?
3. Как trace_id попадает из одного сервиса в другой?
4. Как найти утечку горутин в работающем сервисе?
