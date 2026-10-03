# Live-coding задачи — расширенный разбор (5 задач)

---

## Задача 1. Async-генератор пагинации с retry

**Условие:** пагинирует API, с обработкой ошибок и защитой от бесконечного цикла.

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

**Что проверяют:** async-генераторы, обработка ошибок, краевые случаи (пустая страница).

---

## Задача 2. Rate limiter декоратор с состоянием

**Условие:** `@rate_limit(max_calls=5, period=10)` — не более 5 вызовов за 10 секунд.

```python
import time, functools

class RateLimitExceeded(Exception): ...

def rate_limit(max_calls: int, period: float):
    def decorator(func):
        timestamps: list[float] = []
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            now = time.monotonic()
            timestamps[:] = [t for t in timestamps if now - t < period]
            if len(timestamps) >= max_calls:
                retry = period - (now - timestamps[0])
                raise RateLimitExceeded(f"Retry after {retry:.1f}s")
            timestamps.append(now)
            return func(*args, **kwargs)
        return wrapper
    return decorator
```

---

## Задача 3. SQL — департамент с максимальной средней зарплатой

```sql
SELECT d.name, ROUND(AVG(e.salary), 2) AS avg_salary
FROM department d
JOIN employee e ON e.dept_id = d.id
GROUP BY d.id, d.name
ORDER BY avg_salary DESC
LIMIT 1;
```

**Что проверяют:** JOIN, GROUP BY, агрегация, сортировка + LIMIT. Если несколько департаментов с одинаковой макс. — добавить `RANK() OVER`.

---

## Задача 4. RabbitMQ consumer с retry и DLX

```python
MAX_RETRIES = 3

async def callback(message: aio_pika.IncomingMessage):
    async with message.process():
        try:
            retry_count = int(message.headers.get("x-retry-count", 0))
            await process(message.body)
        except TemporaryError:
            if retry_count < MAX_RETRIES:
                # Перепубликуем с увеличенным retry count
                await message.channel.default_exchange.publish(
                    aio_pika.Message(
                        body=message.body,
                        headers={"x-retry-count": retry_count + 1},
                        expiration=str(2 ** retry_count * 1000),
                    ),
                    routing_key="orders.retry",
                )
                await message.ack()
            else:
                await message.reject(requeue=False)  # → DLX
        except PermanentError:
            await message.reject(requeue=False)  # → DLX сразу
```

---

## Задача 5. Cache-aside endpoint с Redis

```python
@app.get("/api/users/{user_id}")
async def get_user(user_id: int, request: Request):
    redis = request.app.state.redis
    cache_key = f"user:{user_id}"

    if cached := await redis.get(cache_key):
        return User.model_validate_json(cached)

    user = await db.get(User, user_id)
    if not user: raise HTTPException(404)

    await redis.setex(cache_key, 3600, user.model_dump_json())
    return user
```

---

## Советы для live-coding

1. **Читай задачу вслух** — уточни неясное, покажи что мыслишь
2. **Думай вслух** — «я могу использовать async-генератор / defaultdict / оконную функцию»
3. **Пиши код слоями** — сначала простое решение, потом улучшения:
   - «работает? ок, добавляю retry»
   - «теперь обработку ошибок»
   - «теперь тест на краевой случай»
4. **Проверяй краевые случаи** — пустой ответ, нулевые значения, дубликаты
5. **Если не помнишь синтаксис** — скажи «точно не помню, но идея такая», объясни подход
6. **Пиши читаемый код** — осмысленные имена переменных, никаких single-letter кроме i/j
7. **Упоминай trade-off** — «тут можно было бы через join, но EXISTS быстрее потому что...»