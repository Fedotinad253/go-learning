# 06. HTTP и REST API

## Теория HTTP (спрашивают на собесах)
- **Методы:** GET (чтение), POST (создание), PUT (полная замена), PATCH (частичное изменение), DELETE.
- **Идемпотентность:** GET, PUT и DELETE идемпотентны (повтор даёт тот же результат), POST нет.
- **Статус-коды:** 200 OK, 201 Created, 204 No Content, 400 Bad Request, 401 Unauthorized (не аутентифицирован), 403 Forbidden (нет прав), 404, 409 Conflict, 422, 429 Too Many Requests, 500, 502, 503, 504.
- **Заголовки:** Content-Type, Authorization, Accept, Cache-Control, X-Request-ID.
- **REST:** ресурсы во множественном числе (`/tasks`, `/tasks/42`), глаголы через методы, а не в URL; пагинация `?limit=20&offset=40` или `?cursor=...`; фильтры `?status=done`.
- Stateless, HTTPS/TLS, CORS, cookie и токены.

## Сервер на стандартной библиотеке

Начиная с Go 1.22 стандартный `ServeMux` поддерживает методы и параметры в путях, так что для учёбы роутер не нужен.

```go
package main

import (
    "context"
    "encoding/json"
    "errors"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

type Task struct {
    ID    int64  `json:"id"`
    Title string `json:"title"`
    Done  bool   `json:"done"`
}

func writeJSON(w http.ResponseWriter, status int, v any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    _ = json.NewEncoder(w).Encode(v)
}

func main() {
    log := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    mux := http.NewServeMux()
    mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
        writeJSON(w, http.StatusOK, map[string]string{"status": "ok"})
    })
    mux.HandleFunc("GET /tasks/{id}", func(w http.ResponseWriter, r *http.Request) {
        id := r.PathValue("id")
        writeJSON(w, http.StatusOK, map[string]string{"id": id})
    })

    srv := &http.Server{
        Addr:              ":8080",
        Handler:           loggingMiddleware(log, mux),
        ReadHeaderTimeout: 5 * time.Second,
        ReadTimeout:       10 * time.Second,
        WriteTimeout:      10 * time.Second,
        IdleTimeout:       60 * time.Second,
    }

    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()

    go func() {
        log.Info("server started", "addr", srv.Addr)
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            log.Error("listen", "err", err)
            os.Exit(1)
        }
    }()

    <-ctx.Done()
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    if err := srv.Shutdown(shutdownCtx); err != nil {
        log.Error("shutdown", "err", err)
    }
    log.Info("server stopped")
}

func loggingMiddleware(log *slog.Logger, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Info("request", "method", r.Method, "path", r.URL.Path, "dur", time.Since(start))
    })
}
```

Проверка: `go run .`, затем `curl -s localhost:8080/tasks/5 | jq`.

## Что ещё изучить
- **Middleware:** логирование, recover от panic, request-id, CORS, аутентификация, rate limit. Это функция `func(http.Handler) http.Handler`.
- **Валидация входных данных:** `github.com/go-playground/validator`.
- **Конфигурация:** из переменных окружения (`os.Getenv`, библиотеки `caarlos0/env`, `ilyakaznacheev/cleanenv`).
- **HTTP-клиент:** никогда не используй `http.Get` без таймаута. Создавай `&http.Client{Timeout: 5 * time.Second}`, передавай `ctx`, закрывай `resp.Body`.
- **Аутентификация:** хеширование паролей `golang.org/x/crypto/bcrypt`, JWT (`github.com/golang-jwt/jwt/v5`), access и refresh токены.
- **Роутеры**, которые встречаются в вакансиях: **chi** (близок к стандартной библиотеке), **gin**, **echo**, **fiber**. Достаточно знать стандартный mux и один из них.
- **Документация API:** OpenAPI (Swagger), генерация через `swaggo/swag` или кода из спеки через `oapi-codegen`.

## Задачи (начинаем сквозной проект **Tasks API**)

- [ ] ⭐ **Hello API.** Сервер с `GET /health` и `GET /hello?name=Ivan` → `{"message":"Hello, Ivan"}`. Если name не передан, вернуть 400 с JSON-ошибкой.
- [ ] ⭐⭐ **Tasks API (in-memory).** CRUD для задач:
  - `POST /tasks` (title обязателен, 1–200 символов) → 201 + задача
  - `GET /tasks` с фильтром `?done=true` и пагинацией `?limit=&offset=`
  - `GET /tasks/{id}`, `PATCH /tasks/{id}`, `DELETE /tasks/{id}` → 204
  - Хранилище: map + `sync.RWMutex`, вынесенное за интерфейс `TaskRepository`
  - Ошибки в едином формате `{"error": "..."}` и с правильными статус-кодами
  - Тесты хендлеров через `httptest`
- [ ] ⭐⭐ **Middleware.** Добавь в Tasks API: логирование (метод, путь, статус, длительность; для статуса нужна обёртка над `ResponseWriter`), recover от panic → 500, `X-Request-ID` (взять из заголовка или сгенерировать, положить в context и в логи).
- [ ] ⭐⭐ **Конфиг и graceful shutdown.** Порт, таймауты и уровень логов из env. Корректная остановка по SIGTERM.
- [ ] ⭐⭐ **HTTP-клиент с ретраями.** Функция, которая ходит во внешнее API (например, api.github.com/users/{name}) с таймаутом и до 3 повторами при 5xx и сетевых ошибках, с экспоненциальной задержкой и jitter. Тесты через `httptest.NewServer`.
- [ ] ⭐⭐⭐ **Аутентификация.** `POST /auth/register`, `POST /auth/login` (bcrypt + JWT), middleware проверки токена. Задачи привязываются к пользователю: каждый видит только свои. Чужая задача → 404.
- [ ] ⭐⭐⭐ **Rate limit middleware.** Не больше 10 запросов в секунду на IP (token bucket, `x/time/rate`, map лимитеров с очисткой старых). Превышение → 429 и заголовок `Retry-After`.
- [ ] ⭐⭐⭐ **URL shortener.** Отдельный мини-проект: `POST /shorten {url}` → короткий код, `GET /{code}` → редирект 302, счётчик переходов, `GET /stats/{code}`. Позже переведёшь его на Postgres и Redis.
- [ ] ⭐⭐ **Переписать на chi.** Перенеси Tasks API на `go-chi/chi`. Сравни, что стало удобнее.

## Вопросы для самопроверки
1. Разница между PUT и PATCH? Что такое идемпотентность?
2. 401 или 403: когда какой?
3. Зачем нужны таймауты у `http.Server` и `http.Client`?
4. Почему обязательно закрывать `resp.Body`?
5. Как работает graceful shutdown?
6. Где хранить JWT на клиенте и чем опасно каждое место?
