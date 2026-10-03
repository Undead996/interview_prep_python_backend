# Задачи Python — расширенный разбор

---

## Задача 1. Bulk insert с async

**Условие:** Напишите функцию `bulk_insert_users`, которая вставляет 1000 пользователей батчами по 100. Используйте async + asyncpg.

```python
import asyncio
from dataclasses import dataclass, asdict

@dataclass
class User:
    name: str
    email: str

async def bulk_insert_users(pool, users: list[User], batch_size: int = 100):
    async with pool.acquire() as conn:
        for i in range(0, len(users), batch_size):
            batch = users[i:i + batch_size]
            values = [(u.name, u.email) for u in batch]
            await conn.executemany(
                "INSERT INTO users (name, email) VALUES ($1, $2)",
                values,
            )

async def main():
    pool = await asyncpg.create_pool(DATABASE_URL, min_size=5, max_size=20)
    users = [User(name=f"User{i}", email=f"user{i}@test.com") for i in range(1000)]
    await bulk_insert_users(pool, users)
    await pool.close()
```

**Почему батчи:** каждый `INSERT` — round-trip к БД. 10 батчей по 100 = 10 round-trip вместо 1000. Ускорение ~100x.

---

## Задача 2. Rate limiting декоратор

**Условие:** Декоратор `@rate_limit(max_calls=5, period=10)` — не более 5 вызовов за 10 секунд. При превышении — `RateLimitExceeded`.

```python
import time
import functools

class RateLimitExceeded(Exception):
    pass

def rate_limit(max_calls: int, period: float):
    def decorator(func):
        timestamps: list[float] = []

        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            now = time.monotonic()
            # Удаляем устаревшие
            timestamps[:] = [t for t in timestamps if now - t < period]
            if len(timestamps) >= max_calls:
                raise RateLimitExceeded()
            timestamps.append(now)
            return func(*args, **kwargs)
        return wrapper
    return decorator
```

**Почему `timestamps[:] = ...`:** чтобы изменить исходный список (в замыкании), а не создать новый.

---

## Задача 3. Async генератор для пагинации API

```python
import httpx
from typing import AsyncIterator

async def paginate(url: str, page_size: int = 100) -> AsyncIterator[dict]:
    page = 1
    async with httpx.AsyncClient() as client:
        while True:
            resp = await client.get(url, params={"page": page, "size": page_size})
            data = resp.json()
            if not data["items"]:
                break
            for item in data["items"]:
                yield item
            page += 1

async def process():
    async for user in paginate("https://api.example.com/users"):
        process_user(user)
```

**Что проверяют:** async-генератор (yield внутри async def), обработка пустой страницы (break).