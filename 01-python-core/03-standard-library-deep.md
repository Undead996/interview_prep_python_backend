# Стандартная библиотека Python — глубокий разбор

> **Цель:** не просто перечислить модули, а понять, **какую проблему** каждый решает
> и **когда** его выбрать вместо аналогичного решения.

---

## 1. collections — структуры данных "из коробки"

### defaultdict — словарь с "фабрикой по умолчанию"

**Проблема:** без `defaultdict` каждый раз нужно проверять, есть ли ключ:

```python
# ❌ Без defaultdict
users_by_role = {}
for user in users:
    if user.role not in users_by_role:
        users_by_role[user.role] = []
    users_by_role[user.role].append(user)

# ✅ С defaultdict
from collections import defaultdict
users_by_role = defaultdict(list)
for user in users:
    users_by_role[user.role].append(user)  # ключ создаётся автоматически!
```

**Важно:** `defaultdict(lambda: None)` создаёт ключи **при любом обращении** — даже при проверке `if key in d` (нет) или `d[key]` (да, создаст). Будьте осторожны — можете нечаянно наплодить ключей.

### Counter — подсчёт элементов

```python
from collections import Counter

logs = ["ERROR", "INFO", "ERROR", "WARN", "INFO", "ERROR"]

cnt = Counter(logs)
# cnt = {'ERROR': 3, 'INFO': 2, 'WARN': 1}

# Топ-N самых частых
cnt.most_common(2)  # [('ERROR', 3), ('INFO', 2)]

# Арифметика
cnt1 = Counter(a=3, b=1)
cnt2 = Counter(a=1, b=2)
cnt1 + cnt2  # Counter({'a': 4, 'b': 3})
cnt1 - cnt2  # Counter({'a': 2})  (исчезают, если <= 0)
```

**Когда применяется:** анализ логов, подсчёт тэгов, любые частотные метрики.

### deque — быстрая очередь/стек

**Почему `list.pop(0)` — медленно?** Потому что после удаления первого элемента все остальные сдвигаются на одну позицию — O(n). `deque.popleft()` — O(1).

```python
from collections import deque

# Очередь FIFO
queue = deque()
queue.append("task1")
queue.append("task2")
task = queue.popleft()  # "task1"

# Стек LIFO
stack = deque()
stack.append("action1")
stack.append("action2")
action = stack.pop()  # "action2"

# Скользящее окно
recent = deque(maxlen=10)  # хранит последние 10 элементов
for event in events:
    recent.append(event)
    process_window(list(recent))
```

### ChainMap — layered config

**Проблема:** у вас есть настройки из разных источников (дефолты → переменные среды → аргументы командной строки). Хочется, чтобы приоритет был: **аргументы > среда > дефолты**.

```python
from collections import ChainMap

defaults = {"debug": False, "port": 8080, "host": "localhost"}
env = {"port": 5432, "host": "db.example.com"}  # из os.environ
cli = {"host": "127.0.0.1"}  # из argparse

config = ChainMap(cli, env, defaults)
# config["debug"] → False (из defaults, не переопределён)
# config["port"] → 5432 (из env, переопределяет defaults)
# config["host"] → "127.0.0.1" (из cli, наивысший приоритет)

# ChainMap НЕ копирует данные — все изменения идут в ПЕРВЫЙ словарь:
config["debug"] = True  # изменит cli!
```

---

## 2. itertools — эффективные итераторы (без загрузки в память)

### product — декартово произведение

```python
import itertools

envs = ["dev", "staging", "prod"]
dbs = ["pg", "mysql"]

for env, db in itertools.product(envs, dbs):
    print(f"Testing {env} with {db}")
# dev pg, dev mysql, staging pg, staging mysql, prod pg, prod mysql
```

**Без product:** два вложенных цикла. С product — один цикл, читаемее.

### cycle — бесконечный round-robin

```python
workers = itertools.cycle(["worker1", "worker2", "worker3"])
for task in tasks:
    worker = next(workers)
    assign(task, worker)
```

### groupby — группировка (сортировка обязательна!)

```python
data = [
    {"date": "2024-01-01", "amount": 100},
    {"date": "2024-01-01", "amount": 200},
    {"date": "2024-01-02", "amount": 50},
]

# groupby работает ТОЛЬКО на отсортированных данных!
sorted_data = sorted(data, key=lambda x: x["date"])
for date, group in itertools.groupby(sorted_data, key=lambda x: x["date"]):
    total = sum(item["amount"] for item in group)
    print(f"{date}: {total}")
```

**Важно:** `groupby` создаёт группы на лету — если данные не отсортированы, один и тот же ключ может появиться в нескольких группах.

### chain — объединение итераторов

