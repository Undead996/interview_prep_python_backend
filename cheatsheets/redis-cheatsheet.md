# Шпаргалка: Redis — последний день

---

## CLI

```bash
redis-cli
  SET key value [EX 3600]
  GET key
  INCR counter
  LPUSH queue value
  RPOP queue
  KEYS *      # НЕ ИСПОЛЬЗОВАТЬ НА ПРОДЕ!
  SCAN 0      # вместо KEYS
  INFO
  FLUSHALL    # не на проде!
```

## Структуры

| Тип | Команды | Когда |
|-----|---------|-------|
| String | `GET`, `SET`, `SETEX`, `INCR` | Кэш, счётчики |
| List | `LPUSH`, `RPOP`, `LLEN` | Очереди (без гарантий) |
| Set | `SADD`, `SMEMBERS`, `SINTER` | Уникальные значения |
| Hash | `HSET`, `HGET`, `HGETALL` | Объекты |
| Sorted Set | `ZADD`, `ZRANGE`, `ZREVRANGE` | Рейтинги |
| Stream | `XADD`, `XREADGROUP`, `XACK` | Надёжные очереди |

## Python (redis.asyncio)

```python
r = aioredis.Redis(decode_responses=True)
await r.setex("key", 3600, "value")
await r.get("key")
await r.incr("counter")
```

## Persistence

```
RDB (dump.rdb)  — снэпшоты (потеря минут)
AOF             — лог запросов (потеря секунды)
RDB + AOF       — лучшее из двух
```

## HA

```
Sentinel — failover master → replica (HA)
Cluster  — шардирование (16384 слотов) + HA
```

## Rate limiting

```python
key = f"rl:{ip}:{int(time.time()) // 60}"
count = await r.incr(key)
await r.expire(key, 61)
```

## Distributed Lock

```python
lock_value = str(uuid.uuid4())
await r.setnx(f"lock:{name}", lock_value)
await r.expire(f"lock:{name}", 10)
# Lua для атомарного удаления:
"if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end"
```