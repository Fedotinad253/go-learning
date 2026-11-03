# 12. RabbitMQ

В файле 11 RabbitMQ был только в сравнении с Kafka. Но во многих компаниях (особенно не самых крупных) основной брокер именно он, поэтому стоит один раз сделать с ним pet-проект.

**Когда проходить:** после файла 11.

## Модель: exchange → binding → queue

```
producer ──► exchange ──(binding по routing key)──► queue ──► consumer
```

- **Producer** публикует сообщение не в очередь, а в **exchange** с **routing key**.
- **Exchange** по правилам (**bindings**) решает, в какие очереди положить копию.
- **Queue** хранит сообщения, пока их не заберёт и не подтвердит consumer.
- **Consumer** получает сообщения, обрабатывает и отправляет **ack**. После ack сообщение удаляется.

### Типы exchange
| Тип | Как маршрутизирует | Пример |
|---|---|---|
| direct | routing key совпадает с ключом binding | `task.created` → очередь `notifier` |
| topic | шаблоны: `*` — ровно одно слово, `#` — ноль и больше | `task.*` ловит `task.created` и `task.deleted`; `#.error` ловит любые ошибки |
| fanout | во все привязанные очереди, ключ игнорируется | рассылка событий нескольким сервисам |
| headers | по заголовкам сообщения | редко |

Есть **default exchange** (пустое имя `""`): он кладёт сообщение в очередь, имя которой равно routing key.

## Что нужно знать

- **Ack / Nack / Reject.** `Ack` — обработано, удалить. `Nack(requeue=true)` — вернуть в очередь. `Nack(requeue=false)` — выбросить или отправить в DLX (см. ниже).
- **autoAck=true** — сообщение считается доставленным сразу при отправке потребителю. Если сервис упал во время обработки, сообщение потеряно. В продакшене почти всегда ручной ack.
- **Prefetch (QoS).** `ch.Qos(10, 0, false)` — брокер отдаёт потребителю не больше 10 неподтверждённых сообщений. Без этого один потребитель может забрать всю очередь в память.
- **Надёжность:** durable-очередь + persistent-сообщения (`DeliveryMode: Persistent`) + **publisher confirms** (брокер подтверждает, что принял сообщение) + ручной ack у потребителя.
- **Типы очередей:**
  - *classic* — обычная очередь на одном узле;
  - *quorum* — реплицируется между узлами (Raft), рекомендуемый вариант для надёжных очередей. Classic mirrored queues в RabbitMQ 4 удалены;
  - *stream* — лог на добавление с перечитыванием, похож на Kafka.
- **DLX (dead letter exchange).** Очередь объявляют с аргументом `x-dead-letter-exchange`. Сообщения, получившие `Nack(requeue=false)`, истёкшие по TTL или вытесненные из-за переполнения, уходят туда.
- **Отложенный retry:** отдельная очередь с `x-message-ttl` и DLX, указывающим обратно на основной exchange. Сообщение «полежит» N секунд и вернётся на обработку.
- **Семантика:** как и в Kafka, по умолчанию at-least-once, поэтому обработчик должен быть идемпотентным (файл 11).
- **Порядок** гарантируется в пределах одной очереди с одним потребителем. Несколько потребителей на одной очереди обрабатывают сообщения параллельно, и порядок теряется.

### Kafka или RabbitMQ
| | RabbitMQ | Kafka |
|---|---|---|
| Модель | очередь: сообщение удаляется после ack | лог: сообщения хранятся по retention |
| Маршрутизация | гибкая (exchange, шаблоны) | по топику и ключу партиции |
| Перечитать старые сообщения | нет (кроме streams) | да, сдвигом offset |
| Масштабирование потребителей | сколько угодно на одной очереди | не больше числа партиций в группе |
| Типичное применение | очереди задач, RPC, маршрутизация команд | поток событий, аналитика, event sourcing, высокая нагрузка |

## Запуск

```bash
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:4-management
```
Веб-интерфейс: http://localhost:15672, логин и пароль `guest` / `guest`. Там видно очереди, сообщения, потребителей, можно публиковать сообщения руками.

## RabbitMQ из Go

Официальный клиент: `github.com/rabbitmq/amqp091-go`.

