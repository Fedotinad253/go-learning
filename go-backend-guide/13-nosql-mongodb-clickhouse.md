# 13. NoSQL: MongoDB и ClickHouse

Основная база для Go-бэкенда — PostgreSQL (файл 07). Но в вакансиях регулярно встречаются MongoDB (документная БД) и ClickHouse (аналитическая). Джуну достаточно понимать, **когда** они лучше Postgres, и один раз поработать с каждой из Go.

**Когда проходить:** после файлов 10–12 (Redis, Kafka, RabbitMQ). Kafka нужна для части про ClickHouse: данные в него обычно приходят оттуда.

## Какую базу когда выбирать

| База | Модель | Сильные стороны | Слабые стороны | Пример |
|---|---|---|---|---|
| PostgreSQL | реляционная | транзакции, JOIN, целостность, универсальность | горизонтальное масштабирование сложнее | почти всё: пользователи, заказы, платежи |
| MongoDB | документы (JSON/BSON) | гибкая схема, вложенные данные, шардирование из коробки | JOIN неудобны, легко получить бардак в схеме | каталоги товаров с разными атрибутами, CMS, логи событий |
| ClickHouse | колоночная (OLAP) | агрегаты по миллиардам строк за секунды, сжатие | плохо для точечных UPDATE/DELETE, нет обычных транзакций | аналитика, метрики, события, логи |
| Redis | ключ–значение в памяти | скорость | объём ограничен памятью | кэш, сессии, счётчики (файл 10) |
| Elasticsearch / OpenSearch | поисковый индекс | полнотекстовый поиск, фасеты | не основное хранилище | поиск по товарам и логам |

Правило для собеседования: **по умолчанию Postgres**, другая база — когда есть конкретная причина (схема сильно разная у разных записей, нужна аналитика по огромным объёмам, нужен полнотекстовый поиск).

---

## Часть 1. MongoDB

### Понятия
- **Документ** — JSON-подобный объект (хранится как BSON). **Коллекция** — набор документов (аналог таблицы). Схема не обязательна.
- `_id` — первичный ключ, по умолчанию `ObjectID` (12 байт, внутри время создания).
- **Встраивание или ссылки.** Данные, которые читаются вместе и принадлежат одному родителю, встраивают (комментарии внутри статьи, если их немного). Большие и растущие списки выносят в отдельную коллекцию со ссылкой. Лимит размера документа — 16 МБ.
- **Индексы:** одиночные, составные (порядок полей важен, как в Postgres), уникальные, **TTL** (документ удаляется через N секунд — удобно для сессий и временных данных), текстовые.
- **Aggregation pipeline** — цепочка стадий: `$match` (фильтр) → `$group` → `$sort` → `$project`, `$lookup` (аналог LEFT JOIN).
- **Транзакции** на несколько документов есть, но требуют replica set. Операции над одним документом атомарны всегда.
- **Write concern** (`w: "majority"`) и **read concern** — насколько надёжно подтверждается запись и какие данные видит чтение.
- Масштабирование: **replica set** (копии для отказоустойчивости), **шардирование** по shard key.

### Запуск
```bash
docker run -d --name mongo -p 27017:27017 mongo:8
docker exec -it mongo mongosh
```
```js
use app
db.comments.insertOne({ task_id: 1, author: "ivan", text: "Готово", created_at: new Date() })
db.comments.find({ task_id: 1 }).sort({ created_at: -1 })
db.comments.createIndex({ task_id: 1, created_at: -1 })
db.comments.find({ task_id: 1 }).explain("executionStats")   // использует ли индекс
db.comments.aggregate([
  { $group: { _id: "$author", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
])
```

### MongoDB из Go
Официальный драйвер v2: `go.mongodb.org/mongo-driver/v2`.

