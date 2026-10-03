# Redis + Python: интеграция и задачи — расширенно

---

## 1. redis.asyncio — современный клиент

```python
import redis.asyncio as aioredis

# Connection pool (всегда используйте pool для продакшена!)
pool = aioredis.ConnectionPool(
    host="localhost", port=6379, db=0,
    max_connections=20,
    decode_responses=True,
    socket_connect_timeout=5,
    socket_keepalive=True,
    retry_on_timeout=True,
)
r = aioredis.Redis(connection_pool=pool)

# Базовые операции:
await r.set("key", "value")
await r.setex("key", 3600, "value")     # TTL 1 час
await r.get("key")                       # "value"
await r.delete("key")
await r.exists("key")                    # 0 или 1
await r.expire("key", 60)                # продлить TTL
await r.ttl("key")                       # оставшееся время в секундах

# Pipeline (группа команд без атомарности, но одним round-trip):
async with r.pipeline() as pipe:
    pipe.set("a", "1")
    pipe.incr("counter")
    pipe.get("a")
    results = await pipe.execute()  # [True, 1, "1"]

# Transaction (MULTI/EXEC) — атомарно:
async with r.pipeline(transaction=True) as pipe:
    pipe.set("a", "1")
    pipe.set("b", "2")
    await pipe.execute()  # атомарно!

# Watch + MULTI/EXEC — optimistic locking:
async with r.pipeline() as pipe:
    await pipe.watch("balance")
    balance = int(await r.get("balance") or 0)
    if balance >= 100:
        pipe.multi()
        pipe.decrby("balance", 100)
        pipe.incrby("spent", 100)
        await pipe.execute()
```

---

## 2. FastAPI + Redis (lifespan + cache middleware)

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    pool = aioredis.ConnectionPool.from_url("redis://localhost:6379/0", decode_responses=True)
    app.state.redis = aioredis.Redis(connection_pool=pool)
    yield
    await app.state.redis.aclose()

app = FastAPI(lifespan=lifespan)

# Cache Aside endpoint:
@app.get("/api/users/{user_id}")
async def get_user(user_id: int):
    redis: aioredis.Redis = request.app.state.redis
    cache_key = f"user:{user_id}"

    if cached := await redis.get(cache_key):
        return User.model_validate_json(cached)

    user = await db.get(User, user_id)
    if not user:
        raise HTTPException(404)

    await redis.setex(cache_key, 3600, user.model_dump_json())
    return user

# Cache invalidation на update:
@app.put("/api/users/{user_id}")
async def update_user(user_id: int, payload: UserUpdate):
    user = await update_in_db(user_id, payload)
    await request.app.state.redis.delete(f"user:{user_id}")
    return user
```

---

## 3. Мониторинг

```python
@app.get("/api/cache/stats")
async def cache_stats(request: Request):
    info = await request.app.state.redis.info("stats")
    memory = await request.app.state.redis.info("memory")
    return {
        "hit_ratio": round(
            info["keyspace_hits"] / (info["keyspace_hits"] + info["keyspace_misses"]) * 100, 1
        ) if (info["keyspace_hits"] + info["keyspace_misses"]) > 0 else 0,
        "hits": info["keyspace_hits"],
        "misses": info["keyspace_misses"],
        "used_memory": memory["used_memory_human"],
        "uptime_days": info.get("uptime_in_days", 0),
    }
```

---

## Задача 1. Rate limiter middleware для FastAPI

```python
@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    redis: aioredis.Redis = request.app.state.redis
    ip = request.client.host if request.client else "unknown"

    current_window = int(time.time()) // 60  # минута
    key = f"ratelimit:{ip}:{current_window}"
    count = await redis.incr(key)
    if count == 1:
        await redis.expire(key, 62)

    if count > 100:
        return JSONResponse(
            status_code=429,
            content={"detail": "Too many requests. Max 100/minute."},
            headers={"Retry-After": str(60 - int(time.time()) % 60)},
        )

    return await call_next(request)
```

---

## Задача 2. Distributed lock декоратор

```python
import functools, uuid

def with_distributed_lock(lock_name: str, ttl: int = 30):
    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            redis = kwargs.get("redis")
            if not redis:
                raise ValueError("redis not found in kwargs")

            lock_value = str(uuid.uuid4())
            acquired = await redis.set(f"lock:{lock_name}", lock_value, nx=True, ex=ttl)
            if not acquired:
                raise RuntimeError(f"Could not acquire lock: {lock_name}")

            try:
                return await func(*args, **kwargs)
            finally:
                script = """
                if redis.call("get", KEYS[1]) == ARGV[1] then
                    return redis.call("del", KEYS[1])
                else
                    return 0
                end
                """
                await redis.eval(script, 1, f"lock:{lock_name}", lock_value)
        return wrapper
    return decorator

@with_distributed_lock("process-payments", ttl=30)
async def process_payments(redis: Redis, batch: list[Payment]):
    ...
```

---

## Задача 3. Надёжная очередь на Streams

```python
# Producer:
await r.xadd("order-stream", {"order_id": "42", "user_id": "7", "total": "99.99"})

# Consumer group:
try:
    await r.xgroup_create("order-stream", "processors", id="0", mkstream=True)
except ResponseError:
    pass  # группа уже существует

messages = await r.xreadgroup("processors", "worker1", {"order-stream": ">"}, count=10, block=5000)
for msg_id, fields in messages[0][1]:
    await process_order(fields)
    await r.xack("order-stream", "processors", msg_id)

# Pending (необработанные):
pending = await r.xpending("order-stream", "processors")
```

---

## Задача 4. Leaderboard (Sorted Set)

```python
# Добавить очки:
await r.zincrby("leaderboard", 42, "Alice")

# Топ-10:
top = await r.zrevrange("leaderboard", 0, 9, withscores=True)
# [("Alice", 42.0), ("Bob", 35.0), ...]

# Ранг игрока:
rank = await r.zrevrank("leaderboard", "Alice")  # 0-based

# Очки игрока:
score = await r.zscore("leaderboard", "Alice")

# Игроки с очками в диапазоне:
players = await r.zrangebyscore("leaderboard", 50, 100)
```

---

> **На собесе:** «Кэширование — как инвалидировать?» —
> «1) TTL — автоматическое истечение (setex). 2) Event-driven — DEL при изменении данных. 3) Versioned keys — user:42:v2. 4) Cache-Aside — пишем в БД, удаляем из кэша. 5) Write-Through — пишем в БД + кэш синхронно. Выбор зависит от того, насколько критична консистентность vs скорость записи.»