# Redis: структуры данных и кэширование

---

## Strings (самое простое)

```python
import redis.asyncio as aioredis

r = await aioredis.Redis()

# Базовые операции
await r.set("user:42:name", "Alice")
name = await r.get("user:42:name")  # b"Alice"

# С TTL
await r.setex("session:token", 3600, "user_data")
await r.expire("temp_key", 300)

# Счётчики
await r.incr("page_views")
await r.incrby("daily:visitors", 5)
```

---

## Lists (очереди/стеки)

```python
# FIFO очередь
await r.lpush("tasks", "task1")
await r.lpush("tasks", "task2")
task = await r.rpop("tasks")  # task1

# LIFO стек
await r.lpush("history", "action1")
await r.lpush("history", "action2")
last = await r.lpop("history")  # action2

# Trim — держать последние N
await r.ltrim("recent_actions", 0, 99)

# Длина
count = await r.llen("queue")
```

---

## Sets (множества — уникальные значения)

```python
# Уникальные email
await r.sadd("emails", "alice@ex.com")
await r.sadd("emails", "alice@ex.com")  # 0 — уже существует

all_emails = await r.smembers("emails")  # все unique

# Операции множеств
await r.sadd("users:active", "alice", "bob")
await r.sadd("users:vip", "alice")
await r.sinter("users:active", "users:vip")  # {"alice"} — пересечение
await r.sdiff("users:active", "users:vip")     # {"bob"} — разность
```

---

## Hashes (объекты)

```python
# Хранение объектов (лучше, чем JSON в string)
await r.hset("user:42", mapping={
    "name": "Alice",
    "email": "alice@ex.com",
    "age": 30,
})
name = await r.hget("user:42", "name")  # b"Alice"
all_data = await r.hgetall("user:42")   # {b"name": b"Alice", ...}

# Инкремент поля
await r.hincrby("user:42", "login_count", 1)
```

---

## Sorted Sets (рейтинги)

```python
# Рейтинг
await r.zadd("leaderboard", {"Alice": 100, "Bob": 85, "Charlie": 95})

# Топ-3
top = await r.zrevrange("leaderboard", 0, 2, withscores=True)
# [(b"Alice", 100.0), (b"Charlie", 95.0), (b"Bob", 85.0)]

# Диапазон по очкам
between = await r.zrangebyscore("leaderboard", 90, 100)

# Количество
count = await r.zcard("leaderboard")
```

---

## Streams (Redis 5.0+ — надёжные очереди)

```python
# Producer
msg_id = await r.xadd("mystream", {"event": "order.created", "order_id": "42"})
# "1712345678-0"

# Consumer group (аналог Kafka consumer group)
await r.xgroup_create("mystream", "processors", id="0")

# Consumer читает
messages = await r.xreadgroup("processors", "worker1", {"mystream": ">"}, count=10)
for stream, msgs in messages:
    for msg_id, fields in msgs:
        print(fields)
        await r.xack("mystream", "processors", msg_id)
```

---

## Стратегии кэширования

| Стратегия | Описание | Когда |
|---|---|---|
| **Cache Aside** | Приложение читает из Redis → при промахе — из БД → пишет в Redis | Универсально |
| **Write Through** | Пишем и в БД, и в Redis одновременно | Нужна консистентность |
| **Write Behind** | Пишем в Redis, асинхронно — в БД | Высокая скорость записи |
| **Stale-while-revalidate** | Отдаём старый кэш, фоном обновляем | Соцсети, ленты |

```python
# Cache Aside
async def get_user(user_id: int) -> User:
    cache_key = f"user:{user_id}"

    # 1. Читаем из кэша
    if cached := await redis.get(cache_key):
        return User.model_validate_json(cached)

    # 2. Промах — читаем из БД
    user = await db.get(User, user_id)
    if not user:
        return None

    # 3. Пишем в кэш
    await redis.setex(cache_key, 3600, user.model_dump_json())
    return user
```

---

## Подводные камни

| ❌ Ошибка | ✅ Правильно |
|---|---|
| `KEYS *` | `SCAN 0` — не блокирует сервер |
| Нет TTL | Redis переполнится → eviction |
| Redis как надёжная очередь (List) | Для надёжности — Streams |
| `pool = redis.ConnectionPool()` без max_connections | Утечка соединений |
| `INFO` в тестах | `FLUSHALL` для чистки между тестами |

---

> **Технически:** Redis — in-memory key-value store. Все операции атомарны.
> Транзакции — через MULTI/EXEC (не ACID). Lua-скриптинг для атомарных операций.
>
> **На собесе:** «Какие структуры Redis используете?» —
> «Strings для кэша (JSON-сериализованный объект), Hashes для полей объекта,
> Sorted Sets для рейтингов и rate limiting, Streams для надёжных очередей
> (вместо List).»