# Redis: Persistence, HA, Advanced Patterns — расширенно

> **Цель:** понимать RDB vs AOF, Sentinel vs Cluster, distributed locks, rate limiting.

---

## 1. Persistence — RDB vs AOF (детально)

| Характеристика | RDB (snapshot) | AOF (append-only file) | RDB + AOF (гибрид) |
|---------------|---------------|----------------------|-------------------|
| **Что сохраняет** | Полный снэпшот (`dump.rdb`) | Лог каждой команды записи | RDB + инкрементальный AOF |
| **Размер файла** | Компактный | Большой (растёт) | Средний |
| **Запись** | Раз в N минут (настраивается) | Каждую секунду (или каждый запрос) | Смешанный |
| **Восстановление** | Быстрое | Медленное (проигрывание лога) | Быстрое (RDB) + точное (AOF) |
| **Потеря данных при сбое** | Последние ~минуты | Последняя секунда (по умолчанию) | Последняя секунда |
| **CPU / I/O при записи** | Высокий (fork + запись) | Низкий (только append) | Низкий |

```bash
# redis.conf
# RDB:
save 900 1        # 1 изменение за 15 мин → снапшот
save 300 10       # 10 изменений за 5 мин
save 60 10000     # 10K изменений за 1 мин

# AOF:
appendonly yes
appendfsync everysec   # fsync каждую секунду (баланс)
# appendfsync always   # fsync КАЖДЫЙ запрос (медленно, надёжно)
# appendfsync no       # fsync по решению ОС (быстро, ненадёжно)

# AOF rewrite (сжатие лога):
auto-aof-rewrite-percentage 100  # когда AOF вырос на 100%
auto-aof-rewrite-min-size 64mb
```

**Выбор:**
| Сценарий | Рекомендация |
|----------|-------------|
| Кэш, данные восстанавливаются из БД | Только RDB (или вообще без persistence) |
| Данные критичны, потеря пары секунд допустима | AOF everysec |
| Быстрое восстановление + точность | RDB + AOF (гибридный, Redis 5+) |
| Абсолютная надёжность | AOF always (очень медленно) |

---

## 2. Sentinel — High Availability (без шардирования)

```
Архитектура:
  Sentinel 1 (26379) ─┐
  Sentinel 2 (26379) ─┼─ Наблюдают за мастером и репликами
  Sentinel 3 (26379) ─┘

  Redis Master (6379) ← клиенты пишут сюда
  Redis Replica 1 (6380) ← асинхронная репликация
  Redis Replica 2 (6381) ← асинхронная репликация

Процесс failover:
1. Sentinel обнаружил, что мастер недоступен (quorum = 2/3)
2. Выбирает новую мастер-реплику (наиболее актуальные данные)
3. Переконфигурирует реплики на нового мастера
4. Клиенты через Sentinel API узнают нового мастера
```

```python
from redis.asyncio.sentinel import Sentinel

sentinel = Sentinel(
    [("sentinel1", 26379), ("sentinel2", 26379), ("sentinel3", 26379)],
    socket_timeout=0.5,
)
master = sentinel.master_for("mymaster", decode_responses=True)
slave = sentinel.slave_for("mymaster", decode_responses=True)

await master.set("key", "value")   # запись → мастер
value = await slave.get("key")     # чтение → реплика
```

---

## 3. Cluster — шардирование + HA

```
16384 хэш-слотов распределены по N мастерам.
Каждый мастер может иметь реплики (HA).

  Master 1 (slots 0-5460)     ← Replica 1
  Master 2 (slots 5461-10922) ← Replica 2
  Master 3 (slots 10923-16383) ← Replica 3

Ключ → CRC16(key) % 16384 → номер слота → мастер.
Если операция затрагивает ключи на разных мастерах → ошибка CROSSSLOT.
Используйте hash tags: {user:42}:profile, {user:42}:orders → оба на одном мастере.
```

```python
from redis.asyncio.cluster import RedisCluster

rc = RedisCluster(host="localhost", port=7000, decode_responses=True)
await rc.set("{user:42}:name", "Alice")   # автоматически на нужный мастер
await rc.get("{user:42}:name")            # автоматически с нужного мастера
```

---

## 4. Rate Limiting — две реализации

### Sliding window (простая, эффективная)

```python
async def check_sliding_window(redis: Redis, key: str, max_calls: int, window: int) -> bool:
    """Возвращает True, если запрос разрешён."""
    current = int(time.time()) // window
    redis_key = f"ratelimit:{key}:{current}"

    count = await redis.incr(redis_key)
    if count == 1:
        await redis.expire(redis_key, window + 1)
    return count <= max_calls
```

### Token bucket (точнее, сложнее)

```lua
-- Lua-скрипт для token bucket:
local key = KEYS[1]
local rate = tonumber(ARGV[1])     -- tokens per second
local burst = tonumber(ARGV[2])    -- max burst
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4]) -- обычно 1

local data = redis.call('HMGET', key, 'tokens', 'last')
local tokens = tonumber(data[1]) or burst
local last = tonumber(data[2]) or now

-- Refill:
local elapsed = now - last
tokens = math.min(burst, tokens + elapsed * rate)

local allowed = 0
if tokens >= requested then
    tokens = tokens - requested
    allowed = 1
end

redis.call('HMSET', key, 'tokens', tokens, 'last', now)
redis.call('EXPIRE', key, math.ceil(burst / rate) + 1)
return allowed
```

---

## 5. Distributed Lock — правильная реализация

```python
import uuid

async def acquire_lock(redis: Redis, name: str, ttl: int = 10) -> str | None:
    """Получить блокировку. Возвращает lock_value (нужен для release)."""
    lock_value = str(uuid.uuid4())
    acquired = await redis.set(f"lock:{name}", lock_value, nx=True, ex=ttl)
    return lock_value if acquired else None

async def release_lock(redis: Redis, name: str, lock_value: str) -> bool:
    """Освободить блокировку (атомарно — только свою)."""
    script = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    """
    result = await redis.eval(script, 1, f"lock:{name}", lock_value)
    return bool(result)

# Redlock (для распределённой среды):
# 1. Получаем lock_value = SET lock NX PX ttl на N независимых Redis-узлах
# 2. Если получено на большинстве (N/2+1) за время < ttl → lock acquired
# 3. Release на всех узлах
```

**Почему Lua?** Гарантирует атомарность: между GET и DEL другой процесс не может вклиниться.

---

## 6. Lua-скриптинг — другие применения

```python
# Атомарный инкремент + проверка (без race condition):
lua_incr_and_check = """
local current = redis.call('INCR', KEYS[1])
if current > tonumber(ARGV[1]) then
    return 0
end
redis.call('EXPIRE', KEYS[1], ARGV[2])
return current
"""

result = await redis.eval(lua_incr_and_check, 1, "page_views", "1000", "3600")
```

---

> **На собесе:** «Redis: как хранить данные надёжно?» —
> «RDB — снэпшоты (потеря минут), AOF — лог запросов (потеря секунды). Рекомендую гибридный RDB + AOF. Для HA — Sentinel (3+ узла для кворума). Для шардирования — Cluster (16384 слотов). Distributed lock — SET NX PX + Lua для атомарного release. Rate limiting — sliding window (INCR + EXPIRE) или token bucket (Lua).»