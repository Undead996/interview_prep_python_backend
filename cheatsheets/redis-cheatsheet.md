# Redis Cheatsheet

## CLI

```bash
redis-cli
  SET key value [EX 3600]
  GET key
  DEL key
  INCR counter
  LPUSH q v; RPOP q
  SADD set val; SMEMBERS set
  HSET h field val; HGETALL h
  ZADD lb 100 player; ZREVRANGE lb 0 9
  KEYS *       # ❌ НИКОГДА на проде!
  SCAN 0        # ✅ итеративный
  INFO; FLUSHALL ASYNC
```

## Структуры

| Тип | Команды | Для чего |
|-----|---------|---------|
| String | GET,SET,SETEX,INCR | Кэш, счётчики |
| List | LPUSH,RPOP,LLEN | Простые очереди |
| Set | SADD,SINTER,SCARD | Уникальные, тэги |
| Hash | HSET,HGET,HGETALL | Объекты с полями |
| Sorted Set | ZADD,ZRANGE,ZRANK | Рейтинги, rate lim |
| Stream | XADD,XREADGROUP,XACK | Надёжные очереди |
| Bitmap | SETBIT,BITCOUNT | Флаги, аналитика |
| HyperLogLog | PFADD,PFCOUNT | ~уникальные (12KB) |

## redis.asyncio

```python
pool = aioredis.ConnectionPool.from_url("redis://localhost", decode_responses=True)
r = aioredis.Redis(connection_pool=pool)
await r.setex("key", 3600, "value")
await r.get("key")
await r.incr("counter")

# Pipeline (неатомарный, быстрый):
async with r.pipeline() as pipe:
    pipe.set("a", "1"); pipe.incr("c")
    results = await pipe.execute()

# Transaction (атомарный):
async with r.pipeline(transaction=True) as pipe:
    pipe.set("a", "1"); pipe.set("b", "2")
    await pipe.execute()
```

## Persistence

```
RDB (dump.rdb)  — снапшоты (потеря минут)
AOF             — лог запросов (потеря секунды)
RDB + AOF       — гибрид (лучшее из двух)
```

## HA

```
Sentinel — failover (3+ Sentinel, quorum)
Cluster  — шардирование (16384 слотов) + HA
```

## Rate Limiting (sliding window)

```python
key = f"rl:{ip}:{int(time.time()) // 60}"
count = await r.incr(key)
if count == 1: await r.expire(key, 61)
if count > 100: raise HTTPException(429)
```

## Distributed Lock

```python
# Acquire:
lock_val = str(uuid.uuid4())
ok = await r.set(f"lock:{name}", lock_val, nx=True, ex=10)
# Release (Lua — атомарно!):
script = """if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1]) else return 0 end"""
await r.eval(script, 1, f"lock:{name}", lock_val)
```

## Кэш-стратегии

```
Cache Aside: читаем Redis → miss → БД → пишем в Redis
Write Through: пишем в БД + Redis синхронно
Write Behind: пишем в Redis → асинхронно в БД
Инвалидация: TTL | DEL при изменении | versioned keys
```