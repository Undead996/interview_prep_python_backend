# Стандартная библиотека Python — глубоко

---

## collections — продвинутые структуры

```python
from collections import defaultdict, Counter, deque, ChainMap, OrderedDict

# defaultdict — словарь с default-фабрикой
users = defaultdict(list)
users["admin"].append("Alice")  # автоматически users["admin"] = []

# Counter — подсчёт
from collections import Counter
logs = ["ERROR", "INFO", "ERROR", "WARN", "INFO", "ERROR"]
cnt = Counter(logs)  # {"ERROR": 3, "INFO": 2, "WARN": 1}
cnt.most_common(1)   # [("ERROR", 3)]

# deque — быстрая очередь/стек
queue = deque(maxlen=100)
queue.append("task1")
queue.popleft()  # быстрее, чем list.pop(0)

# ChainMap — объединение словарей (layered config)
defaults = {"debug": False, "port": 8080}
env = {"port": 5432}
config = ChainMap(env, defaults)  # config["port"] = 5432, config["debug"] = False
```

---

## itertools — эффективные итераторы

```python
import itertools

# product — декартово произведение (параметризация тестов)
for env, db in itertools.product(["dev", "prod"], ["pg", "mysql"]):
    print(env, db)

# cycle — бесконечный цикл (round-robin)
workers = itertools.cycle(["worker1", "worker2", "worker3"])
for task in tasks:
    worker = next(workers)
    assign(task, worker)

# groupby — группировка (данные должны быть отсортированы!)
data = sorted(data, key=lambda x: x["status"])
for status, group in itertools.groupby(data, key=lambda x: x["status"]):
    print(status, list(group))

# chain — цепочка итераторов
all_items = itertools.chain(page1, page2, page3)

# batched (3.12+) — разбить на батчи
for batch in itertools.batched(users, n=100):
    process_batch(batch)
```

---

## typing — строгая типизация

```python
from typing import (
    Optional, Union, Literal, TypeAlias, Protocol,
    Any, Callable, Awaitable, TypeVar, Generic,
)

T = TypeVar("T", bound="BaseModel")

def get_first(items: list[T]) -> T | None:
    return items[0] if items else None

# Protocol — утиная типизация
class Streamable(Protocol):
    async def read(self) -> bytes: ...

async def process_stream(stream: Streamable):
    data = await stream.read()
    ...

# Literal — константа
def set_mode(mode: Literal["sync", "async", "batch"]):
    ...
```

---

## dataclasses + enum

```python
from dataclasses import dataclass, field, asdict
from enum import Enum, auto, StrEnum

class Status(StrEnum):
    ACTIVE = "active"
    BLOCKED = "blocked"
    DELETED = "deleted"

    @classmethod
    def active_values(cls) -> list[str]:
        return [m.value for m in cls if m != cls.DELETED]

@dataclass
class Order:
    id: int
    status: Status
    items: list[str] = field(default_factory=list, repr=False)
    total: float = field(init=False)

    def __post_init__(self):
        self.total = sum(get_price(i) for i in self.items)
```

---

## pathlib (pathlib.Path вместо os.path)

```python
from pathlib import Path

BASE = Path(__file__).parent.parent
DATA = BASE / "data" / "fixtures"

for file in DATA.glob("*.json"):
    print(file.stem)  # имя без расширения
    data = file.read_text()  # UTF-8

# Временные файлы
import tempfile
with tempfile.TemporaryDirectory() as tmp:
    path = Path(tmp) / "test.txt"
    path.write_text("data")
```

---

## functools

```python
import functools

# lru_cache — кэширование результатов функции
@functools.lru_cache(maxsize=128)
def get_user_permissions(user_id: int):
    ...

# partial — фиксация аргументов
async_fetch = functools.partial(client.fetch, timeout=5)

# singledispatch — мультиметод
@functools.singledispatch
def serialize(obj):
    raise NotImplementedError

@serialize.register
def _(obj: dict):
    return json.dumps(obj)

@serialize.register
def _(obj: datetime):
    return obj.isoformat()
```

---

## Подводные камни

| ❌ Ошибка | ✅ Правильно |
|---|---|
| `isinstance(10, int)` — ОК, `type(10) == int` — не ОК | `isinstance` поддерживает наследование |
| `dict.keys()` не список в Python 3 | `list(dict.keys())` |
| `groupby` без сортировки | `sorted()` + `groupby` |
| `defaultdict(lambda: None)` | Создаёт ключи при обращении — осторожно! |

---

> **На собесе:** «Какие модули из стандартной библиотеки используете?» —
> «collections.Counter для подсчёта, itertools.product для комбинаций, dataclasses
> для DTO, pathlib для путей, functools.lru_cache для кэширования,
> typing.Literal для типов-констант.»