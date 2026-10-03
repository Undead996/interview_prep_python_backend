# Шпаргалка: Python — последний день

---

## Типы / структуры

```python
immutable: int, str, float, bool, tuple, frozenset, bytes
mutable:   list, dict, set, bytearray

# Tuple с mutable внутри — не hashable!
d = {(1, [2]): "x"}  # ❌ TypeError
d = {(1, 2): "x"}    # ✅
```

## Async

```python
async def fetch(url): ...
async with httpx.AsyncClient() as c: ...
await asyncio.gather(t1, t2, return_exceptions=True)
asyncio.run(main())
asyncio.create_task(coro)
loop.run_in_executor(None, sync_fn)

# Async queue
queue = asyncio.Queue()
await queue.put(item)
item = await queue.get()
queue.task_done()

# Semaphore — ограничение конкурентности
sem = asyncio.Semaphore(10)
async with sem:
    await request()
```

## Декоратор

```python
@functools.wraps(func)
def wrapper(*args, **kwargs):
    return func(*args, **kwargs)
```

## Контекстный менеджер

```python
# Класс
class Ctx:
    def __enter__(self): return self
    def __exit__(self, typ, val, tb): ...

# @contextmanager
@contextmanager
def ctx():
    try:
        yield resource
    finally:
        cleanup()
```

## dataclass

```python
@dataclass(order=True, frozen=True)
class User:
    id: int
    name: str = field(compare=False)
    tags: list[str] = field(default_factory=list)

    def __post_init__(self):
        if not self.name: raise ValueError
```

## Типизация

```python
from typing import Protocol, TypeAlias, Literal

JSON: TypeAlias = dict[str, "JSON"] | list | str | int | float | bool | None

class Streamable(Protocol):
    async def read(self) -> bytes: ...
```

## Стандартная библиотека

```python
from collections import defaultdict, Counter, deque
from itertools import product, cycle, groupby, chain, batched
from functools import lru_cache, partial, singledispatch
from pathlib import Path
```