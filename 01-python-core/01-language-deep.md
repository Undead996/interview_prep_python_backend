# Python Language Deep Dive

---

## Типы данных и изменяемость

```python
immutable: int, float, str, bool, tuple, frozenset, bytes
mutable:   list, dict, set, bytearray

# Строки — неизменяемы!
s = "hello"
s[0] = "H"  # ❌ TypeError

# Список — изменяем
lst = [1, 2, 3]
lst[0] = 99  # ✅

# Tuple как ключ словаря
d = {(1, 2): "point"}  # ✅
d = {[1, 2]: "point"}  # ❌ TypeError
```

---

## ООП и MRO

```python
class A:
    def method(self): return "A"

class B(A):
    def method(self): return "B"

class C(A):
    def method(self): return "C"

class D(B, C):
    pass

print(D.mro())
# D → B → C → A → object
print(D().method())  # "B"

# super() в множественном наследовании
class A:
    def __init__(self):
        print("A")

class B(A):
    def __init__(self):
        super().__init__()
        print("B")

class C(A):
    def __init__(self):
        super().__init__()
        print("C")

class D(B, C):
    def __init__(self):
        super().__init__()

D()  # A → C → B  (MRO!)
```

---

## Декораторы

```python
import functools
import time

# Декоратор с аргументами
def retry(max_attempts: int = 3, delay: float = 0.1):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts:
                        raise
                    time.sleep(delay)
            return None
        return wrapper
    return decorator

@retry(max_attempts=3, delay=0.5)
def unstable_network_call():
    ...

# Декоратор как класс
class CountCalls:
    def __init__(self, func):
        functools.update_wrapper(self, func)
        self.func = func
        self.calls = 0

    def __call__(self, *args, **kwargs):
        self.calls += 1
        return self.func(*args, **kwargs)
```

---

## Контекстные менеджеры

```python
# Через класс
class DatabaseSession:
    def __enter__(self):
        self.conn = create_connection()
        return self.conn

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type:
            self.conn.rollback()
        else:
            self.conn.commit()
        self.conn.close()

with DatabaseSession() as conn:
    conn.execute("INSERT ...")

# Через contextlib
from contextlib import contextmanager

@contextmanager
def db_session():
    conn = create_connection()
    try:
        yield conn
    except Exception:
        conn.rollback()
        raise
    else:
        conn.commit()
    finally:
        conn.close()
```

---

## Data Classes

```python
from dataclasses import dataclass, field, asdict
from datetime import datetime
from typing import Optional

@dataclass(order=True, frozen=True)
class User:
    id: int
    name: str = field(compare=False)
    email: str
    created_at: datetime = field(default_factory=datetime.now)
    tags: list[str] = field(default_factory=list)

    def __post_init__(self):
        """Валидация после инициализации"""
        if not self.email.count("@"):
            raise ValueError("Invalid email")

user = User(id=1, name="Alice", email="alice@example.com")
print(asdict(user))  # в dict
```

---

## Аннотации типов (typing)

```python
from typing import Optional, Union, Literal, TypeAlias
from collections.abc import Sequence, Mapping

JSON: TypeAlias = dict[str, "JSON"] | list["JSON"] | str | int | float | bool | None

def process_users(
    users: Sequence[User],
    role: Literal["admin", "user"] = "user",
    limit: int = 10,
) -> list[dict]:
    """Type hint — документирует и помогает mypy"""
    ...

def parse_config(path: str) -> dict[str, int | str]:
    ...
```

---

## Подводные камни

| ❌ Ошибка | ✅ Правильно |
|---|---|
| `is` для сравнения чисел | `is None`, `==` для значений |
| `except:` без типа | `except Exception:` |
| mutable default | `def f(x=None):` |
| Late binding в лямбдах | `lambda x=x: x` |
| `[] * n` создаёт ссылки на один список | `[[] for _ in range(n)]` |

---

> **На собесе:** «Расскажите про MRO в Python» — «C3 linearization. Python строит
> порядок разрешения методов. super() следует MRO. В D(B, C) → D → B → C → object.
> Используется для кооперативного наследования через super().»