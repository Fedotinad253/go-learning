# 05. Тестирование

В Go тесты встроены в язык. Код без тестов на собеседовании выглядит слабо, а с хорошими тестами сразу выделяет тебя.

## Основы
- Файл `xxx_test.go` рядом с кодом, функция `func TestXxx(t *testing.T)`.
- Запуск: `go test ./...`, подробно `-v`, конкретный тест `-run TestName`, с детектором гонок `-race`.
- `t.Errorf` (тест продолжается), `t.Fatalf` (тест останавливается), `t.Helper()`, `t.Cleanup()`, `t.Parallel()`.

## Табличные тесты (стандарт в Go)
```go
func TestDivide(t *testing.T) {
    tests := []struct {
        name    string
        a, b    float64
        want    float64
        wantErr bool
    }{
        {"normal", 10, 2, 5, false},
        {"negative", -9, 3, -3, false},
        {"by zero", 1, 0, 0, true},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Divide(tt.a, tt.b)
            if (err != nil) != tt.wantErr {
                t.Fatalf("err = %v, wantErr %v", err, tt.wantErr)
            }
            if got != tt.want {
                t.Errorf("got %v, want %v", got, tt.want)
            }
        })
    }
}
```

## Библиотеки
- **testify** (`github.com/stretchr/testify`): `assert.Equal`, `require.NoError`. Используется в большинстве компаний.
- **Моки:** вручную через интерфейсы, или генерация `go.uber.org/mock` (mockgen), `mockery`.
- **httptest:** `httptest.NewRecorder()` для тестов хендлеров, `httptest.NewServer()` для фейкового внешнего API.
- **testcontainers-go:** поднимает настоящий Postgres/Redis/Kafka в Docker для интеграционных тестов.

## Мок через интерфейс
```go
type UserRepo interface {
    GetByID(ctx context.Context, id int64) (User, error)
}

type fakeRepo struct{ users map[int64]User }

func (f *fakeRepo) GetByID(_ context.Context, id int64) (User, error) {
    u, ok := f.users[id]
    if !ok {
        return User{}, ErrNotFound
    }
    return u, nil
}

// в тесте: svc := NewUserService(&fakeRepo{users: ...})
```

## Бенчмарки, фаззинг, покрытие
```go
func BenchmarkConcat(b *testing.B) {
    for b.Loop() { // в старых версиях Go: for i := 0; i < b.N; i++
        _ = concat(words)
    }
}

func FuzzReverse(f *testing.F) {
    f.Add("hello")
    f.Fuzz(func(t *testing.T, s string) {
        if Reverse(Reverse(s)) != s {
            t.Errorf("double reverse mismatch for %q", s)
        }
    })
}
```
```bash
go test -bench=. -benchmem
go test -fuzz=FuzzReverse -fuzztime=30s
go test -cover -coverprofile=c.out ./... && go tool cover -html=c.out
```

## Виды тестов
- **Unit** — одна функция или слой, зависимости замоканы. Быстрые, их больше всего.
- **Интеграционные** — с настоящей БД и брокером (testcontainers). Часто помечают build tag `//go:build integration`.
- **E2E** — поднять весь сервис и дёргать его API.

## Ресурсы
- go.dev/doc/tutorial/add-a-test, go.dev/doc/tutorial/fuzz
- Книга *Learn Go with Tests* (Chris James), бесплатна на quii.gitbook.io. Очень рекомендую: учит Go через TDD.

## Задачи

- [ ] ⭐ **Покрой прошлые задачи.** Табличные тесты для `Reverse`, `IsPalindrome`, шифра Цезаря, `Stack`, `Account`. Цель: покрытие ≥ 80% (`go test -cover`).
- [ ] ⭐ **Сначала тест (TDD).** Напиши тесты для функции `ParseSize("10KB") → 10240` (B, KB, MB, GB, ошибки на мусор), а только потом реализацию.
- [ ] ⭐⭐ **Мок репозитория.** `UserService` с методом `Register(ctx, email, password)`: проверяет, что email не занят, хеширует пароль, сохраняет пользователя. Протестируй через ручной мок `UserRepo`: успех, email занят, ошибка БД. Потом сделай то же через mockgen или mockery.
- [ ] ⭐⭐ **Внешний API.** Функция `GetWeather(ctx, client, city)` ходит в HTTP API. Протестируй её через `httptest.NewServer`: успешный ответ, 500, невалидный JSON, таймаут (сервер спит дольше таймаута клиента).
- [ ] ⭐⭐ **Бенчмарк.** Сравни склейку 10 000 строк через `+`, `strings.Builder`, `strings.Join`, `bytes.Buffer`. Добавь `-benchmem` и объясни разницу в аллокациях.
- [ ] ⭐⭐ **Фаззинг.** Запусти фазз-тест для своего `Reverse` из файла 02. Если найдётся баг (например, с невалидным UTF-8), исправь его. Найденный кейс сохранится в `testdata/` — посмотри на него.
- [ ] ⭐⭐⭐ **Тест конкурентного кода.** Тесты для TTL-кэша из файла 04: параллельные Get/Set из 100 горутин с `-race`, проверка истечения TTL, проверка, что `Close()` не оставляет горутин (`go.uber.org/goleak`).
