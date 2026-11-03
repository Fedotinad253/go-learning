# 08. Docker, CI/CD, Kubernetes

## Docker: понятия
- **Образ (image)** — шаблон из слоёв; **контейнер** — запущенный экземпляр образа.
- **Dockerfile** описывает сборку образа; каждая инструкция = слой (порядок важен для кэша).
- **Volume** хранит данные вне контейнера (база не теряет данные при перезапуске).
- **Network**: контейнеры в одной сети общаются по имени сервиса (`postgres:5432`, а не `localhost`!).
- **docker compose** поднимает несколько контейнеров одной командой.

Команды: `build`, `run`, `ps`, `logs -f`, `exec -it`, `stop`, `rm`, `images`, `system prune`, `compose up -d`, `compose down -v`.

> **Mac на Apple Silicon:** образы собираются под `linux/arm64`. Если нужен образ для сервера на x86, собирай так: `docker build --platform linux/amd64 .`

## Многоэтапный Dockerfile для Go
```dockerfile
# версию поставь как в своём go.mod
FROM golang:1.25-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download            # отдельный слой: кэшируется, пока не изменились зависимости
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /app ./cmd/api

FROM gcr.io/distroless/static-debian12   # или alpine:3, или scratch
COPY --from=build /app /app
EXPOSE 8080
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```
Итоговый образ весит 10–20 МБ. Добавь `.dockerignore` (`.git`, бинарники, `.env`).

## docker-compose.yml
```yaml
services:
  api:
    build: .
    ports: ["8080:8080"]
    environment:
      DATABASE_URL: postgres://postgres:secret@postgres:5432/app?sslmode=disable
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:17
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 3s
      retries: 10

volumes:
  pgdata:
```

## Makefile
```makefile
.PHONY: run test lint up down
run:  ; go run ./cmd/api
test: ; go test -race -cover ./...
lint: ; golangci-lint run
up:   ; docker compose up -d --build
down: ; docker compose down
```
(в Makefile команды обычно пишут с новой строки с отступом **табом**; запись через `;` тоже работает)

## CI: GitHub Actions
Файл `.github/workflows/ci.yml`:
```yaml
name: ci
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
      - run: go vet ./...
      - run: go test -race -cover ./...
      - run: go build ./...
```
(версии actions со временем обновляются: если увидишь предупреждение, подними номер)

## Kubernetes (уровень понимания для джуна)
- **Pod** — один или несколько контейнеров; **Deployment** — управляет репликами подов и обновлениями; **Service** — стабильный адрес и балансировка для подов; **Ingress** — вход снаружи по HTTP.
- **ConfigMap / Secret** — конфигурация и секреты.
- **Probes:** liveness (жив ли процесс, иначе перезапуск), readiness (готов ли принимать трафик).
- **Resources:** requests и limits по CPU и памяти.
- Локально на Mac: в OrbStack есть встроенный Kubernetes, или `brew install kind kubectl`.
- `kubectl get pods`, `describe`, `logs`, `apply -f`, `port-forward`.

## Ресурсы
- docs.docker.com/get-started
- Документация GitHub Actions, раздел «Building and testing Go»
- kubernetes.io/docs/tutorials/kubernetes-basics

## Задачи
- [ ] ⭐ **Контейнеры руками.** Запусти nginx (`-p 8081:80`), открой в браузере, посмотри логи, зайди внутрь через `exec`, останови и удали.
- [ ] ⭐⭐ **Dockerfile для Tasks API.** Многоэтапная сборка, итоговый образ меньше 30 МБ, запуск не от root. Проверь размер через `docker images`.
- [ ] ⭐⭐ **Compose.** API + Postgres + применение миграций (отдельный сервис-мигратор или миграции при старте). `docker compose up` с нуля должен поднимать рабочий сервис.
- [ ] ⭐⭐ **Makefile и линтер.** Добавь Makefile и `.golangci.yml`, исправь все замечания линтера.
- [ ] ⭐⭐ **CI.** GitHub Actions: линтер, тесты с `-race`, сборка Docker-образа. Добавь бейдж статуса в README.
- [ ] ⭐⭐⭐ **Kubernetes.** Разверни Tasks API в локальном кластере: Deployment (2 реплики), Service, ConfigMap, Secret для пароля БД, liveness и readiness probes (сделай отдельный `/ready`, который проверяет соединение с БД). Убей под и посмотри, как Kubernetes поднимает новый.
- [ ] ⭐⭐⭐ **Публикация образа.** В CI собирай образ и пушь его в GitHub Container Registry (ghcr.io) при пуше тега `v*`.

## Вопросы для самопроверки
1. Чем образ отличается от контейнера? Что такое слой?
2. Зачем многоэтапная сборка? Почему `CGO_ENABLED=0`?
3. Почему из контейнера API нельзя подключиться к БД по `localhost`?
4. Liveness и readiness: в чём разница?
5. Чем Deployment отличается от Pod?
