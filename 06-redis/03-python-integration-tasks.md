# Redis + Python: интеграция и задачи

---

## redis-py (синхронно)

```python
import redis

r = redis.Redis(host="localhost", port=6379, db=0, decode_responses=True)

r.set("key", "value")
r.get("key")  # "value" (decode_responses=True)

# Пул соединений
pool = redis.ConnectionPool(host="localhost", port=6379, max_connections=20)
r = redis.Redis(connection_pool=pool)
```

---

## redis.asyncio (async/await)

```python
import redis.asyncio as aioredis

async def example():
    r = aioredis.Redis(
        host="localhost",
        port=6379,
        decode_responses=True,
        socket_connect_timeout=5,
    )

    # Pipeline — атомарное выполнение группы команд
    async with r.pipeline() as pipe:
        pipe.set("key1", "value1")
        pipe.incr("counter")
        pipe.get("key1")
        result = await pipe.execute()  # [True, 1, "value1"]

    # Transaction (MULTI/EXEC)
    async with r.pipeline(transaction=True) as pipe:
        pipe.set("a", "1")
        pipe.set("b", "2")
        await pipe.execute()

    await r.aclose()
```

---

## FastAPI + Redis

```python
from fastapi import FastAPI
from contextlib import asynccontextmanager
import redis.asyncio as aioredis


@asynccontextmanager
async def lifespan(app):
    app.state.redis = aioredis.Redis(
        host="localhost",
        decode_responses=True,
    )
    yield
    await app.state.redis.aclose()


app = FastAPI(lifespan=lifespan)


@app.get("/api/users/{user_id}")
async def get_user(user_id: int):
    cache_key = f"user:{user_id}"

    # Cache Aside
    if cached := await app.state.redis.get(cache_key):
        return User.model_validate_json(cached)

    user = await db.get(User, user_id)
    if not user:
        raise HTTPException(status_code=404)

    await app.state.redis.setex(cache_key, 3600, user.model_dump_json())
    return user


@app.get("/api/cache/stats")
async def cache_stats():
    info = await app.state.redis.info()
    return {
        "hits": info["keyspace_hits"],
        "misses": info["keyspace_misses"],
        "used_memory": info["used_memory_human"],
    }
```

---

## Задача 1. Rate limiter для API

**Условие:** Напишите middleware, которое ограничивает 10 запросов в минуту по IP.

<details>
<summary>Решение</summary>

```python
import time

async def rate_limit_middleware(request, redis):
    ip = request.client.host
    key = f"ratelimit:{ip}:{int(time.time()) // 60}"

    count = await redis.incr(key)
    if count == 1:
        await redis.expire(key, 61)

    if count > 10:
        raise HTTPException(
            status_code=429,
            detail="Too many requests",
            headers={"Retry-After": str(60 - (int(time.time()) % 60))},
        )
```
</details>

---

## Задача 2. Distributed lock для конкурентного доступа

**Условие:** Напишите декоратор, который блокирует выполнение функции,
чтобы её не выполнили два сервиса одновременно.

<details>
<summary>Решение</summary>

```python
import functools
import uuid

def distributed_lock(lock_key: str, ttl: int = 30):
    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            redis = kwargs.get("redis") or args[0]  # предположим
            lock_value = str(uuid.uuid4())

            acquired = await redis.setnx(f"lock:{lock_key}", lock_value)
            if not acquired:
                raise RuntimeError(f"Resource locked: {lock_key}")

            await redis.expire(f"lock:{lock_key}", ttl)
            try:
                return await func(*args, **kwargs)
            finally:
                # Атомарное удаление (если наш замок)
                script = """
                if redis.call("get", KEYS[1]) == ARGV[1] then
                    return redis.call("del", KEYS[1])
                end
                """
                await redis.eval(script, 1, f"lock:{lock_key}", lock_value)
        return wrapper
    return decorator
```
</details>

---

> **На собесе:** «Кэширование — как инвалидировать кэш?» —
> «1) TTL (setex) — автоматическое истечение. 2) Event-driven инвалидация —
> при изменении данных удаляем ключ. 3) Version key — user:42:v2.
> 4) Write Through — обновляем БД и кэш в одной транзакции.
> Выбор зависит от того, критична ли история или достаточно TTL.»