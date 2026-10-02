# Live-coding задачи

---

## Задача 1. Функция пагинации

**Условие:** Напишите async-генератор, который пагинирует API `/api/users?page=N`.

<details>
<summary>Решение</summary>

```python
import httpx
from typing import AsyncIterator

async def get_all_users(base_url: str) -> AsyncIterator[dict]:
    page = 1
    async with httpx.AsyncClient() as client:
        while True:
            resp = await client.get(f"{base_url}/api/users", params={"page": page, "size": 100})
            data = resp.json()
            if not data["items"]:
                break
            for user in data["items"]:
                yield user
            page += 1
```
</details>

---

## Задача 2. RabbitMQ consumer + retry

**Условие:** Напишите consumer, который при временной ошибке возвращает сообщение
в очередь, при постоянной — в DLX.

<details>
<summary>Решение</summary>

```python
async def callback(channel, message: aio_pika.IncomingMessage):
    async with message.process():
        try:
            await process(message.body)
        except TemporaryError:
            # сообщение вернётся в очередь (requeue=True)
            raise  # ack не будет → requeue
        except PermanentError:
            await message.reject(requeue=False)  # → DLX
```
</details>

---

## Задача 3. SQL — департамент с макс. средней зарплатой

**Условие:** `department(id, name)`, `employee(id, name, salary, dept_id)`.
Найти департамент с макс. средней зарплатой.

<details>
<summary>Решение</summary>

```sql
SELECT d.name, AVG(e.salary) AS avg_salary
FROM department d
JOIN employee e ON e.dept_id = d.id
GROUP BY d.id, d.name
ORDER BY avg_salary DESC
LIMIT 1;
```
</details>

---

> **На собесе:** Live-coding — не «решить любой ценой», а показать ход мыслей.
> Читай задачу вслух → уточни → подумай → пиши код → проверь граничные случаи.
> Если не помнишь синтаксис — скажи «точно не помню, но идея такая».