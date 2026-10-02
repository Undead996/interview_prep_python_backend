# Подводные камни и задачи Python

---

## Подводный камень 1. Mutable default

```python
def add_user(user_list=[]):
    user_list.append("new")
    return user_list

print(add_user())  # ['new']
print(add_user())  # ['new', 'new'] — БАГ!
```

**Фикс:** `def add_user(user_list=None): if user_list is None: user_list = []`

---

## Подводный камень 2. Late binding

```python
funcs = [lambda: i for i in range(3)]
print([f() for f in funcs])  # [2, 2, 2]

funcs = [lambda i=i: i for i in range(3)]
print([f() for f in funcs])  # [0, 1, 2]
```

---

## Подводный камень 3. `is` vs `==`

```python
a = 256; b = 256
a is b  # True (кэширование -5..256)

a = 1000; b = 1000
a is b  # False!
```

---

## Задача 1. Класс для кэша с TTL

<details>
<summary>Решение</summary>

```python
import time
from typing import Any, Optional

class TTLCache:
    def __init__(self, ttl: float = 60):
        self._data: dict[str, tuple[Any, float]] = {}
        self._ttl = ttl

    def get(self, key: str) -> Optional[Any]:
        if key not in self._data:
            return None
        value, expires = self._data[key]
        if time.monotonic() > expires:
            del self._data[key]
            return None
        return value

    def set(self, key: str, value: Any):
        self._data[key] = (value, time.monotonic() + self._ttl)
```
</details>

---

## Задача 2. Rate limiter с декоратором

<details>
<summary>Решение</summary>

```python
import time
import functools

def rate_limit(max_calls: int, period: float):
    def decorator(func):
        calls: list[float] = []

        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            now = time.monotonic()
            # Отсекаем старые
            calls[:] = [c for c in calls if now - c < period]
            if len(calls) >= max_calls:
                raise RuntimeError("Rate limit exceeded")
            calls.append(now)
            return func(*args, **kwargs)
        return wrapper
    return decorator

@rate_limit(max_calls=10, period=1.0)
def api_call():
    ...
```
</details>

---

## Задача 3. Async bulk insert

<details>
<summary>Решение</summary>

```python
import asyncio
from dataclasses import dataclass, asdict

@dataclass
class User:
    name: str
    email: str

async def bulk_insert(db, users: list[User], batch_size: int = 100):
    for i in range(0, len(users), batch_size):
        batch = users[i:i+batch_size]
        values = [asdict(u) for u in batch]
        await db.execute("INSERT INTO users (name, email) VALUES ($1, $2)", values)

async def main():
    users = [User(name=f"User{i}", email=f"user{i}@test.com") for i in range(1000)]
    await bulk_insert(db_conn, users)
```
</details>

---

> **На собесе:** «Три самых частых бага в Python-коде» —
> «1) mutable default-аргументы (список создаётся один раз).
> 2) Late binding в лямбдах (замыкание на переменную цикла).
> 3) `is` вместо `==` (сравнение ID, а не значения).
> В бэкенде это превращается в баги кэша, race condition и неправильную бизнес-логику.»