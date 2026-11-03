# 04. Конкурентность в Go

Главная «фишка» Go и **тема номер один на собеседованиях**. Не торопись, сделай все задачи.

## Горутины

```go
go doWork()   // запускает функцию конкурентно
```
- Горутина очень лёгкая: начальный стек несколько КБ, он растёт по мере нужды. Можно запускать сотни тысяч.
- Планировщик Go (модель **G-M-P**: Goroutine, Machine = поток ОС, Processor = логический процессор) распределяет горутины по потокам ОС. `GOMAXPROCS` — число P (по умолчанию = число ядер).
- `main` не ждёт горутины: когда `main` завершается, завершается вся программа.

## Каналы

```go
ch := make(chan int)      // небуферизованный: отправитель ждёт получателя
bch := make(chan int, 10) // буферизованный: блокирует, только когда буфер полон

ch <- 1        // отправить
v := <-ch      // получить
v, ok := <-ch  // ok == false, если канал закрыт и пуст
close(ch)      // закрывает ОТПРАВИТЕЛЬ, а не получатель
for v := range ch { } // читает до закрытия

func producer(out chan<- int) {} // только отправка
func consumer(in <-chan int)  {} // только чтение
```

**Таблица поведения каналов (выучи наизусть):**

| Операция | nil-канал | закрытый канал | открытый канал |
|---|---|---|---|
| чтение | блокируется навсегда | нулевое значение, `ok=false` | ждёт или читает |
| запись | блокируется навсегда | **panic** | ждёт или пишет |
| close | **panic** | **panic** | закрывает |

### select
```go
select {
case v := <-ch1:
    fmt.Println(v)
case ch2 <- x:
case <-time.After(time.Second):
    fmt.Println("timeout")
case <-ctx.Done():
    return ctx.Err()
default: // если есть default, select не блокируется
}
```

## Пакет sync
- `sync.WaitGroup` — дождаться группы горутин (`Add`, `Done`, `Wait`; в новых версиях Go есть ещё `wg.Go(func)`).
- `sync.Mutex` / `sync.RWMutex` — защита общих данных. Не копируй мьютекс (передавай указатель на структуру).
- `sync.Once` — выполнить ровно один раз.
- `sync/atomic` — атомарные счётчики (`atomic.Int64`).
- `golang.org/x/sync/errgroup` — группа горутин с ошибкой и отменой (очень часто в реальном коде).

## context
```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel() // ВСЕГДА вызывай cancel

req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
```
- Передаётся **первым аргументом**: `func Do(ctx context.Context, ...)`.
- Нужен для отмены, таймаутов и дедлайнов по всей цепочке вызовов (HTTP → сервис → БД).
- `context.WithValue` только для request-scoped данных (request id, user id), не для параметров функций.

## Паттерны

### Worker pool
```go
func process(ctx context.Context, items []string, workers int) []Result {
    jobs := make(chan string)
    results := make(chan Result)

    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for it := range jobs {
                results <- handle(ctx, it)
            }
        }()
    }

    go func() { // отправитель закрывает jobs
        defer close(jobs)
        for _, it := range items {
            select {
            case jobs <- it:
            case <-ctx.Done():
                return
            }
        }
    }()

    go func() { // когда все воркеры закончили, закрываем results
        wg.Wait()
        close(results)
    }()

    var out []Result
    for r := range results {
        out = append(out, r)
    }
    return out
}
```

Ещё паттерны: **pipeline** (цепочка стадий через каналы), **fan-out/fan-in** (раздать работу, собрать результаты в один канал), **семафор** (буферизованный канал как ограничитель), **rate limiting** (`time.Ticker` или `golang.org/x/time/rate`), **graceful shutdown**.

## Типичные проблемы
- **Data race** — одновременный доступ к данным, когда хотя бы одна операция пишет. Ищется так: `go run -race .`, `go test -race ./...`
- **Deadlock** — все горутины ждут друг друга (`fatal error: all goroutines are asleep`).
- **Утечка горутин** — горутина навсегда заблокирована на канале, который никто не читает. Лечится через context и правильное закрытие каналов.

