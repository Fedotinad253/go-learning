# 07. SQL и PostgreSQL

SQL есть почти в каждой вакансии, и на собесах часто дают задачи на SQL. Уделяй этой теме не меньше времени, чем Go.

## Запуск Postgres на Mac через Docker
```bash
docker run --name pg -e POSTGRES_PASSWORD=secret -e POSTGRES_DB=app \
  -p 5432:5432 -v pgdata:/var/lib/postgresql/data -d postgres:17

docker exec -it pg psql -U postgres -d app   # консоль psql
```
Полезные команды psql: `\dt` (таблицы), `\d tasks` (структура), `\x` (вертикальный вывод), `\q` (выход).
GUI-клиенты: DBeaver (`brew install --cask dbeaver-community`), TablePlus, DataGrip.

## Что нужно знать

### SQL
- DDL: `CREATE TABLE`, типы (`bigint`, `text`, `varchar`, `boolean`, `timestamptz`, `numeric`, `jsonb`, `uuid`), `PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `NOT NULL`, `CHECK`, `DEFAULT`.
- `SELECT … WHERE … ORDER BY … LIMIT/OFFSET`, `INSERT … RETURNING`, `UPDATE`, `DELETE`, `ON CONFLICT DO UPDATE` (upsert).
- `JOIN`: INNER, LEFT, RIGHT, FULL, CROSS, self-join.
- Агрегаты: `COUNT`, `SUM`, `AVG`, `GROUP BY`, `HAVING`.
- Подзапросы, `EXISTS`, `IN`, CTE (`WITH`), оконные функции (`ROW_NUMBER`, `RANK`, `LAG`, `SUM() OVER (PARTITION BY …)`).
- NULL-логика: `NULL = NULL` даёт не true, а NULL; используй `IS NULL`, `COALESCE`.

### Индексы
- B-tree (по умолчанию), составной индекс (порядок колонок важен!), уникальный, частичный, GIN (для jsonb и полнотекстового поиска).
- `EXPLAIN ANALYZE` — смотреть план запроса: Seq Scan, Index Scan, Index Only Scan.
- Индексы ускоряют чтение, но замедляют запись и занимают место.

### Транзакции (обязательно на собесе)
- **ACID:** атомарность, согласованность, изолированность, долговечность.
- **Аномалии:** dirty read, non-repeatable read, phantom read, lost update, write skew.
- **Уровни изоляции:** Read Uncommitted, Read Committed (по умолчанию в Postgres), Repeatable Read, Serializable. Какие аномалии на каком уровне возможны.
- **MVCC** в Postgres: читатели не блокируют писателей.
- Блокировки: `SELECT … FOR UPDATE`, взаимоблокировки (deadlock) и как их избегать (единый порядок блокировки).

### Проектирование
- Нормальные формы (1НФ–3НФ), связи 1:1, 1:N, M:N (через связующую таблицу).
- Проблема N+1 запросов.
- Пагинация: OFFSET (медленная на больших смещениях) против keyset/cursor (`WHERE id > $1 ORDER BY id LIMIT 20`).
- Репликация и шардирование на уровне понимания.

## Postgres из Go

Самый популярный драйвер — **pgx** (`github.com/jackc/pgx/v5`).

```go
import (
    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"
)

pool, err := pgxpool.New(ctx, "postgres://postgres:secret@localhost:5432/app")
if err != nil { return err }
defer pool.Close()

var t Task
err = pool.QueryRow(ctx,
    `SELECT id, title, done FROM tasks WHERE id = $1`, id,
).Scan(&t.ID, &t.Title, &t.Done)
if errors.Is(err, pgx.ErrNoRows) {
    return Task{}, ErrNotFound
}

