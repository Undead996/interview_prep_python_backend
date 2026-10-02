# Шпаргалка: Redis

---

## CLI

```bash
redis-cli
  SET key value
  GET key
  EXPIRE key 3600
  INCR counter
  LPUSH queue value
  RPOP queue
  KEYS *      # НЕ ИСПОЛЬЗОВАТЬ НА ПРОДЕ!
  SCAN 0      # вместо KEYS
  INFO
  FLUSHALL    # не на проде!
```

## Структуры

| Тип | Команды |
|---|---|
| String | `GET`, `SET`, `SETEX`, `INCR`, `INCRBY` |
| List | `LPUSH`, `RPUSH`, `LPOP`, `RPOP`, `LLEN`, `LRANGE` |
| Set | `SADD`, `SMEMBERS`, `SINTER`, `SUNION`, `SCARD` |
| Hash | `HSET`, `HGET`, `HGETALL`, `HDEL`, `HINCRBY` |
| Sorted Set | `ZADD`, `ZRANK`, `ZREVRANK`, `ZRANGE`, `ZREVRANGE` |
| Stream | `XADD`, `XREAD`, `XREADGROUP`, `XACK`, `XGROUP` |

## Python (redis.asyncio)

```python
r = aioredis.Redis(decode_responses=True)
await r.setex("key", 3600, "value")
await r.get("key")
await r.incr("counter")
```

## Persistence

```
RDB (dump.rdb)  — снэпшоты
AOF             — лог запросов
RDB + AOF       — лучшее из двух
```

## HA

```
Sentinel — failover master → replica
Cluster  — шардирование (16384 слотов) + HA
```

## Rate limiting

```python
key = f"rl:{ip}:{int(time.time()) // 60}"
count = await r.incr(key)
await r.expire(key, 61)
```