## Ресурсы
- gobyexample.com: разделы Goroutines … Stateful Goroutines
- go.dev/blog/pipelines — паттерны пайплайнов
- Доклад Rob Pike «Concurrency is not Parallelism» и «Go Concurrency Patterns» (YouTube)
- Книга *Concurrency in Go* (Katherine Cox-Buday)

## Задачи

- [ ] ⭐ **Параллельная сумма.** Раздели слайс из 10 млн чисел на N частей, посчитай суммы частей в горутинах, собери результат через канал. Сравни время с последовательной версией.
- [ ] ⭐ **Найди гонку.** Запусти с `-race` и исправь тремя способами: Mutex, atomic, канал:
  ```go
  counter := 0
  var wg sync.WaitGroup
  for i := 0; i < 1000; i++ {
      wg.Add(1)
      go func() { defer wg.Done(); counter++ }()
  }
  wg.Wait()
  fmt.Println(counter) // почти никогда не 1000
  ```
- [ ] ⭐ **Предскажи поведение.** Что выведет каждая программа (или как упадёт)? Проверь запуском:
  1. `ch := make(chan int); ch <- 1; fmt.Println(<-ch)`
  2. `ch := make(chan int, 1); ch <- 1; fmt.Println(<-ch)`
  3. `ch := make(chan int, 2); ch <- 1; close(ch); fmt.Println(<-ch, <-ch)`
  4. `var ch chan int; close(ch)`
- [ ] ⭐⭐ **Pipeline.** Три стадии: `generate(n)` отдаёт числа 1..n, `square` возводит в квадрат, `filter` пропускает чётные. Каждая стадия — функция, которая принимает и возвращает канал. Добавь `ctx` для досрочной остановки.
- [ ] ⭐⭐ **Generic fan-in.** `func Merge[T any](ctx context.Context, chans ...<-chan T) <-chan T` — объединяет несколько каналов в один и закрывает его, когда все входные закрыты.
- [ ] ⭐⭐ **Проверка сайтов.** CLI: получает файл со списком URL и проверяет их параллельно (не больше N одновременно) с таймаутом 3 секунды на запрос и общим таймаутом 30 секунд. Выводит статус-код и время ответа, в конце сводку. Используй worker pool или `errgroup` с `SetLimit`.
- [ ] ⭐⭐ **Семафор.** Реализуй тип `Semaphore` на буферизованном канале с методами `Acquire(ctx) error` и `Release()`. Запусти 20 горутин, из которых одновременно работают не больше 3 (выводи количество активных).
- [ ] ⭐⭐ **Rate limiter.** Функция обрабатывает 50 «запросов» со скоростью не больше 5 в секунду. Сделай двумя способами: `time.Ticker` и `golang.org/x/time/rate`.
- [ ] ⭐⭐⭐ **TTL-кэш.** Потокобезопасный `Cache[K, V]` с `Set(k, v, ttl)`, `Get(k)`, `Delete(k)`. Фоновая горутина раз в N секунд удаляет протухшие записи. Метод `Close()` корректно останавливает горутину (без утечки). Покрой тестами с `-race`.
- [ ] ⭐⭐⭐ **Graceful shutdown.** Программа запускает 5 воркеров, которые бесконечно обрабатывают задачи. По Ctrl+C (`signal.NotifyContext`) воркеры дорабатывают текущую задачу и завершаются, программа печатает «stopped gracefully». Если за 5 секунд не успели, выход принудительный.
- [ ] ⭐⭐⭐ **Pub/Sub в памяти.** Брокер с `Subscribe(topic) (<-chan Msg, unsubscribe func())` и `Publish(topic, msg)`. Медленный подписчик не должен блокировать остальных (буфер + отбрасывание или таймаут). Без гонок и утечек.

## Вопросы для самопроверки
1. Чем отличаются буферизованный и небуферизованный каналы?
2. Что будет при чтении из закрытого канала? При записи? При закрытии nil-канала?
3. Кто должен закрывать канал и почему?
4. Что такое G, M, P? Что происходит, когда горутина делает блокирующий системный вызов?
5. Mutex или канал: когда что?
6. Как найти утечку горутин? (подсказка: `runtime.NumGoroutine`, pprof, `go.uber.org/goleak` в тестах)
7. Зачем `defer cancel()` после `context.WithTimeout`?
