# Стандартная библиотека Python — «какую проблему решает»

> **Цель:** не перечислить модули, а понять, **какую проблему** каждый решает и **когда** выбрать его вместо альтернативы. С примерами из реального бэкенда.

---

## 1. collections — структуры данных «из коробки»

### defaultdict — словарь с фабрикой по умолчанию

**Проблема:** без `defaultdict` каждый раз проверяешь, есть ли ключ:

```python
# ❌ Без defaultdict — 4 строки вместо 1
users_by_role: dict[str, list[User]] = {}
for user in users:
    if user.role not in users_by_role:
        users_by_role[user.role] = []
    users_by_role[user.role].append(user)

# ✅ С defaultdict — ключ создаётся автоматически
from collections import defaultdict
users_by_role: defaultdict[str, list[User]] = defaultdict(list)
for user in users:
    users_by_role[user.role].append(user)  # list() вызывается при первом обращении
```

**Внутреннее устройство:** `defaultdict` переопределяет `__missing__()` — метод, который dict вызывает, когда ключ не найден при `__getitem__`. `__missing__` вызывает фабрику и вставляет результат.

**⚠️ Осторожно:** `defaultdict(lambda: 0)` создаст ключ даже при простом обращении `d["nonexistent"]`. Используй `.get()` если не хочешь создавать.

**Типичные фабрики:**
| Фабрика | Для чего |
|---------|---------|
| `list` | Группировка элементов по ключу |
| `set` | Уникальные значения по ключу |
| `int` | Счётчик (аналог Counter) |
| `dict` | Вложенные словари |
| `lambda: []` | Кастомная логика |

### Counter — подсчёт с арифметикой

```python
from collections import Counter

# Подсчёт
logs = ["ERROR", "INFO", "ERROR", "WARN", "INFO", "ERROR"]
cnt = Counter(logs)  # Counter({'ERROR': 3, 'INFO': 2, 'WARN': 1})

# Топ-N
cnt.most_common(2)  # [('ERROR', 3), ('INFO', 2)]

# Арифметика (удобно для агрегации метрик)
cnt1 = Counter(a=3, b=1)
cnt2 = Counter(a=1, b=2)
cnt1 + cnt2   # Counter({'a': 4, 'b': 3})
cnt1 - cnt2   # Counter({'a': 2}) — элементы с ≤ 0 исчезают!
cnt1 & cnt2   # Counter({'a': 1, 'b': 1}) — минимумы (пересечение)
cnt1 | cnt2   # Counter({'a': 3, 'b': 2}) — максимумы (объединение)

# В бэкенде: анализ логов, частотные метрики, подсчёт тэгов
```

### deque — двухсторонняя очередь O(1)

**Почему `list.pop(0)` — медленно?** Потому что после удаления первого элемента все остальные сдвигаются — O(n). `deque.popleft()` — O(1).

```python
from collections import deque

# Очередь FIFO
queue: deque[str] = deque()
queue.append("task1")
queue.append("task2")
task = queue.popleft()  # "task1"

# Стек LIFO
stack: deque[str] = deque()
stack.append("action1")
stack.pop()  # "action1"

# Скользящее окно (maxlen — автоудаление старых)
recent = deque(maxlen=10)  # хранит только последние 10
for event in stream:
    recent.append(event)
    process_window(list(recent))

# В бэкенде: очередь задач, LRU-кэш, буфер последних событий
```

### ChainMap — многослойный конфиг без копирования

**Проблема:** настройки из разных источников (дефолты → переменные среды → аргументы CLI). Хочется, чтобы приоритет был: CLI > среда > дефолты.

```python
from collections import ChainMap
import os

defaults = {"debug": False, "port": 8080, "host": "localhost"}
env = {"port": int(os.environ.get("PORT", 5432)), "host": os.environ.get("HOST", "")}
cli = {"host": "127.0.0.1"}  # из argparse

config = ChainMap(cli, env, defaults)
print(config["debug"])  # False (из defaults)
print(config["port"])   # 5432 (из env, переопределил defaults)
print(config["host"])   # "127.0.0.1" (из cli, наивысший приоритет)

# ChainMap НЕ копирует словари — изменения идут в ПЕРВЫЙ:
config["debug"] = True  # изменил cli, defaults остался неизменным
```

---