```go
import (
    "go.mongodb.org/mongo-driver/v2/bson"
    "go.mongodb.org/mongo-driver/v2/mongo"
    "go.mongodb.org/mongo-driver/v2/mongo/options"
)

type Comment struct {
    ID        bson.ObjectID `bson:"_id,omitempty"`
    TaskID    int64         `bson:"task_id"`
    Author    string        `bson:"author"`
    Text      string        `bson:"text"`
    CreatedAt time.Time     `bson:"created_at"`
}

client, err := mongo.Connect(options.Client().ApplyURI("mongodb://localhost:27017"))
if err != nil { return err }
defer client.Disconnect(context.Background())
coll := client.Database("app").Collection("comments")

// вставка
res, err := coll.InsertOne(ctx, Comment{TaskID: 1, Author: "ivan", Text: "Готово", CreatedAt: time.Now()})
id := res.InsertedID.(bson.ObjectID)

// поиск с сортировкой и лимитом
opts := options.Find().
    SetSort(bson.D{{Key: "created_at", Value: -1}}).
    SetLimit(20)
cur, err := coll.Find(ctx, bson.D{{Key: "task_id", Value: int64(1)}}, opts)
if err != nil { return err }
var comments []Comment
if err := cur.All(ctx, &comments); err != nil { return err }

// один документ
var c Comment
err = coll.FindOne(ctx, bson.D{{Key: "_id", Value: id}}).Decode(&c)
if errors.Is(err, mongo.ErrNoDocuments) {
    // 404
}

// индекс
_, err = coll.Indexes().CreateOne(ctx, mongo.IndexModel{
    Keys: bson.D{{Key: "task_id", Value: 1}, {Key: "created_at", Value: -1}},
})
```

`bson.D` — упорядоченный документ (нужен, когда важен порядок полей: сортировка, индексы), `bson.M` — map, когда порядок не важен.

---

## Часть 2. ClickHouse

### Понятия
- **Колоночное хранение:** каждая колонка лежит отдельно и хорошо сжимается. Запрос `SELECT count() … GROUP BY type` читает только нужные колонки, поэтому агрегаты по сотням миллионов строк выполняются за секунды.
- **OLTP против OLAP.** Postgres — OLTP: много мелких транзакций, точечные чтения и обновления. ClickHouse — OLAP: мало больших аналитических запросов.
- **Вставляй батчами.** Тысячи строк одним INSERT раз в секунду, а не по одной строке. Каждая вставка создаёт кусок данных (part), которые потом сливаются в фоне. Много мелких вставок перегружают сервер.
- **Движок MergeTree** и его варианты: `ReplacingMergeTree` (схлопывает дубли по ключу при слиянии — помогает с at-least-once из Kafka), `SummingMergeTree`, `AggregatingMergeTree`.
- **ORDER BY** в таблице — это ключ сортировки и основа разреженного первичного индекса. Ставь в начало поля, по которым чаще всего фильтруешь. **PARTITION BY** — обычно по месяцу или дню.
- `UPDATE` и `DELETE` — тяжёлые фоновые «мутации», их стараются избегать.
- **Materialized views** — считают агрегаты при вставке. Есть движок Kafka для чтения прямо из топика.

### Запуск
Порт 9000 оставим под S3 из файла 14, поэтому нативный протокол ClickHouse пробросим на 19000:
```bash
docker run -d --name clickhouse -p 8123:8123 -p 19000:9000 \
  -e CLICKHOUSE_PASSWORD=secret clickhouse/clickhouse-server
docker exec -it clickhouse clickhouse-client --password secret
```
```sql
CREATE TABLE task_events (
    event_id    UUID,
    type        LowCardinality(String),
    task_id     Int64,
    occurred_at DateTime64(3)
) ENGINE = MergeTree
PARTITION BY toYYYYMM(occurred_at)
ORDER BY (type, occurred_at);

-- 10 млн тестовых строк за пару секунд
INSERT INTO task_events
SELECT generateUUIDv4(),
       ['task.created','task.updated','task.deleted'][rand() % 3 + 1],
       rand() % 100000,
       now() - toIntervalSecond(rand() % 2592000)
FROM numbers(10000000);

SELECT type, count() FROM task_events
WHERE occurred_at >= now() - INTERVAL 1 DAY
GROUP BY type;
```

