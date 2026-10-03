# Redis: Persistence, Advanced, Patterns (расширенно)

> **Цель:** понять RDB vs AOF, Sentinel vs Cluster, distributed locks, rate limiting.

---

## 1. Persistence — RDB vs AOF

| Режим | Что сохраняет | Размер | Запись | Восстановление |
|-------|--------------|--------|--------|---------------|
| **RDB** | Снэпшот (dump.rdb) | Компактный | 1 раз в N минут | Быстрое |
| **AOF** | Лог каждой записи (appendonly.aof) | Большой | Каждую секунду | Медленнее |
| **RDB + AOF** | Оба | RDB + AOF | Смешанный | AOF (новый формат) |

```bash
# redis.conf
save 900 1       # RDB: 1 изменение за 15 мин
save 300 10      # RDB: 10 изменений за 5 мин
appendonly yes   # AOF
appendfsync everysec  # fsync каждую секунду
```

**Выбор:**
- Терпите потерю 5 минут данных? → только RDB
- Каждая запись критична? → AOF (потеря 1 секунды)
- Хотите быстрое восстановление? → RDB + AOF (новый гибридный формат)

---

## 2. Sentinel — High Availability

```
Sentinel — это "наблюдатель" за Redis-серверами.
Он:
1. Мониторит мастер (ping каждую секунду)
2. Если мастер упал — выбирает нового из реплик
3. Меняет конфигурацию (клиенты узнают нового мастера)
```

**Схема:** 3+ Sentinels для кворума. Если ≥2 согласны, что мастер упал → failover.

```python
from redis.sentinel import Sentinel
sentinel = Sentinel([("localhost", 26379)])
master = sentinel.master_for("mymaster")
slave = sentinel.slave_for("mymaster")
```

---

## 3. Cluster — шардирование

```
16384 хэш-слотов распределены по N мастерам.
Каждый ключ → CRC16(key) % 16384 → номер мастера.
```

**В отличие от Sentinel:** Cluster не просто HA, но и распределение данных. Каждый мастер хранит свою часть данных.

```python
from redis.cluster import RedisCluster
rc = RedisCluster(host="localhost", port=7000)
await rc.set("key", "value")  # автоматически на нужный мастер
```

---

## 4. Rate Limiting (sliding window)

```python
async def check_rate_limit(user_id: str, max_calls: int = 10, window: int = 60):
    # Ключ вида ratelimit:user_42:2024-01-01-14:00 -> счётчик за эту минуту
    current_window = int(time.time()) // window
    key = f"ratelimit:{user_id}:{current_window}"
    
    count = await redis.incr(key)
    if count == 1:
        await redis.expire(key, window + 1)  # +1 запас
    return count <= max_calls
```

---

## 5. Distributed Lock

```python
async def acquire_lock(lock_name: str, ttl: int = 10):
    lock_value = str(uuid.uuid4())
    acquired = await redis.setnx(f"lock:{lock_name}", lock_value)
    if acquired:
        await redis.expire(f"lock:{lock_name}", ttl)
        return lock_value
    return None

async def release_lock(lock_name: str, lock_value: str):
    # Lua script — атомарно проверяем и удаляем
    script = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    """
    return await redis.eval(script, 1, f"lock:{lock_name}", lock_value)
```

**Почему Lua script?** Чтобы гарантировать: "мы удаляем только свой замок". Без Lua возможна гонка: мы проверяем (get), но в этот момент другой процесс установил свой замок.

---

> **На собесе:** «Redis Persistence — RDB vs AOF?» —  
> «RDB — снэпшоты (dump.rdb), компактные, быстрый старт, потеря последней
> минуты. AOF — лог каждой операции, потеря последней секунды, больше места.
> Рекомендую RDB + AOF — гибридный режим.»