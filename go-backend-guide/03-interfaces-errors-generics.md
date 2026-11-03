# 03. Интерфейсы, ошибки, дженерики

## Интерфейсы

Интерфейс — набор методов. Тип реализует интерфейс **неявно**: достаточно иметь нужные методы, никакого `implements`.

```go
type Shape interface {
    Area() float64
    Perimeter() float64
}

type Rect struct{ W, H float64 }
func (r Rect) Area() float64      { return r.W * r.H }
func (r Rect) Perimeter() float64 { return 2 * (r.W + r.H) }

var s Shape = Rect{2, 3} // ок
```

Ключевые идеи:
- **Маленькие интерфейсы** лучше больших: `io.Reader` (один метод `Read`), `io.Writer`, `fmt.Stringer`, `error`.
- «Принимай интерфейсы, возвращай структуры».
- Интерфейс объявляют **там, где он используется** (у потребителя), а не рядом с реализацией. Это основа для моков в тестах.
- `any` (= `interface{}`) — любое значение. Используй редко.
- Приведение типа: `v, ok := x.(string)`; type switch: `switch v := x.(type) { case int: ... }`.

### Ловушка: nil-интерфейс
```go
type MyErr struct{}
func (*MyErr) Error() string { return "boom" }

func do() error {
    var e *MyErr = nil
    return e
}

func main() {
    fmt.Println(do() == nil) // false!
}
```
Интерфейс внутри хранит пару **(тип, значение)**. Здесь тип `*MyErr` не nil, значит и интерфейс не nil.
Вывод: возвращай `nil` явно, а не типизированный nil-указатель. Это популярный вопрос на собесах.

## Ошибки

В Go нет исключений. Ошибка — обычное значение типа `error`, которое возвращается последним.

```go
var ErrNotFound = errors.New("not found")      // sentinel-ошибка

type ValidationError struct {                  // свой тип ошибки
    Field string
    Msg   string
}
func (e *ValidationError) Error() string { return e.Field + ": " + e.Msg }

func GetUser(id int) (*User, error) {
    u, err := repo.Find(id)
    if err != nil {
        return nil, fmt.Errorf("get user %d: %w", id, err) // %w оборачивает ошибку
    }
    return u, nil
}

// проверка:
if errors.Is(err, ErrNotFound) { ... }          // есть ли в цепочке конкретная ошибка
var ve *ValidationError
if errors.As(err, &ve) { fmt.Println(ve.Field) } // есть ли ошибка такого типа
```

Правила:
- Ошибку либо **обрабатываешь**, либо **возвращаешь с контекстом**. Логировать и одновременно возвращать нельзя (будут дубли в логах).
- `panic` только для невозможных ситуаций (баг программиста). `recover` используют в middleware, чтобы сервер не падал целиком.
- `errors.Join` объединяет несколько ошибок.

## Дженерики

```go
func Map[T, U any](s []T, f func(T) U) []U {
    res := make([]U, 0, len(s))
    for _, v := range s {
        res = append(res, f(v))
    }
    return res
}

type Number interface {
    ~int | ~int64 | ~float64
}

func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}

type Set[T comparable] struct{ m map[T]struct{} }
```
- `comparable` — типы, которые можно сравнивать через `==` (нужно для ключей map).
- `cmp.Ordered` — типы с `<`, `>`.
- Пакеты `slices` и `maps` в стандартной библиотеке уже на дженериках.
- Не злоупотребляй: дженерики нужны для контейнеров и утилит, а не для бизнес-логики.

## Ресурсы
- go.dev/blog/error-handling-and-go, go.dev/blog/go1.13-errors
- go.dev/doc/tutorial/generics
- *100 Go Mistakes*: главы про интерфейсы и ошибки

## Задачи

- [ ] ⭐ **Фигуры.** Интерфейс `Shape`, типы `Rect`, `Circle`, `Triangle`. Функция, которая принимает `[]Shape` и возвращает фигуру с наибольшей площадью. Реализуй `String()` у каждой фигуры, чтобы `fmt.Println` выводил красиво.
- [ ] ⭐ **Type switch.** Функция `Describe(v any) string`, которая для int, string, []int, map[string]int, error и nil возвращает разное описание.
- [ ] ⭐⭐ **io.Reader.** Напиши тип `UpperReader`, который оборачивает любой `io.Reader` и отдаёт текст в верхнем регистре. Проверь: `io.Copy(os.Stdout, UpperReader{strings.NewReader("hello")})`. Затем `CountingWriter`, который считает записанные байты.
- [ ] ⭐⭐ **Ошибки валидации.** Функция `ValidateUser(u User) error` проверяет email, возраст (18–120) и непустое имя. Возвращает **все** ошибки сразу через `errors.Join`. В `main` через `errors.As` достань каждую `ValidationError` и выведи поле.
- [ ] ⭐⭐ **Цепочка ошибок.** Три слоя функций: `repo → service → handler`. Repo возвращает `ErrNotFound`, service оборачивает её через `%w`, handler через `errors.Is` превращает её в «404». Выведи итоговый текст ошибки и посмотри, как выглядит цепочка.
- [ ] ⭐⭐ **Generic-утилиты.** `Map`, `Filter`, `Reduce`, `Uniq[T comparable]`, `GroupBy[T any, K comparable](s []T, key func(T) K) map[K][]T`. Покрой тестами (после файла 05).
- [ ] ⭐⭐ **Generic Set.** `Set[T comparable]` с методами `Add`, `Remove`, `Contains`, `Len`, `Union`, `Intersection`, `Items()`.
- [ ] ⭐⭐⭐ **LRU-кэш.** `LRU[K comparable, V any]` с ёмкостью N, методами `Get(k) (V, bool)` и `Put(k, v)` за O(1). Подсказка: map + двусвязный список (`container/list` или свой). Это классическая задача на собеседовании.
- [ ] ⭐⭐⭐ **Плагинная система.** Интерфейс `Notifier` с методом `Notify(ctx, msg) error`. Реализации `EmailNotifier`, `SMSNotifier` (просто печатают). `MultiNotifier`, который сам реализует `Notifier` и рассылает во все, собирая ошибки. Это паттерн «Composite».

## Вопросы для самопроверки
1. Почему интерфейс с nil-указателем внутри не равен nil?
2. Разница между `errors.Is` и `errors.As`?
3. Чем `%w` отличается от `%v` в `fmt.Errorf`?
4. Где объявлять интерфейс: у реализации или у потребителя? Почему?
5. Что такое `~int` в ограничении типа?
