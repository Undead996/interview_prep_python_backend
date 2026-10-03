# Задачи Python — 5 задач с разбором

---

## Задача 1. Bulk insert с async + connection pool

```python
import asyncpg

async def bulk_insert_users(pool: asyncpg.Pool, users: list[dict], batch_size: int = 100) -> int:
    total = 0
    async with pool.acquire() as conn:
        async with conn.transaction():
            for i in range(0, len(users), batch_size):
                batch = users[i : i + batch_size]
                values = [(u["name"], u["email"]) for u in batch]
                await conn.executemany(
                    "INSERT INTO users (name, email) VALUES ($1, $2) "
                    "ON CONFLICT (email) DO NOTHING",
                    values,
                )
                total += len(batch)
    return total
```

**Почему батчи:** 10 round-trips вместо 1000 = ~100x быстрее. Транзакция = все или ничего.

---

## Задача 2. Rate limiting декоратор

```python
import time, functools

class RateLimitExceeded(Exception):
    def __init__(self, retry_after: float):
        self.retry_after = retry_after
        super().__init__(f"Rate limit exceeded. Retry after {retry_after:.1f}s")

def rate_limit(max_calls: int, period: float):
    def decorator(func):
        timestamps: list[float] = []
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            now = time.monotonic()
            timestamps[:] = [t for t in timestamps if now - t < period]
            if len(timestamps) >= max_calls:
                retry = period - (now - timestamps[0]) if timestamps else period
                raise RateLimitExceeded(retry)
            timestamps.append(now)
            return func(*args, **kwargs)
        return wrapper
    return decorator
```

**Почему `timestamps[:] = ...`:** модифицирует список в замыкании на месте, а не создаёт локальную переменную.

---

## Задача 3. Async-генератор пагинации с retry и timeout

```python
import httpx
from typing import AsyncIterator

async def paginate(url: str, page_size: int = 100, max_pages: int | None = None) -> AsyncIterator[dict]:
    page = 1
    async with httpx.AsyncClient(timeout=30.0) as client:
        while max_pages is None or page <= max_pages:
            for attempt in range(3):
                try:
                    resp = await client.get(url, params={"page": page, "size": page_size})
                    resp.raise_for_status()
                    data = resp.json()
                    break
                except (httpx.TimeoutException, httpx.HTTPStatusError):
                    if attempt == 2: raise
                    await asyncio.sleep(2 ** attempt)
            if not data.get("items"): break
            for item in data["items"]: yield item
            page += 1
```

---

## Задача 4. Producer-Consumer с asyncio.Queue + Semaphore

```python
import asyncio

async def worker(name: str, queue: asyncio.Queue, sem: asyncio.Semaphore):
    while True:
        item = await queue.get()
        if item is None:
            queue.task_done()
            break
        async with sem:
            print(f"[{name}] {item}")
            await asyncio.sleep(0.05)  # имитация работы
        queue.task_done()

async def main():
    queue = asyncio.Queue(maxsize=50)
    sem = asyncio.Semaphore(3)
    workers = [asyncio.create_task(worker(f"W{i}", queue, sem)) for i in range(3)]
    for i in range(20): await queue.put(f"task-{i}")
    for _ in workers: await queue.put(None)
    await asyncio.gather(*workers)
```

---

## Задача 5. Thread-safe Singleton декоратор

```python
import threading, functools

def singleton(cls):
    _instances = {}
    _lock = threading.Lock()
    @functools.wraps(cls)
    def get_instance(*args, **kwargs):
        if cls not in _instances:
            with _lock:
                if cls not in _instances:  # double-checked locking
                    _instances[cls] = cls(*args, **kwargs)
        return _instances[cls]
    return get_instance

@singleton
class Config:
    def __init__(self, path: str):
        self.path = path
        self.data = json.loads(Path(path).read_text())
```