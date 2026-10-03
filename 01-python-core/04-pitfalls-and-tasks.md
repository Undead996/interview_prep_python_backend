# Подводные камни и задачи Python — с глубоким разбором

> **Цель:** не просто "вот баг, вот фикс", а **почему** это происходит на уровне
> языка и **как** не допускать в продакшене.

---

## Подводный камень 1. Mutable default аргументы

### Почему это происходит

**Коротко:** дефолтные значения вычисляются **один раз** при определении функции, а не каждый раз при вызове.

```python
def add_user(user_list=[]):      # [] создаётся ОДИН раз!
    user_list.append("new")
    return user_list

print(add_user())  # ['new']
print(add_user())  # ['new', 'new'] — баг!
```

Под капотом: Python хранит дефолтные значения в `func.__defaults__`:

```python
print(add_user.__defaults__)  # (['new', 'new'],)
```

Каждый вызов `add_user()` получает **тот же самый список** из `__defaults__`.

### Где это больно в бэкенде

```python
# ❌ Опасный кэш
def get_user(user_id: int, cache={}):
    if user_id in cache:
        return cache[user_id]
    user = fetch_from_db(user_id)
    cache[user_id] = user  # mutable default!
    return user

# ✅ Правильно
def get_user(user_id: int, cache=None):
    if cache is None:
        cache = {}
    ...
```

**Правило:** mutable default — только `None`, внутри функции — проверка и создание.

---

## Подводный камень 2. Late binding в лямбдах

### Почему это происходит

```python
funcs = [lambda: i for i in range(3)]
print([f() for f in funcs])  # [2, 2, 2], а не [0, 1, 2]!
```

**Почему:** лямбда захватывает **ссылку** на переменную `i`, а не её значение. К моменту, когда лямбда выполняется, `i` уже равна 2 (последнее значение цикла).

**Замыкание:** `lambda: i` — это "анонимная функция, которая возвращает значение `i` из окружающей области видимости". `i` — не константа, а переменная, которая живёт и меняется.

### Фикс

```python
# Вариант 1: зафиксировать значение как дефолт
funcs = [lambda i=i: i for i in range(3)]
# Теперь каждая лямбда имеет свой i=0, i=1, i=2 как default

# Вариант 2: functools.partial
from functools import partial
def get_i(i): return i
funcs = [partial(get_i, i) for i in range(3)]

# Вариант 3: list comprehension с обычной функцией
def make_lambda(val):
    return lambda: val
funcs = [make_lambda(i) for i in range(3)]
```

---

## Подводный камень 3. `is` vs `==`

**Простыми словами:** `==` проверяет **значение**, `is` проверяет **идентичность** (это один и тот же объект в памяти?).

```python
# Кэширование маленьких int
a = 256
b = 256
a is b  # True — CPython кэширует int от -5 до 256

a = 1000
b = 1000
a is b  # False — разные объекты!

# None — синглтон
x = None
x is None  # ✅ всегда правильно
x == None  # ❌ технически работает, но неправильно
```

**Правило:** `is` — только для сравнения с `None`, `True`, `False`. Для всего остального — `==`.

---

## Задача 1. TTL Cache — класс

```python
import time
from typing import Any, Optional

class TTLCache:
    """Кэш с временем жизни. Потокобезопасный (для sync-кода)."""
    
    def __init__(self, ttl: float = 60):
        self._data: dict[str, tuple[Any, float]] = {}
        self._ttl = ttl

    def get(self, key: str) -> Optional[Any]:
        if key not in self._data:
            return None
        value, expires = self._data[key]
        if time.monotonic() > expires:
            del self._data[key]  # ленивая очистка
            return None
        return value

    def set(self, key: str, value: Any):
        self._data[key] = (value, time.monotonic() + self._ttl)

    def cleanup(self):
        """Принудительная очистка просроченных ключей."""
        now = time.monotonic()
        expired = [k for k, (_, exp) in self._data.items() if now > exp]
        for k in expired:
            del self._data[k]
```

**Почему `time.monotonic()`?** Потому что `time.time()` может "прыгать" назад (NTP, ручная корректировка). `monotonic()` — всегда растёт.

---

## Задача 2. Rate limiter-декоратор

```python
import time
import functools

class RateLimitExceeded(Exception):
    pass

def rate_limit(max_calls: int, period: float):
    def decorator(func):
        calls: list[float] = []

        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            now = time.monotonic()
            # Удаляем устаревшие вызовы
            calls[:] = [c for c in calls if now - c < period]
            if len(calls) >= max_calls:
                raise RateLimitExceeded()
            calls.append(now)
            return func(*args, **kwargs)
        return wrapper
    return decorator
```

**Почему `calls[:] = ...`?** Потому что `calls = ...` создаст новую локальную переменную, а исходный список не изменится. `calls[:] = ...` модифицирует исходный список (который хранится в замыкании).

---

## Задача 3. Async bulk insert

```python
import asyncio
from dataclasses import dataclass, asdict

@dataclass
class User:
    name: str
    email: str

async def bulk_insert(conn, users: list[User], batch_size: int = 100):
    """Вставка батчами — меньше запросов к БД."""
    for i in range(0, len(users), batch_size):
        batch = users[i:i + batch_size]
        values = [(u.name, u.email) for u in batch]
        await conn.executemany(
            "INSERT INTO users (name, email) VALUES ($1, $2)",
            values,
        )
```

**Почему батчи?** Каждый `execute()` — round-trip к БД. Если у нас 1000 строк, 10 батчей по 100 — 10 round-trip вместо 1000. Ускорение в 10–50x.

---

> **На собесе:** "Три самых частых бага в Python-коде" —  
> "1) mutable default-аргументы (список создаётся один раз при определении функции).  
> 2) Late binding в лямбдах (замыкание на переменную цикла, а не её значение).  
> 3) `is` вместо `==` для сравнения значений (проверка идентичности, а не равенства).  
> В бэкенде это превращается в баги кэша, race condition и неправильную бизнес-логику."