### ClickHouse из Go
Клиент: `github.com/ClickHouse/clickhouse-go/v2`.

```go
import "github.com/ClickHouse/clickhouse-go/v2"

conn, err := clickhouse.Open(&clickhouse.Options{
    Addr: []string{"localhost:19000"},
    Auth: clickhouse.Auth{Database: "default", Username: "default", Password: "secret"},
})
if err != nil { return err }
defer conn.Close()

// батчевая вставка
batch, err := conn.PrepareBatch(ctx, "INSERT INTO task_events")
if err != nil { return err }
for _, e := range events {
    if err := batch.Append(e.EventID, e.Type, e.TaskID, e.OccurredAt); err != nil {
        return err
    }
}
if err := batch.Send(); err != nil { return err }

// аналитический запрос
rows, err := conn.Query(ctx, `
    SELECT type, count() AS cnt
    FROM task_events
    WHERE occurred_at >= now() - INTERVAL 7 DAY
    GROUP BY type ORDER BY cnt DESC`)
if err != nil { return err }
defer rows.Close()
for rows.Next() {
    var typ string
    var cnt uint64
    if err := rows.Scan(&typ, &cnt); err != nil { return err }
    fmt.Println(typ, cnt)
}
return rows.Err()
```

`e.EventID` может быть `uuid.UUID` из `github.com/google/uuid`, драйвер его понимает.

## Ресурсы
- mongodb.com/docs/manual — разделы CRUD, Indexes, Aggregation, Data Modeling
- learn.mongodb.com — бесплатные курсы, есть по Go
- clickhouse.com/docs — начни с «Concepts» и «Best practices»
- Книга *Designing Data-Intensive Applications* (Martin Kleppmann), главы 2 и 3 — модели данных и устройство хранилищ (колоночное хранение там объяснено лучше всего)

## Задачи
- [ ] ⭐ **mongosh руками.** Создай коллекцию, вставь 20 документов с разной структурой, сделай фильтры, сортировку, индекс. Через `explain` сравни запрос с индексом и без.
- [ ] ⭐⭐ **Комментарии к задачам в MongoDB.** В Tasks API добавь `POST /tasks/{id}/comments` и `GET /tasks/{id}/comments?limit=&before=` (курсорная пагинация по `created_at`). Задачи остаются в Postgres, комментарии — в Mongo. Репозиторий за интерфейсом, интеграционные тесты через testcontainers.
- [ ] ⭐⭐ **Aggregation.** Эндпоинт `GET /stats/comments`: топ-10 авторов по числу комментариев за последние 30 дней, через aggregation pipeline.
- [ ] ⭐⭐ **TTL-индекс.** Храни в Mongo «черновики» задач, которые автоматически удаляются через 24 часа. Проверь на TTL в 60 секунд.
- [ ] ⭐⭐ **Postgres против ClickHouse.** Залей одинаковые 10 млн событий в Postgres и ClickHouse. Сравни время `GROUP BY type` за неделю, размер таблиц на диске и время точечного `UPDATE` одной строки. Запиши выводы в README.
- [ ] ⭐⭐⭐ **Аналитика из Kafka.** Сервис `analytics` читает `task-events` из Kafka (файл 11), копит события в буфере и раз в секунду или каждые 1000 событий пишет их батчем в ClickHouse. Коммит offset — только после успешной вставки. Дубли схлопывай через `ReplacingMergeTree` по `event_id`. Эндпоинт `GET /analytics/tasks?period=7d` отдаёт количество событий по типам и по дням.

## Вопросы для самопроверки
1. Когда MongoDB лучше Postgres? Когда хуже?
2. Встраивание или отдельная коллекция: как решаешь?
3. Почему колоночная база быстро считает агрегаты, но плохо обновляет отдельные строки?
4. Почему в ClickHouse нельзя вставлять по одной строке?
5. Что задаёт `ORDER BY` в таблице MergeTree?
6. Чем OLTP отличается от OLAP?
