# Подводные камни Python и задачи с разбором

> **Цель:** не просто «вот баг, вот фикс», а **почему** это происходит на уровне языка и **как** не допускать в продакшене.

---

## Топ-10 подводных камней

### 1. Mutable default-аргументы

**Почему:** дефолтные значения вычисляются **один раз** при определении функции (compile-time), а не при каждом вызове.

```python
# ❌ Баг — один список на все вызовы
def add_user(user, users=[]):
    users.append(user)
    return users

print(add_user("Alice"))  # ['Alice']
print(add_user("Bob"))    # ['Alice', 'Bob'] ← БАГ!

# Доказательство:
print(add_user.__defaults__)  # (['Alice', 'Bob'],)

# ✅ Фикс
def add_user(user, users=None):
    if users is None:
        users = []
    users.append(user)
    return users
```

**Где кусает в бэкенде:** кэши, накопление результатов, состояние между вызовами.

### 2. Late binding в замыканиях (lambda/closure)

**Почему:** лямбда (или вложенная функция) захватывает **ссылку** на переменную, а не её значение на момент создания.

```python
# ❌ Баг — все лямбды возвращают последнее значение i
funcs = [lambda: i for i in range(3)]
print([f() for f in funcs])  # [2, 2, 2] ← БАГ!

# ✅ Фикс: зафиксировать как дефолтный аргумент
funcs = [lambda i=i: i for i in range(3)]
print([f() for f in funcs])  # [0, 1, 2] ✅

# ✅ Фикс: функция-фабрика
def make_func(val):
    return lambda: val
funcs = [make_func(i) for i in range(3)]
```

### 3. `is` vs `==`

**Правило:** `is` проверяет **идентичность** (один и тот же объект в памяти?), `==` проверяет **равенство** значений.

```python
# Кэш маленьких int (CPython: -5..256)
a = 256; b = 256
print(a is b)   # True (кэш!)

a = 1000; b = 1000
print(a is b)   # False (разные объекты)

# None, True, False — синглтоны
x = None
print(x is None)  # ✅ всегда правильно
print(x == None)  # ⚠️ технически работает, но неправильно

# Intern-строки
a = "hello"; b = "hello"
print(a is b)  # True (короткие строки без пробелов intern-ируются)

a = "hello world!"; b = "hello world!"
print(a is b)  # False (с пробелом/длинные — нет)
```

### 4. Изменение списка при итерации

```python
# ❌ Баг — пропуск элементов!
items = [1, 2, 3, 4, 5]
for i in items:
    if i % 2 == 0:
        items.remove(i)  # сдвиг индексов!
print(items)  # [1, 3, 5]? Нет — зависит от версии Python

# ✅ Фильтрация — новый список
items = [i for i in items if i % 2 != 0]

# ✅ Или итерация по копии
for i in items[:]:
    if i % 2 == 0:
        items.remove(i)
```

### 5. Циклический import

```python
# module_a.py:
from module_b import func_b
def func_a(): return func_b()

# module_b.py:
from module_a import func_a
def func_b(): return func_a()

# ❌ ImportError или AttributeError!

# ✅ Фикс: импорт внутри функции или импорт модуля целиком
import module_b
def func_a(): return module_b.func_b()
```

### 6. Изменяемый ключ dict

```python
d = {[1, 2]: "value"}  # ❌ TypeError: unhashable type: 'list'

class User:
    def __hash__(self): return hash(self.id)

u = User(); u.id = 1
d = {u: "data"}
u.id = 2  # хэш изменился!
print(d[u])  # ❌ KeyError — хэш изменился, объект не найден!
```

### 7. Копирование — shallow vs deep

```python
import copy

original = [{"name": "Alice"}, {"name": "Bob"}]

# Shallow — копирует список, но dict-ы внутри — те же объекты!
shallow = original.copy()
shallow[0]["name"] = "Eve"
print(original[0]["name"])  # "Eve" — изменился!

# Deep — копирует всё дерево объектов
deep = copy.deepcopy(original)
deep[0]["name"] = "Mallory"
print(original[0]["name"])  # "Eve" — не изменился ✅
```

### 8. try/except без конкретного исключения

```python
# ❌ Опасно — ловит ВСЁ включая KeyboardInterrupt и SystemExit!
try:
    result = risky_operation()
except:
    result = default_value

# ✅ Ловим только ожидаемое
try:
    result = risky_operation()
except (ValueError, ConnectionError) as e:
    logger.error("Operation failed", exc_info=e)
    result = default_value
```

### 9. `+=` на tuple (изменяет, если внутри mutable)

```python
t = (1, 2, [3, 4])
try:
    t[2] += [5]  # ❌ TypeError, НО список ИЗМЕНИЛСЯ!
except TypeError:
    pass
print(t)  # (1, 2, [3, 4, 5]) — баг: исключение было, а изменение — тоже было!
```

### 10. `defaultdict` создаёт ключ при обращении

```python
from collections import defaultdict

d = defaultdict(int)
print(d["nonexistent"])  # 0 (создал ключ!)
print("nonexistent" in d)  # True — ключ появился!

# ⚠️ Если используешь defaultdict, не проверяй if key in d — используй .get()
```

---

## Задачи с разбором

### Задача 1. TTL Cache с потокобезопасностью