## 2. itertools — эффективные итераторы (без загрузки в память)

### product — декартово произведение

```python
import itertools

envs = ["dev", "staging", "prod"]
dbs = ["pg", "mysql"]

# Вместо двух вложенных циклов:
for env, db in itertools.product(envs, dbs):
    print(f"Testing {env} with {db}")
# dev pg, dev mysql, staging pg, staging mysql, prod pg, prod mysql
```

### chain — объединение итераторов без копирования

```python
# Вместо: all_items = page1 + page2 + page3 (создаёт новый список!)
all_pages = itertools.chain(page1, page2, page3)  # итератор, не копирует
# Для вложенных: itertools.chain.from_iterable(list_of_lists)
```

### groupby — группировка (⚠️ нужна сортировка!)

```python
data = [
    {"date": "2024-01-01", "amount": 100},
    {"date": "2024-01-01", "amount": 200},
    {"date": "2024-01-02", "amount": 50},
]

# ❗ ОБЯЗАТЕЛЬНО сортировать перед groupby!
sorted_data = sorted(data, key=lambda x: x["date"])
for date, group in itertools.groupby(sorted_data, key=lambda x: x["date"]):
    total = sum(item["amount"] for item in group)
    print(f"{date}: {total}")
# 2024-01-01: 300
# 2024-01-02: 50
```

### batched (Python 3.12+) — разбивка на батчи

```python
# До 3.12: [items[i : i + batch_size] for i in range(0, len(items), batch_size)]
# Python 3.12+:
for batch in itertools.batched(users, n=100):
    await bulk_insert(batch)
```

### cycle — бесконечный round-robin

```python
workers_cycle = itertools.cycle(["worker1", "worker2", "worker3"])
for task in tasks:
    assign(task, next(workers_cycle))  # worker1 → worker2 → worker3 → worker1 → ...
```

---

## 3. functools — функциональные утилиты

### lru_cache — кэш с вытеснением

```python
import functools

@functools.lru_cache(maxsize=256)
def get_user_permissions(user_id: int) -> frozenset[str]:
    """Дорогой запрос к БД. Результат кэшируется."""
    return frozenset(db.fetch_permissions(user_id))

# Статистика:
print(get_user_permissions.cache_info())
# CacheInfo(hits=50, misses=10, maxsize=256, currsize=10)

# Очистка:
get_user_permissions.cache_clear()

# ⚠️ Осторожно: НЕЛЬЗЯ с async-функциями!
# lru_cache — thread-safe для sync, но НЕ для async (нет await-блокировки)
# Для async используй @cache (3.9+) с asyncio.Lock внутри
```

### partial — фиксация аргументов

```python
from functools import partial

def fetch(url: str, timeout: float, retries: int, headers: dict | None = None):
    ...

# Фиксируем часть аргументов:
fetch_fast = partial(fetch, timeout=5.0, retries=3)
fetch_with_auth = partial(fetch, headers={"Authorization": "Bearer xxx"})

# В бэкенде: преконфигурация клиентов, callback-и для очередей
process_order = partial(handle_order, db=session, metrics=metrics_collector)
```

### singledispatch — перегрузка по типу первого аргумента

```python
from functools import singledispatch
import json
from datetime import datetime
from decimal import Decimal

@singledispatch
def serialize(obj) -> str:
    raise NotImplementedError(f"No serializer for {type(obj)}")

@serialize.register(dict)
def _(obj: dict) -> str:
    return json.dumps(obj)

@serialize.register(datetime)
def _(obj: datetime) -> str:
    return obj.isoformat()

@serialize.register(Decimal)
def _(obj: Decimal) -> str:
    return str(obj)

@serialize.register(bytes)
def _(obj: bytes) -> str:
    return obj.decode("utf-8", errors="replace")
```

---

## 4. dataclasses + enum

### dataclass — DTO без boilerplate

```python
from dataclasses import dataclass, field, asdict, astuple
from datetime import datetime, timezone

@dataclass(order=True, frozen=True)
class Address:
    street: str
    city: str
    zip_code: str = field(compare=False)

@dataclass
class UserDTO:
    id: int
    name: str
    address: Address
    tags: list[str] = field(default_factory=list, repr=False)
    created_at: datetime = field(default_factory=lambda: datetime.now(timezone.utc))

    def __post_init__(self):
        if not self.name.strip():
            raise ValueError("Name is required")

# Сериализация:
dto = UserDTO(id=1, name="Alice", address=Address("Main St", "NYC", "10001"))
print(asdict(dto))
# {'id': 1, 'name': 'Alice', 'address': {'street': 'Main St', 'city': 'NYC', 'zip_code': '10001'}, ...}
```