```go
import amqp "github.com/rabbitmq/amqp091-go"

conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
if err != nil { return err }
defer conn.Close()
ch, err := conn.Channel()
if err != nil { return err }
defer ch.Close()

// топология: exchange, DLX, очередь с привязкой
_ = ch.ExchangeDeclare("tasks", "topic", true, false, false, false, nil)
_ = ch.ExchangeDeclare("tasks.dlx", "fanout", true, false, false, false, nil)
q, _ := ch.QueueDeclare("notifier", true, false, false, false, amqp.Table{
    "x-queue-type":           "quorum",
    "x-dead-letter-exchange": "tasks.dlx",
})
_ = ch.QueueBind(q.Name, "task.*", "tasks", false, nil)
// (обработка ошибок опущена для краткости — в своём коде проверяй каждую)
```

**Producer с publisher confirms:**
```go
if err := ch.Confirm(false); err != nil { return err }
confirms := ch.NotifyPublish(make(chan amqp.Confirmation, 1))

err = ch.PublishWithContext(ctx, "tasks", "task.created", false, false, amqp.Publishing{
    ContentType:  "application/json",
    DeliveryMode: amqp.Persistent,
    MessageId:    eventID, // пригодится для идемпотентности
    Body:         payload,
})
if err != nil { return err }

select {
case c := <-confirms:
    if !c.Ack {
        return errors.New("broker nacked message")
    }
case <-ctx.Done():
    return ctx.Err()
}
```

**Consumer с ручным ack и prefetch:**
```go
if err := ch.Qos(10, 0, false); err != nil { return err }
msgs, err := ch.Consume("notifier", "notifier-1", false, false, false, false, nil)
if err != nil { return err }

for {
    select {
    case <-ctx.Done():
        return nil
    case d, ok := <-msgs:
        if !ok {
            return errors.New("channel closed") // нужно переподключиться
        }
        if err := handle(ctx, d.Body); err != nil {
            log.Error("handle", "err", err, "id", d.MessageId)
            _ = d.Nack(false, false) // уйдёт в DLX
            continue
        }
        _ = d.Ack(false)
    }
}
```

**Важно:** `amqp091-go` **не переподключается сам**. Подпишись на `conn.NotifyClose(...)` и при разрыве заново открывай соединение, канал и `Consume`. Один `Channel` не используй из нескольких горутин для публикации одновременно: делай канал на горутину или защищай его мьютексом.

## Ресурсы
- rabbitmq.com/tutorials — 7 коротких туториалов, есть версии на Go (обязательно пройди 1–5)
- rabbitmq.com/docs: разделы Reliability Guide, Quorum Queues, Dead Letter Exchanges, Consumer Prefetch
- pkg.go.dev/github.com/rabbitmq/amqp091-go

## Задачи
- [ ] ⭐ **Туториалы.** Пройди официальные туториалы 1–5 на Go. В веб-интерфейсе наблюдай, как меняются очереди.
- [ ] ⭐ **Worker queue.** Producer отправляет 100 «задач» со случайным временем обработки. Запусти 3 потребителя с `prefetch=1` и с `prefetch=50`, сравни, как распределяется нагрузка.
- [ ] ⭐⭐ **Topic exchange.** Tasks API публикует `task.created`, `task.updated`, `task.deleted` в exchange `tasks`. Сервис `notifier` слушает `task.*`, сервис `audit` — `#`. Оба в docker-compose.
- [ ] ⭐⭐ **Надёжная доставка.** Включи publisher confirms и ручной ack. Убей потребителя посреди обработки (`kill -9`) и проверь, что сообщение не потерялось. Затем перезапусти контейнер RabbitMQ и проверь, что durable-сообщения на месте.
- [ ] ⭐⭐ **Retry и DLX.** Потребитель «падает» в 30% случаев. Сделай 3 попытки с задержкой 5, 15, 60 секунд через очереди с TTL, после чего сообщение уходит в `tasks.dead`. Счётчик попыток храни в заголовке сообщения.
- [ ] ⭐⭐ **Переподключение.** Напиши обёртку над соединением, которая при разрыве переподключается с экспоненциальной задержкой и заново объявляет топологию. Проверь: `docker restart rabbitmq` не роняет сервис.
- [ ] ⭐⭐⭐ **Outbox на RabbitMQ.** Переделай transactional outbox из файла 11 так, чтобы воркер отправлял события в RabbitMQ и помечал запись отправленной только после publisher confirm. Сделай отправку через интерфейс `EventPublisher` с двумя реализациями: Kafka и RabbitMQ.

## Вопросы для самопроверки
1. Чем exchange отличается от очереди? Какие бывают типы exchange?
2. Что будет с сообщением при `autoAck=true`, если потребитель упал во время обработки?
3. Зачем нужен prefetch?
4. Что такое publisher confirms? Чем они отличаются от ack потребителя?
5. Как сделать отложенный retry в RabbitMQ?
6. Когда выберешь RabbitMQ, а когда Kafka?
