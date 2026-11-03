# 10. Redis

In-memory хранилище «ключ–значение». Используется для кэша, сессий, rate limiting, очередей, распределённых блокировок, счётчиков, рейтингов.

## Запуск
```bash
docker run --name redis -p 6379:6379 -d redis:7
docker exec -it redis redis-cli
```

## Типы данных и команды
| Тип | Команды | Применение |
|---|---|---|
| String | `SET k v EX 60`, `GET`, `INCR`, `SETNX`, `MGET` | кэш, счётчики, блокировки |
| Hash | `HSET`, `HGET`, `HGETALL`, `HINCRBY` | объекты (профиль пользователя) |
| List | `LPUSH`, `RPOP`, `LRANGE`, `BLPOP` | простые очереди, ленты |
| Set | `SADD`, `SISMEMBER`, `SINTER` | уникальные значения, теги |
| Sorted Set | `ZADD`, `ZRANGE … WITHSCORES`, `ZREVRANK`, `ZREMRANGEBYSCORE` | рейтинги, sliding window |
| Stream | `XADD`, `XREADGROUP` | очереди с группами потребителей |

Плюс `EXPIRE` и `TTL` (время жизни ключей), `PUBLISH` и `SUBSCRIBE`, `SCAN` (вместо `KEYS *`, которая блокирует сервер).

## Что знать
- Redis однопоточный для выполнения команд, поэтому каждая команда атомарна. Для нескольких команд: `MULTI/EXEC`, пайплайны или Lua-скрипты.
- Персистентность: RDB (снапшоты) и AOF (журнал).
- Политики вытеснения при нехватке памяти (`maxmemory-policy`, например `allkeys-lru`).
- **Паттерны кэширования:** cache-aside (самый частый), write-through, write-behind.
- **Проблемы кэша:** инвалидация, cache stampede / thundering herd (много запросов одновременно идут в БД после истечения ключа; решается через `golang.org/x/sync/singleflight` или блокировку), устаревшие данные.
- Распределённая блокировка: `SET lock:x <uuid> NX PX 10000`, снимать только свою (проверка значения через Lua).

## Redis из Go
Клиент: `github.com/redis/go-redis/v9`.

```go
rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})

func (s *Service) GetTask(ctx context.Context, id int64) (Task, error) {
    key := fmt.Sprintf("task:%d", id)

    data, err := s.rdb.Get(ctx, key).Bytes()
    if err == nil {
        var t Task
        if json.Unmarshal(data, &t) == nil {
            return t, nil // cache hit
        }
    } else if !errors.Is(err, redis.Nil) {
        s.log.Warn("redis get", "err", err) // Redis упал: не ломаем запрос, идём в БД
    }

    t, err := s.repo.GetByID(ctx, id) // cache miss
    if err != nil {
        return Task{}, err
    }
    if b, err := json.Marshal(t); err == nil {
        s.rdb.Set(ctx, key, b, 5*time.Minute)
    }
    return t, nil
}
```

## Ресурсы
- redis.io/docs (раздел Data types)
- redis.io/try-free или локальный redis-cli для экспериментов

## Задачи
- [ ] ⭐ **Команды руками.** В redis-cli поработай со всеми типами из таблицы. Создай ключ с TTL 10 секунд и проверь, что он исчез.
- [ ] ⭐⭐ **Кэш для Tasks API.** Cache-aside для `GET /tasks/{id}`. При `PATCH` и `DELETE` инвалидируй ключ. Добавь Redis в docker-compose. Если Redis недоступен, сервис продолжает работать через БД.
- [ ] ⭐⭐ **Rate limiter на Redis.** Перепиши rate limit middleware из файла 06 так, чтобы он работал для нескольких инстансов сервиса: fixed window (`INCR` + `EXPIRE`), затем sliding window на Sorted Set.
- [ ] ⭐⭐ **Лидерборд.** Мини-API игры: `POST /scores {user, score}`, `GET /leaderboard?top=10`, `GET /users/{name}/rank` на Sorted Set.
- [ ] ⭐⭐ **Singleflight.** Сымитируй 1000 одновременных запросов к одному холодному ключу и посчитай, сколько раз сходили в «БД». Добавь `singleflight` и сравни.
- [ ] ⭐⭐⭐ **Распределённая блокировка.** Реализуй `Lock(ctx, key, ttl)` и `Unlock` с проверкой владельца через Lua-скрипт. Запусти 5 процессов, которые пытаются выполнить «задачу по расписанию»: выполнять должен только один.
- [ ] ⭐⭐ **URL shortener.** Переведи сокращатель ссылок на Postgres (хранение) + Redis (кэш редиректов и счётчик переходов через `INCR`, периодически сбрасываемый в БД).

## Вопросы для самопроверки
1. Почему Redis быстрый? Почему команды атомарны?
2. Как работает cache-aside? Какие у него проблемы?
3. Что такое cache stampede и как с ним бороться?
4. Чем RDB отличается от AOF?
5. Почему нельзя использовать `KEYS *` в продакшене?