// транзакция
tx, err := pool.Begin(ctx)
if err != nil { return err }
defer tx.Rollback(ctx) // после Commit ничего не делает
if _, err := tx.Exec(ctx, `UPDATE accounts SET balance = balance - $1 WHERE id = $2`, amount, from); err != nil {
    return err
}
// ...
return tx.Commit(ctx)
```

**Никогда** не собирай SQL конкатенацией строк с пользовательскими данными (SQL-инъекция). Только плейсхолдеры `$1, $2`.

Инструменты, которые встречаются в вакансиях:
- `database/sql` + `sqlx` — классика
- **sqlc** — генерирует типобезопасный Go-код из SQL-запросов (очень популярен)
- **squirrel** — построитель запросов
- **GORM** — ORM (есть во многих компаниях, но многие предпочитают чистый SQL)
- **Миграции:** `pressly/goose` или `golang-migrate/migrate`

## Ресурсы
- **sql-ex.ru**, **sqlzoo.net**, **pgexercises.com** — тренажёры SQL
- **use-the-index-luke.com** — про индексы (есть на русском)
- Официальная документация Postgres: главы про транзакции и изоляцию
- Курс Postgres Pro «DBA1» / «DEV1» (бесплатные видео и материалы)

## Задачи

- [ ] ⭐ **Схема магазина.** Создай таблицы `users`, `products`, `orders`, `order_items` с PK, FK, NOT NULL, CHECK (цена > 0). Заполни тестовыми данными (`generate_series` поможет сделать много строк).
- [ ] ⭐⭐ **15 запросов** к этой схеме:
  1. Пользователи, зарегистрированные за последние 30 дней
  2. Топ-5 товаров по выручке
  3. Пользователи без единого заказа (LEFT JOIN и NOT EXISTS: оба способа)
  4. Средний чек по месяцам
  5. Пользователи, потратившие больше 10 000
  6. Для каждого пользователя его последний заказ (оконная функция `ROW_NUMBER`)
  7. Накопительная сумма выручки по дням (`SUM() OVER (ORDER BY …)`)
  8. Товары, которые ни разу не покупали
  9. Процент заказов каждого статуса
  10. Дубликаты email (с учётом регистра)
  11–15. Придумай сам и реши
- [ ] ⭐⭐ **Индексы.** Сгенерируй 1 млн заказов. Найди медленный запрос через `EXPLAIN ANALYZE`, добавь индекс, сравни планы и время. Проверь, когда составной индекс `(user_id, created_at)` используется, а когда нет.
- [ ] ⭐⭐ **Аномалии изоляции.** Открой две сессии psql и воспроизведи руками non-repeatable read на Read Committed и lost update. Затем покажи, что Repeatable Read их предотвращает (одна из транзакций получит ошибку сериализации).
- [ ] ⭐⭐ **Tasks API на Postgres.** Замени in-memory репозиторий на Postgres (pgxpool). Миграции через goose. Обработка `ErrNoRows` → 404. Строка подключения из env.
- [ ] ⭐⭐ **Пагинация.** Сделай в `GET /tasks` keyset-пагинацию (`?cursor=`). Сравни скорость с OFFSET на миллионе строк.
- [ ] ⭐⭐⭐ **Перевод денег.** Сервис счетов: `POST /transfers {from, to, amount}`. Транзакция с `SELECT … FOR UPDATE`, проверка баланса, запись в таблицу операций. Напиши тест, который делает 100 параллельных переводов между двумя счетами в обе стороны: не должно быть ни deadlock, ни отрицательного баланса (подсказка: блокируй счета в порядке возрастания id).
- [ ] ⭐⭐⭐ **Интеграционные тесты.** Тесты репозитория на настоящем Postgres через testcontainers-go: контейнер стартует в тесте, применяются миграции, после теста всё удаляется.
- [ ] ⭐⭐ **sqlc.** Переведи один репозиторий на sqlc. Сравни с ручным кодом.

## Вопросы для самопроверки
1. Какие уровни изоляции есть и какие аномалии на каждом возможны? Какой уровень по умолчанию в Postgres?
2. Как работает B-tree индекс? Почему индекс по `(a, b)` не помогает для `WHERE b = ?`
3. Что такое MVCC?
4. Чем LEFT JOIN отличается от INNER JOIN?
5. Как избежать deadlock?
6. Почему OFFSET-пагинация медленная на больших значениях?
7. Что такое N+1 и как его исправить?
