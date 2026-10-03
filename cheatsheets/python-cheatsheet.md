# Python Cheatsheet — последний день

## Типы / Hashable

```python
immutable: int, str, float, bool, tuple, frozenset, bytes
mutable:   list, dict, set, bytearray

d = {(1, [2]): "x"}  # ❌ TypeError — tuple с list внутри не hashable
```

## Async / Await

```python
async def fetch(url): ...
async with httpx.AsyncClient() as c: ...
await asyncio.gather(t1, t2, return_exceptions=True)
asyncio.create_task(coro)
asyncio.wait_for(task, timeout=5.0)
loop.run_in_executor(None, sync_fn)

# Async queue:
q = asyncio.Queue(maxsize=100)
await q.put(item); item = await q.get(); q.task_done()

# Semaphore:
sem = asyncio.Semaphore(10)
async with sem: await request()

# ❌ time.sleep(1) в async → await asyncio.sleep(1)
# ❌ requests.get() в async → httpx.AsyncClient
```

## Декоратор (3 уровня для @retry(max=3))

```python
def retry(max_attempts=3, delay=0.1):
    def decorator(func):
        @functools.wraps(func)  # ← ОБЯЗАТЕЛЕН
        def wrapper(*args, **kwargs):
            for i in range(max_attempts):
                try: return func(*args, **kwargs)
                except: time.sleep(delay)
            raise
        return wrapper
    return decorator
```

## Контекстный менеджер

```python
# Класс: __enter__ → __exit__(exc_type, val, tb)
# @contextmanager: yield перед try, cleanup в finally
```

## dataclass

```python
@dataclass(order=True, frozen=True)
class User:
    id: int
    tags: list[str] = field(default_factory=list, repr=False)
    def __post_init__(self): ...
```

## collections / itertools

```python
from collections import defaultdict, Counter, deque, ChainMap
from itertools import product, cycle, groupby, chain, batched  # batched=3.12+
from functools import lru_cache, partial, singledispatch
from pathlib import Path

# ⚠️ lru_cache НЕ для async-функций!
```

## typing

```python
from typing import Protocol, TypeAlias, Literal, Generic, TypeVar

JSON: TypeAlias = dict[str, "JSON"] | list["JSON"] | str | int | float | bool | None

class Streamable(Protocol):
    async def read(self) -> bytes: ...

T = TypeVar("T", bound=BaseModel)
class Repo(Generic[T]): ...
```