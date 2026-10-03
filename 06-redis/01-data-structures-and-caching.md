# Redis: структуры данных и кэширование — расширенно

> **Цель:** понимать все 8 структур данных, выбирать правильную под задачу, знать кэш-стратегии.

---

## 1. Почему Redis быстрый

- **In-memory** — данные в RAM, не на диске
- **Однопоточный** — одна команда за раз, нет гонок, нет блокировок
- **O(1) / O(log N)** для всех основных операций
- **Event loop** (как nginx) — мультиплексирование I/O через epoll/kqueue
- **Протокол** — простой текстовый (RESP), минимальный оверхед

**Цена:** потеря данных при падении без persistence. Решается RDB/AOF.

---

## 2. Все 8 структур данных — когда какую

| # | Структура | Команды | Когда | Пример из бэкенда |
|---|----------|---------|-------|-------------------|
| 1 | **String** | `GET`, `SET`, `SETEX`, `INCR`, `DECR`, `GETSET` | Кэш, счётчики, сериализованные объекты | `SETEX user:42 3600 '{...}'` |
| 2 | **List** | `LPUSH`, `RPUSH`, `LPOP`, `RPOP`, `LLEN`, `LRANGE` | Простые очереди/стеки, последние N событий | `LPUSH queue task; RPOP queue` |
| 3 | **Set** | `SADD`, `SREM`, `SMEMBERS`, `SINTER`, `SUNION`, `SCARD` | Уникальные значения, тэги, пересечения | `SADD online:users 42` |
| 4 | **Hash** | `HSET`, `HGET`, `HGETALL`, `HDEL`, `HINCRBY` | Объекты с полями, денормализация | `HSET user:42 name Alice email alice@ex.com` |
| 5 | **Sorted Set** | `ZADD`, `ZRANGE`, `ZREVRANGE`, `ZRANK`, `ZSCORE` | Рейтинги, лидерборды, rate limiting (sliding window) | `ZADD leaderboard 1000 Alice` |
| 6 | **Stream** | `XADD`, `XREADGROUP`, `XACK`, `XPENDING` | Надёжные очереди (consumer groups, ack) | `XADD mystream * order_id 42` |
| 7 | **Bitmap** | `SETBIT`, `GETBIT`, `BITCOUNT`, `BITOP` | Аналитика, флаги, unique users | `SETBIT page:views:2024-01-01 user_id 1` |
| 8 | **HyperLogLog** | `PFADD`, `PFCOUNT`, `PFMERGE` | Приблизительный подсчёт уникальных (~0.81% ошибка, 12KB) | `PFADD unique:visitors 192.168.1.1` |

### Когда Hash, а когда String (JSON)?

```python
# ✅ HASH — когда нужны отдельные поля:
await redis.hset("user:42", mapping={"name": "Alice", "age": "30", "email": "alice@ex.com"})
await redis.hget("user:42", "email")  # только email, не весь объект
await redis.hincrby("user:42", "age", 1)  # атомарный инкремент поля!

# ✅ STRING (JSON) — когда объект всегда нужен целиком:
await redis.setex("user:42", 3600, json.dumps(user_dict))
user = json.loads(await redis.get("user:42"))
```

---

## 3. Кэш-стратегии — когда какую

### Cache Aside (самая популярная)

```
ЧТЕНИЕ:
1. Ищем в Redis → нашли (hit) → вернули
2. Не нашли (miss) → читаем из БД → пишем в Redis с TTL → вернули

ЗАПИСЬ:
1. Пишем в БД
2. Удаляем ключ из Redis (инвалидация)
   или обновляем кэш (если данные уже есть)
```

```python
@app.get("/api/users/{user_id}")
async def get_user(user_id: int, redis: Redis = Depends(get_redis)):
    cache_key = f"user:{user_id}"
    if cached := await redis.get(cache_key):
        return json.loads(cached)

    user = await db.get(User, user_id)
    if not user:
        raise HTTPException(404)

    await redis.setex(cache_key, 3600, json.dumps(user))
    return user
```

**Плюсы:** просто, TTL решает инвалидацию, кэш не мешает записям.

**Минусы:** первый запрос медленный (cache miss), возможен stale read (между записью в БД и инвалидацией).

### Write Through

```
ЗАПИСЬ: БД + Redis (синхронно в одной транзакции)
ЧТЕНИЕ: всегда из Redis (данные всегда свежие)
```

**Когда:** консистентность критична, данные меняются редко.

### Write Behind

```
ЗАПИСЬ: только в Redis → асинхронно в БД (фоновый процесс)
ЧТЕНИЕ: из Redis
```

**Когда:** высокая скорость записи, терпимость к потере данных.

### Write Around

```
ЗАПИСЬ: только в БД, кэш НЕ обновляется
ЧТЕНИЕ: Cache Aside (мисс → из БД)
```

**Когда:** данные редко читаются после записи.

---

## 4. Инвалидация кэша — 4 способа

```python
# 1. TTL — автоматическое истечение
await redis.setex("key", 3600, "value")

# 2. Event-driven — удаление при изменении
@app.put("/api/users/{user_id}")
async def update_user(user_id: int, payload: UserUpdate, redis: Redis = Depends(get_redis)):
    user = await update_in_db(...)
    await redis.delete(f"user:{user_id}")  # invalidate!
    return user

# 3. Versioned key — user:42:v2
version = await redis.incr(f"user:{user_id}:version")
key = f"user:{user_id}:v{version}"

# 4. Namespaced — user:{version}:{user_id}
# Удаляем все ключи namespace'a при инвалидации
```

---

## 5. TTL — как работает удаление истёкших ключей

**Пассивно:** при чтении ключа Redis проверяет — истёк? → удаляет.

**Активно:** каждые 100ms Redis сканирует **случайную** выборку expired-ключей (по умолчанию ~20 активных). Если > 25% истекли — повторяет.

**Настраивается:** `hz 10` (частота, default 10 раз в секунду), `active-expire-effort 1-10`.

**⚠️ Не ставьте 1ms TTL!** Redis будет тратить CPU на постоянную очистку.

---

> **На собесе:** «Какие структуры Redis используете?» —
> «Strings для кэша (JSON), Hashes для объектов с атомарными полями, Sorted Sets для рейтингов и rate limiting, Streams для надёжных очередей с ack. Bitmaps для флагов, HyperLogLog для приблизительного подсчёта уникальных. List — только для временных очередей без гарантий.»