```python
import time
import threading
from typing import Any, Generic, TypeVar

T = TypeVar("T")

class TTLCache(Generic[T]):
    """Кэш с временем жизни. Потокобезопасный (sync)."""

    def __init__(self, ttl: float = 60.0):
        self._data: dict[str, tuple[T, float]] = {}
        self._ttl = ttl
        self._lock = threading.Lock()

    def get(self, key: str) -> T | None:
        with self._lock:
            if key not in self._data:
                return None
            value, expires = self._data[key]
            if time.monotonic() > expires:
                del self._data[key]
                return None
            return value

    def set(self, key: str, value: T, ttl: float | None = None) -> None:
        with self._lock:
            self._data[key] = (value, time.monotonic() + (ttl or self._ttl))

    def cleanup(self) -> int:
        """Принудительная очистка. Возвращает количество удалённых."""
        with self._lock:
            now = time.monotonic()
            expired = [k for k, (_, exp) in self._data.items() if now > exp]
            for k in expired:
                del self._data[k]
            return len(expired)

    def __len__(self) -> int:
        return len(self._data)
```

**Почему `time.monotonic()`?** `time.time()` может прыгнуть назад (NTP-корректировка). `monotonic()` только растёт.

### Задача 2. Rate limiter-декоратор

```python
import time
import functools

class RateLimitExceeded(Exception):
    def __init__(self, retry_after: float = 0):
        self.retry_after = retry_after
        super().__init__(f"Rate limit exceeded. Retry after {retry_after:.1f}s")

def rate_limit(max_calls: int, period: float):
    def decorator(func):
        timestamps: list[float] = []

        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            now = time.monotonic()
            # Удаляем устаревшие (изменяем список на месте!)
            timestamps[:] = [t for t in timestamps if now - t < period]

            if len(timestamps) >= max_calls:
                retry_after = period - (now - timestamps[0]) if timestamps else period
                raise RateLimitExceeded(retry_after)

            timestamps.append(now)
            return func(*args, **kwargs)

        return wrapper
    return decorator

# Использование:
@rate_limit(max_calls=10, period=60)
def api_call(data: dict) -> dict:
    return {"status": "ok", "data": data}
```

**Почему `timestamps[:] = ...` а не `timestamps = ...`?** Потому что `timestamps = [...]` создаст новую локальную переменную, а исходный список в замыкании не изменится. `timestamps[:] = ...` модифицирует исходный список на месте.

### Задача 3. Async bulk insert с транзакцией

```python
import asyncio
from dataclasses import dataclass

@dataclass
class User:
    name: str
    email: str

async def bulk_insert_users(
    conn,
    users: list[User],
    batch_size: int = 100,
) -> int:
    """Вставка батчами в одной транзакции. Возвращает количество вставленных."""
    total = 0
    async with conn.transaction():
        for i in range(0, len(users), batch_size):
            batch = users[i : i + batch_size]
            values = [(u.name, u.email) for u in batch]
            result = await conn.executemany(
                "INSERT INTO users (name, email) VALUES ($1, $2) "
                "ON CONFLICT (email) DO NOTHING",
                values,
            )
            # executemany в asyncpg возвращает строку с количеством
            total += len(batch)
        return total

# Использование с asyncpg pool:
async def main():
    pool = await asyncpg.create_pool(DATABASE_URL, min_size=5, max_size=20)
    users = [User(name=f"User{i}", email=f"user{i}@test.com") for i in range(1000)]
    async with pool.acquire() as conn:
        inserted = await bulk_insert_users(conn, users)
    print(f"Inserted {inserted} users")
    await pool.close()
```

### Задача 4. Async-генератор пагинации с retry

```python
import httpx
from typing import AsyncIterator

async def paginate(
    base_url: str,
    page_size: int = 100,
    max_pages: int | None = None,
) -> AsyncIterator[dict]:
    """Async-генератор, пагинирующий API. С защитой от бесконечного цикла."""
    page = 1
    async with httpx.AsyncClient(timeout=30.0) as client:
        while max_pages is None or page <= max_pages:
            for attempt in range(3):
                try:
                    resp = await client.get(
                        f"{base_url}/api/users",
                        params={"page": page, "size": page_size},
                    )
                    resp.raise_for_status()
                    data = resp.json()
                    break
                except (httpx.TimeoutException, httpx.HTTPStatusError):
                    if attempt == 2:
                        raise
                    await asyncio.sleep(2 ** attempt)

            if not data.get("items"):
                break
            for item in data["items"]:
                yield item
            page += 1

# Использование:
async def process_all():
    async for user in paginate("https://api.example.com"):
        await process_user(user)
```

### Задача 5. Потокобезопасный singleton-декоратор

```python
import threading
import functools
from typing import Any

def singleton(cls):
    """Декоратор, превращающий класс в потокобезопасный Singleton."""
    _instances: dict[type, Any] = {}
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
class Database:
    def __init__(self, url: str):
        self.url = url

db1 = Database("postgres://...")
db2 = Database("different-url")  # игнорируется!
print(db1 is db2)       # True
print(db1.url)          # "postgres://..."
```

---

> **На собесе:** «Три самых частых бага в Python?» —
> «1) Mutable default-аргументы: список/словарь создаётся один раз при определении функции и живёт в `__defaults__`.
> 2) Late binding в лямбдах/замыканиях: захватывается ссылка на переменную, а не её значение на момент создания.
> 3) `is` вместо `==`: проверка идентичности, а не равенства — кэш int-ов и intern-строк создаёт иллюзию, что это работает всегда.»