```python
page1 = [1, 2, 3]
page2 = [4, 5, 6]
page3 = [7, 8, 9]

all_pages = list(itertools.chain(page1, page2, page3))
# [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

### batched (Python 3.12+) — разбить на батчи

```python
batch_size = 100
for batch in itertools.batched(all_users, n=batch_size):
    await bulk_insert(batch)
```

**Раньше (до 3.12):** вручную через range(0, len, batch_size). Теперь — встроено.

---

## 3. typing — строгая типизация для больших проектов

### Protocol — утиная типизация со статической проверкой

```python
from typing import Protocol

class SupportsRead(Protocol):
    def read(self) -> bytes: ...

def process(stream: SupportsRead):
    data = stream.read()
    ...

# Любой объект с методом read() подойдёт
class File:
    def read(self) -> bytes: return b"data"

class Socket:
    def read(self) -> bytes: return b"response"

process(File())    # ✅
process(Socket())  # ✅
```

**В отличие от ABC (Abstract Base Class):** не требует наследования. Проверка — на уровне mypy/pyright, не на уровне runtime.

### TypeAlias — читаемость сложных типов

```python
from typing import TypeAlias

JSON: TypeAlias = dict[str, "JSON"] | list["JSON"] | str | int | float | bool | None

def process(data: JSON) -> None: ...
```

### Generic — переиспользуемый код с типами

```python
from typing import TypeVar, Generic

T = TypeVar("T")

class Repository(Generic[T]):
    async def get_by_id(self, id: int) -> T | None: ...
    async def save(self, entity: T) -> T: ...

class UserRepo(Repository[User]):
    async def get_by_email(self, email: str) -> User | None: ...
```

---

## 4. dataclasses + enum

### dataclass — когда нужен DTO без boilerplate

```python
from dataclasses import dataclass, field, asdict

@dataclass(order=True, frozen=True)
class Address:
    street: str
    city: str
    zip_code: str = field(compare=False)

@dataclass
class User:
    id: int
    name: str
    address: Address
    tags: list[str] = field(default_factory=list, repr=False)

    def __post_init__(self):
        if not self.name:
            raise ValueError("Name is required")
```

### Enum — именованные константы

```python
from enum import Enum, auto, StrEnum

class OrderStatus(StrEnum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class Priority(Enum):
    LOW = auto()   # 1
    MEDIUM = auto()  # 2
    HIGH = auto()    # 3
```

**StrEnum** (3.11+) — значения автоматически сравниваются со строками. Удобно для API.

---

## 5. pathlib — современный путь к файлам

```python
from pathlib import Path

BASE = Path(__file__).parent.parent
DATA = BASE / "data" / "fixtures"  # честный оператор /

# Glob — поиск файлов
for py_file in BASE.glob("**/*.py"):
    print(py_file.relative_to(BASE))

# Чтение/запись
content = (DATA / "config.json").read_text()
Path("output.txt").write_text("hello")

# Временные файлы
import tempfile
with tempfile.TemporaryDirectory() as tmp:
    p = Path(tmp) / "test.txt"
    p.write_text("data")
```

**Почему pathlib, а не os.path?**
- Кроссплатформенный (не нужно думать о / или \\)
- Читаемые цепочки вызовов
- Единый интерфейс для директорий/файлов

---

## 6. functools — функциональные утилиты

### lru_cache — кэш с вытеснением

```python
import functools

@functools.lru_cache(maxsize=128)
def get_user_permissions(user_id: int) -> list[str]:
    # Дорогой запрос к БД, но результат кэшируется
    ...

# Просмотр статистики
print(get_user_permissions.cache_info())
# CacheInfo(hits=50, misses=10, maxsize=128, currsize=10)

# Очистка
get_user_permissions.cache_clear()
```

**Важно:** `lru_cache` **не** thread-safe для async-функций. Для async используйте `@cache` (3.9+) или свою реализацию с asyncio.Lock.

### partial — фиксация аргументов

```python
from functools import partial

def fetch(url: str, timeout: int, retries: int):
    ...

fetch_with_timeout = partial(fetch, timeout=5)
fetch_from_api = partial(fetch_with_timeout, retries=3)

fetch_from_api("https://api.example.com")  # timeout=5, retries=3
```

### singledispatch — перегрузка функций по типу

```python
from functools import singledispatch
import json

@singledispatch
def serialize(obj):
    raise NotImplementedError(f"Unsupported type: {type(obj)}")

@serialize.register
def _(obj: dict):
    return json.dumps(obj)

@serialize.register
def _(obj: datetime):
    return obj.isoformat()

@serialize.register
def _(obj: Decimal):
    return str(obj)
```

---

> **На собесе:** «Какие модули из стандартной библиотеки используете?» —
> «collections.Counter для подсчёта, itertools.product для комбинаций,
> dataclasses для DTO, pathlib для путей, functools.lru_cache для кэширования,
> typing.Protocol для интерфейсов. Стараюсь не тянуть внешние зависимости,
> когда стандартная библиотека решает задачу.»