### Enum — именованные константы

```python
from enum import Enum, auto, StrEnum, IntEnum, Flag

class OrderStatus(StrEnum):  # Python 3.11+
    PENDING = "pending"
    CONFIRMED = "confirmed"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

# StrEnum сравнивается со строками:
assert OrderStatus.PENDING == "pending"
assert OrderStatus.PENDING in {"pending", "confirmed"}

class Priority(IntEnum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3
    CRITICAL = 4

# IntEnum сравнивается с int:
assert Priority.HIGH == 3
assert Priority.HIGH > Priority.LOW

class Permission(Flag):
    READ = 1
    WRITE = 2
    DELETE = 4
    ADMIN = 8

admin = Permission.READ | Permission.WRITE | Permission.DELETE
print(bool(admin & Permission.WRITE))  # True
```

---

## 5. pathlib — современный путь к файлам

```python
from pathlib import Path

# Вместо os.path.join:
BASE = Path(__file__).parent.parent
DATA = BASE / "data" / "fixtures"  # оператор / — кроссплатформенный!

# Чтение/запись:
config = (DATA / "config.json").read_text(encoding="utf-8")
(DATA / "output.json").write_text(json.dumps(data), encoding="utf-8")

# Glob:
for py_file in BASE.glob("**/*.py"):  # рекурсивно!
    relative = py_file.relative_to(BASE)

# Информация:
path = Path("/some/file.txt")
print(path.name)       # "file.txt"
print(path.stem)       # "file" (без расширения)
print(path.suffix)     # ".txt"
print(path.parent)     # Path("/some")
print(path.exists())   # bool
print(path.stat().st_size)  # размер в байтах

# Временные файлы:
import tempfile
with tempfile.TemporaryDirectory() as tmp:
    p = Path(tmp) / "test.txt"
    p.write_text("data")
```

**Почему pathlib, а не os.path?**
- Кроссплатформенный (`/` работает везде)
- Читаемые цепочки: `BASE / "data" / "fixtures" / "config.json"`
- Единый интерфейс для файлов и директорий
- `.read_text()` / `.write_text()` без `open()` + `close()`

---

## 6. Другие модули, которые спрашивают

### `typing` (см. подробно в language-deep)

```python
from typing import Protocol, TypeAlias, Literal, Final, TypedDict, Generic, TypeVar
```

### `abc` — абстрактные базовые классы

```python
from abc import ABC, abstractmethod

class Repository(ABC):
    @abstractmethod
    async def get_by_id(self, id: int) -> Any: ...
    @abstractmethod
    async def save(self, entity: Any) -> Any: ...
```

### `contextlib` — утилиты для контекстных менеджеров

```python
from contextlib import contextmanager, asynccontextmanager, suppress, redirect_stdout

with suppress(FileNotFoundError):
    os.remove("temp.txt")  # FileNotFoundError подавлен

with redirect_stdout(io.StringIO()) as buf:
    print("captured")
    output = buf.getvalue()
```

### `logging` — не print!

```python
import logging

logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)

# Structured logging (для прода):
logger.info("Order created", extra={"order_id": 42, "user_id": 7})

# ⚠️ Никогда не используй print() в продакшене!
```

### `uuid` — генерация уникальных ID

```python
import uuid

uid = uuid.uuid4()  # случайный UUID v4
uid_str = str(uid)  # "550e8400-e29b-41d4-a716-446655440000"
uid_hex = uid.hex   # без дефисов
```

---

> **На собесе:** «Какие модули из stdlib используете?» —
> «`collections` (defaultdict, Counter, deque) для структур данных без внешних зависимостей.
> `itertools` для эффективной работы с итераторами (product, chain, batched).
> `dataclasses` для внутренних DTO, `enum` для именованных констант.
> `pathlib` для файловых операций — кроссплатформенный и читаемый.
> `functools.lru_cache` для кэширования (но не для async!).
> Стараюсь не тянуть зависимости, когда stdlib решает задачу.»