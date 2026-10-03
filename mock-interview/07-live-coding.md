# Live-coding задачи — расширенный разбор

---

## Задача 1. Async-генератор пагинации

**Условие:** Напишите async-генератор, который пагинирует API `/api/users?page=N`.

**Что проверяют:** понимание async-генераторов, обработка краевых случаев (пустой ответ).

```python
import httpx
from typing import AsyncIterator

async def paginate(base_url: str, page_size: int = 100) -> AsyncIterator[dict]:
    page = 1
    async with httpx.AsyncClient() as client:
        while True:
            resp = await client.get(
                f"{base_url}/api/users",
                params={"page": page, "size": page_size}
            )
            data = resp.json()
            if not data["items"]:  # пустая страница → конец
                break
            for user in data["items"]:
                yield user
            page += 1
```

**Как улучшить:**
- Добавить `max_pages` — защита от бесконечного цикла
- `resp.raise_for_status()` — обработка ошибок HTTP
- `retry` при таймауте

---

## Задача 2. RabbitMQ consumer + retry

**Условие:** Consumer, который при временной ошибке возвращает сообщение в очередь, при постоянной — в DLX.

```python
async def callback(message: aio_pika.IncomingMessage):
    async with message.process():
        try:
            await process(message.body)
        except TemporaryError:
            # Если не ack — сообщение вернётся в очередь (requeue)
            raise
        except PermanentError:
            await message.reject(requeue=False)  # → DLX
```

**Что проверяют:** понимание ack/nack/reject, DLX, временные vs постоянные ошибки.

---

## Задача 3. SQL — департамент с макс. средней зарплатой

```sql
SELECT d.name, AVG(e.salary) AS avg_salary
FROM department d
JOIN employee e ON e.dept_id = d.id
GROUP BY d.id, d.name
ORDER BY avg_salary DESC
LIMIT 1;
```

**Что проверяют:** JOIN, GROUP BY, агрегация, ORDER BY + LIMIT.

---

## Советы для live-coding

1. **Читай задачу вслух** — уточни неясное
2. **Подумай вслух** — "я могу использовать async-генератор"
3. **Пиши код** — начни с простого решения, потом улучшай
4. **Проверь краевые случаи** — пустая страница, отсутствие результата
5. **Если не помнишь синтаксис** — скажи "точно не помню, но идея такая"