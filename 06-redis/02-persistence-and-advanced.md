# Redis: Persistence, Advanced, Patterns

---

## Persistence: RDB vs AOF

| Режим | Описание | Плюсы | Минусы |
|---|---|---|---|
| **RDB** | Снэпшот (dump.rdb) | Компактный, быстрый восстановление | Потеря данных с последнего снэпшота |
| **AOF** | Append-only log (каждая запись) | Мин. потеря (1 сек) | Больше места, медленнее | 
| **RDB + AOF** | Оба | Баланс | — |

```bash
# Настройка в redis.conf
save 900 1       # RDB: 1 изменение за 15 мин
save 300 10      # RDB: 10 изменений за 5 мин
save 60 10000    # RDB: 10000 изменений за 1 мин

appendonly yes   # AOF
appendfsync everysec  # fsync каждую секунду
```

---

## Sentinel — HA

```
         ┌──────────────┐
         │  Sentinel 1  │
         │  Sentinel 2  │────── monitor → Master
         │  Sentinel 3  │
         └──────────────┘
               │
          failover
               │
         ┌─────▼──────┐
         │   Master   │ ←── Replica ──→ Replica
         └────────────┘
```

```python
# Sentinel — получаем мастера
from redis.sentinel import Sentinel

sentinel = Sentinel([("localhost", 26379)], socket_timeout=0.1)
master = sentinel.master_for("mymaster")
slave = sentinel.slave_for("mymaster", db=0)
```

- **Sentinel** мониторит мастер.
- Падение мастера → Sentinel выбирает нового мастера из реплик.
- Клиенты должны спрашивать Sentinel, кто сейчас мастер.

---

## Cluster

```
         ┌─────────┐
         │Cluster  │
         ├────┬────┤
         │M1 │S1  │  ← Master 1 + Slave 1
         ├────┼────┤
         │M2 │S2  │  ← Master 2 + Slave 2
         ├────┼────┤
         │M3 │S3  │  ← Master 3 + Slave 3
         └────┴────┘

16384 хэш-слотов распределены по мастерам.
```

```python
from redis.cluster import RedisCluster

rc = RedisCluster(host="localhost", port=7000)
await rc.set("key", "value")  # автоматически на правильный мастер
```

---

## Rate Limiting (Sliding Window)

```python
import time

async def check_rate_limit(user_id: str, max_calls: int = 10, window: int = 60) -> bool:
    key = f"ratelimit:{user_id}:{int(time.time()) // window}"
    count = await redis.incr(key)
    if count == 1:
        await redis.expire(key, window + 1)  # +1 запас
    return count <= max_calls
```

---

## Distributed Lock

```python
import asyncio
import uuid

async def acquire_lock(lock_name: str, ttl: int = 10) -> str | None:
    lock_value = str(uuid.uuid4())
    acquired = await redis.setnx(f"lock:{lock_name}", lock_value)
    if acquired:
        await redis.expire(f"lock:{lock_name}", ttl)
        return lock_value
    return None

async def release_lock(lock_name: str, lock_value: str) -> bool:
    # Lua script — атомарно
    script = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    """
    return await redis.eval(script, 1, f"lock:{lock_name}", lock_value)
```

---

## Pub/Sub (но не для надежной доставки)

```python
# Publisher
await redis.publish("notifications", "new_order")

# Subscriber (требует постоянного соединения!)
pubsub = redis.pubsub()
await pubsub.subscribe("notifications")
async for message in pubsub.listen():
    if message["type"] == "message":
        print(message["data"])
```

> ⚠️ Pub/Sub не гарантирует доставку — если subscriber отключился, сообщение потеряно.
> Для надёжной доставки используйте Streams.

---

## Подводные камни

| ❌ Ошибка | ✅ Правильно |
|---|---|
| Pub/Sub для очередей | Используй Streams |
| `KEYS *` на проде | `SCAN` — не блокирует |
| Нет TTL на ключах | Redis eviction — непредсказуем |
| Одна инстанция для всего | Read replica для чтения, master для записи |
| Lock без проверки владельца | Lua script с проверкой UUID владельца |

---

> **Технически:** Redis — однопоточный (одна команда за раз), поэтому все операции
> атомарны. Lua-скриптинг — для атомарных составных операций.
> Sentinel — для failover (автоматическая смена мастера).
> Cluster — для шардирования (16384 слотов, распределённых по мастерам).
>
> **На собесе:** «Redis Persistence — RDB vs AOF?» — «RDB — снэпшоты (dump.rdb),
> компактные, быстрый старт, потеря последних минут. AOF — лог каждой операции,
> потеря последней секунды, больше места. Хорошая практика — RDB + AOF.»