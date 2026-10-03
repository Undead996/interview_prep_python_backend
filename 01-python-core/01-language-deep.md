# Python Language Deep Dive — полный разбор с примерами

> **Цель файла:** понять ***почему*** Python устроен так, а не иначе. Не «список immutable-типов», а «почему tuple можно ключом dict, а list нет». С level-C объяснением под капотом.

---

## 1. Типы данных и изменяемость (mutability)

### Почему это важно на собесе?

Mutability влияет на **всё**: от производительности до трудноуловимых багов. На собесе проверяют не запоминание таблицы, а понимание **последствий**. Типичный вопрос:

> «Почему list нельзя использовать как ключ в dict?»

### Что такое mutability на уровне памяти

**Простыми словами:** immutable-объект — как надпись на камне: нельзя исправить, можно только высечь новую. Mutable-объект — как доска: можно дописывать и стирать.

```python
# Mutable: две переменные указывают на ОДИН объект
a = [1, 2, 3]
b = a               # b — НЕ копия, а та же ссылка
b.append(4)
print(a)            # [1, 2, 3, 4] — a тоже изменился!
print(a is b)       # True — это один и тот же объект

# Immutable: «модификация» создаёт новый объект
x = "hello"
y = x
y += " world"       # создаётся НОВАЯ строка
print(x)            # "hello" — x не изменилась
print(x is y)       # False — разные объекты
```

**Технически:** Каждое имя в Python — это ссылка на `PyObject` в куче. При `b = a` для mutable-типа копируется **указатель**, а не данные. Оба имени указывают на один и тот же участок памяти. При изменении через любое из имён — меняется один и тот же объект.

### Полная таблица типов

| Тип | Mutable? | Hashable (ключ dict)? | Где хранится | Примечание |
|-----|----------|----------------------|-------------|------------|
| `int` | ❌ | ✅ | Внутри PyObject как значение | CPython кэширует -5..256 |
| `float` | ❌ | ✅ | Внутри PyObject | NaN-ы не равны друг другу! |
| `str` | ❌ | ✅ | Внутри PyObject | Intern-строки (короткие, без пробелов) |
| `bytes` | ❌ | ✅ | Внутри PyObject | Как str, но байты |
| `bool` | ❌ | ✅ | True=1, False=0 (подкласс int!) | `isinstance(True, int)` → True |
| `tuple` | ❌ | ✅ (если все элементы hashable) | Ссылки на элементы | Если внутри list — НЕ hashable |
| `frozenset` | ❌ | ✅ | Хэш-таблица | Неизменяемый set |
| `list` | ✅ | ❌ | Динамический массив указателей | `list.append()` — O(1) амортизированно |
| `dict` | ✅ | ❌ | Хэш-таблица | Ключи — hashable |
| `set` | ✅ | ❌ | Хэш-таблица | Только hashable элементы |
| `bytearray` | ✅ | ❌ | Массив байтов | Mutable-версия bytes |

### Провальная зона: tuple с mutable элементом

```python
t = (1, [2, 3])

# Сам tuple изменить нельзя:
# t[1] = [4]   # ❌ TypeError: 'tuple' object does not support item assignment

# Но список ВНУТРИ tuple изменить МОЖНО:
t[1].append(4)  # ✅ работает
print(t)        # (1, [2, 3, 4]) — tuple "изменился"!

# Такой tuple НЕЛЬЗЯ ключом dict:
d = {t: "value"}  # ❌ TypeError: unhashable type: 'list'
```

**Почему?** Хэш tuple вычисляется как `hash(tuple) = hash((hash(e1), hash(e2), ...))`. Если e2 — list (unhashable), то и весь tuple — unhashable. Python проверяет это на этапе вставки в dict.

### Практические примеры из бэкенда

```python
# ❌ Ошибка 1: дефолтный список в параметре (см. pitfalls)
def process(items, processed=[]):
    processed.append(items)
    return processed

# ❌ Ошибка 2: изменение объекта, переданного как параметр
def add_user_role(user: dict, role: str) -> dict:
    user["roles"].append(role)  # меняет переданный dict!
    return user

# ✅ Правильно: копируем или используем immutable
def add_user_role(user: dict, role: str) -> dict:
    return {**user, "roles": [*user["roles"], role]}  # новый dict
```

---

## 2. ООП и MRO (Method Resolution Order)

### Зачем MRO вообще нужно?

**Простыми словами:** представь, что твой класс наследует два других, а те — ещё один общий. В каком порядке искать методы? Python использует алгоритм C3 Linearization, который гарантирует:
1. Ребёнок проверяется раньше родителей
2. Порядок родителей в `class D(B, C)` соблюдается (B раньше C)
3. Общий предок проверяется ровно один раз (нет ромбовидной проблемы)

### Diamond problem — почему без MRO хаос

```
    A
   / \
  B   C
   \ /
    D
```

Без MRO: если `D.method()` вызывает `super().method()`, должен ли он пойти через B или C? «Старый» Python (до 2.3) использовал "depth-first, left-to-right" — это приводило к тому, что A проверялся дважды, или родитель C имел приоритет над B.

**C3 Linearization (с Python 2.3):** `D.mro() = [D, B, C, A, object]`.

### Как работает C3 Linearization

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
# [<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>]
print(D().method())  # "B" — потому что B первый в MRO после D
```

**Алгоритм merge:**
```
MRO(D) = [D] + merge(MRO(B), MRO(C), [B, C])
       = [D] + merge([B, A, object], [C, A, object], [B, C])
       = [D, B] + merge([A, object], [C, A, object], [C])  # B взят из первого списка
       = [D, B, C] + merge([A, object], [A, object], [])   # C взят — он не в хвостах
       = [D, B, C, A] + merge([object], [object], [])
       = [D, B, C, A, object]
```

**Правило:** берём первый элемент первого списка. Если его нет в хвосте (все элементы кроме первого) ни одного другого списка — добавляем в результат. Иначе — пропускаем, переходим к следующему списку.

### super() и кооперативное наследование

```python
class A:
    def __init__(self):
        super().__init__()  # object.__init__()
        print("A")

class B(A):
    def __init__(self):
        super().__init__()  # ИДЁТ ПО MRO → следующий = C!
        print("B")

class C(A):
    def __init__(self):
        super().__init__()  # A.__init__()
        print("C")

class D(B, C):
    def __init__(self):
        super().__init__()  # B.__init__()

D()
# Вывод: A → C → B
# Почему? MRO(D) = [D, B, C, A, object]
# D.__init__ → super() → B.__init__ → super() → C.__init__ (не A!)
#   → super() → A.__init__ → "A" → "C" → "B"
```

**Ключевой вывод:** `super()` не вызывает метод родителя напрямую. Он вызывает **следующий класс в MRO**. Если вы используете множественное наследование, **все** классы должны вызывать `super().__init__()`, иначе цепочка прервётся.

### `__new__` vs `__init__` — порядок создания объекта

```python
class MyClass:
    def __new__(cls, *args, **kwargs):
        # Шаг 1: создание объекта (сырой памяти)
        instance = super().__new__(cls)
        print(f"__new__ called, type={type(instance)}")
        return instance  # ОБЯЗАН вернуть экземпляр

    def __init__(self, value):
        # Шаг 2: инициализация полей
        self.value = value
        print(f"__init__ called, value={value}")

obj = MyClass(42)
# __new__ called, type=<class 'MyClass'>
# __init__ called, value=42
```

| | `__new__` | `__init__` |
|--|----------|-----------|
| Когда вызывается | ДО `__init__` | ПОСЛЕ `__new__` |
| Что принимает | `cls` + args | `self` (уже созданный) + args |
| Что возвращает | Экземпляр (обязан!) | `None` (не должен возвращать) |
| Где переопределять | Singleton, metaclass, immutable types | Обычная инициализация |

### Singleton через `__new__`

```python
class Database:
    _instance = None

    def __new__(cls, url=None):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._initialized = False
        return cls._instance

    def __init__(self, url=None):
        if self._initialized:
            return  # не переинициализировать!
        self.url = url
        self._initialized = True

db1 = Database("postgres://...")
db2 = Database("another-url")  # игнорируется
print(db1 is db2)              # True
print(db1.url)                 # "postgres://..."
```

### Абстрактные классы и протоколы

```python
from abc import ABC, abstractmethod

class PaymentProcessor(ABC):
    @abstractmethod
    async def charge(self, amount: float) -> bool:
        """Списать деньги. Обязан переопределить."""
        ...

    @abstractmethod
    async def refund(self, transaction_id: str) -> bool:
        ...

# Нельзя создать экземпляр ABC:
# processor = PaymentProcessor()  # ❌ TypeError

class StripeProcessor(PaymentProcessor):
    async def charge(self, amount: float) -> bool:
        # реальная имплементация
        return True

    async def refund(self, transaction_id: str) -> bool:
        return True

processor = StripeProcessor()  # ✅
```

---

## 3. Декораторы

### Как работает декоратор

**Простыми словами:** декоратор — это «обёртка» над функцией. Он берёт функцию, добавляет к ней поведение (логирование, кэширование, проверку прав) и возвращает новую функцию. Всё это — синтаксический сахар для обычного вызова.

```python
@decorator
def func():
    pass

# Это РАВНОСИЛЬНО:
# func = decorator(func)
```

**Технически:** В байткоде `@decorator` компилируется в `CALL_FUNCTION` + `STORE_NAME`. Декоратор — это callable, который принимает callable и возвращает callable.

### Базовый декоратор

```python
import functools
import time

def log_execution_time(func):
    @functools.wraps(func)  # ← ОБЯЗАТЕЛЕН!
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result
    return wrapper

@log_execution_time
def process_data(items: list) -> int:
    """Обрабатывает данные и возвращает количество."""
    return sum(1 for _ in items)

print(process_data.__name__)  # "process_data" — с @wraps
print(process_data.__doc__)   # "Обрабатывает данные и возвращает количество."
```

### Почему `@functools.wraps` обязателен

Без него декорированная функция теряет метаданные:

```python
def bad_decorator(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@bad_decorator
def greet(name: str) -> str:
    """Says hello."""
    return f"Hello, {name}"

print(greet.__name__)      # "wrapper" — ❌
print(greet.__doc__)       # None — ❌
print(greet.__wrapped__)   # ❌ AttributeError
help(greet)                # показывает wrapper, а не greet
```

`@functools.wraps(func)` копирует `__name__`, `__doc__`, `__module__`, `__qualname__`, `__dict__`, `__wrapped__` с исходной функции на wrapper. Также обновляет `__signature__` (через `__wrapped__`).

### Декоратор с аргументами — почему три уровня?

```python
import functools
import time

def retry(max_attempts: int = 3, delay: float = 0.1, backoff: float = 2.0):
    """Повторяет вызов функции при ошибке с exponential backoff."""
    # УРОВЕНЬ 1: принимает аргументы декоратора → возвращает декоратор
    def decorator(func):
        # УРОВЕНЬ 2: принимает функцию → возвращает wrapper
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            # УРОВЕНЬ 3: принимает аргументы функции → вызывает функцию
            current_delay = delay
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts:
                        raise  # исчерпаны попытки
                    print(f"Attempt {attempt} failed: {e}. Retrying in {current_delay}s...")
                    time.sleep(current_delay)
                    current_delay *= backoff  # экспоненциальная задержка
        return wrapper
    return decorator

# Использование:
@retry(max_attempts=3, delay=0.5, backoff=2.0)
def unstable_api_call(url: str) -> dict:
    response = requests.get(url)
    if response.status_code >= 500:
        raise RuntimeError(f"Server error: {response.status_code}")
    return response.json()

# Эквивалент без сахара:
# unstable_api_call = retry(max_attempts=3, delay=0.5, backoff=2.0)(unstable_api_call)
```

**Почему три уровня, а не два?** Потому что `@retry` с аргументами — это вызов функции: `retry(max_attempts=3)`. Этот вызов должен **вернуть** что-то, что Python затем применит к функции. Это «что-то» и есть `decorator` (уровень 2).

### Декоратор как класс (с состоянием)

```python
import functools

class CountCalls:
    """Декоратор-класс: считает количество вызовов."""
    def __init__(self, func):
        functools.update_wrapper(self, func)
        self.func = func
        self.calls = 0

    def __call__(self, *args, **kwargs):
        self.calls += 1
        print(f"Call {self.calls} of {self.func.__name__!r}")
        return self.func(*args, **kwargs)

@CountCalls
def say_hi(name: str):
    return f"Hi {name}"

say_hi("Alice")  # Call 1 of 'say_hi'
say_hi("Bob")    # Call 2 of 'say_hi'
print(say_hi.calls)  # 2
```

**Когда класс лучше функции?**
- Когда нужно сохранять состояние между вызовами (счётчик, кэш, метрики)
- Когда декоратор сложный и требует нескольких методов

### Где в реальном бэкенде применяются декораторы

| Место | Декоратор | Что делает |
|-------|----------|-----------|
| FastAPI-эндпоинты | `@app.get("/path")` | Регистрирует функцию в роутере |
| Rate limiting | `@rate_limit(max=100, window=60)` | Сбрасывает 429 |
| Кэширование | `@lru_cache(maxsize=256)` | Кэширует результат |
| Retry | `@retry(max=3, backoff=2.0)` | Повторяет при ошибке |
| Auth | `@require_role("admin")` | Проверяет права |
| Валидация | `@validate(schema=UserSchema)` | Валидирует аргументы |
| Транзакции | `@transactional` | Оборачивает в begin/commit |
| Логирование | `@log_execution_time` | Замеряет время |
| Депрекация | `@deprecated("use new_func")` | Предупреждает |

---

## 4. Контекстные менеджеры

### Зачем они нужны в бэкенде?

**Простыми словами:** любая операция с ресурсом (БД, файл, сетевое соединение, блокировка) требует двух шагов: открыть и гарантированно закрыть. Контекстный менеджер автоматизирует «закрыть», даже если внутри блока произошла ошибка.

### Три способа реализации

#### Способ 1: класс с `__enter__` / `__exit__`

```python
class DatabaseSession:
    """Управляет сессией БД: commit при успехе, rollback при ошибке."""
    def __init__(self, db_url: str):
        self.db_url = db_url
        self.session = None

    def __enter__(self):
        self.session = create_session(self.db_url)
        return self.session

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is not None:
            # Было исключение — откатываем
            self.session.rollback()
            print(f"Rolled back due to: {exc_type.__name__}: {exc_val}")
        else:
            # Без ошибок — коммитим
            self.session.commit()
        self.session.close()
        return False  # НЕ подавляем исключение (если True — подавляет!)

# Использование:
with DatabaseSession("postgres://...") as db:
    db.execute("INSERT INTO ...")
    db.execute("UPDATE ...")
# commit (если без ошибок) или rollback + проброс исключения
```

#### Способ 2: `@contextmanager` (для простых случаев)

```python
from contextlib import contextmanager

@contextmanager
def db_session(db_url: str):
    """То же самое, но через генератор. Только ОДИН yield!"""
    session = create_session(db_url)
    try:
        yield session       # ← это точка входа в with-блок
    except Exception:
        session.rollback()
        raise               # пробрасываем исключение дальше
    else:
        session.commit()    # без ошибок
    finally:
        session.close()

# Использование идентично:
with db_session("postgres://...") as db:
    db.execute("...")
```

**⚠️ Осторожно:** `@contextmanager` позволяет **ровно один** `yield`. Если нужно несколько — используйте класс.

#### Способ 3: `contextlib.closing` (для объектов с `.close()`)

```python
from contextlib import closing
import urllib.request

with closing(urllib.request.urlopen("http://example.com")) as page:
    data = page.read()
# close() вызван гарантированно
```

### Вложенные контекстные менеджеры

```python
# Python 3.10+: можно через запятую
with (
    open("input.txt") as fin,
    open("output.txt", "w") as fout
):
    for line in fin:
        fout.write(line.upper())

# Старый способ:
with open("input.txt") as fin:
    with open("output.txt", "w") as fout:
        for line in fin:
            fout.write(line.upper())
```

Все файлы закроются даже при ошибке в любом из блоков.

### Что делает `__exit__`

Сигнатура: `__exit__(self, exc_type, exc_value, traceback) -> bool | None`

- Если исключения не было: `exc_type = exc_value = traceback = None`
- Если было: три аргумента содержат информацию об исключении
- Если `__exit__` возвращает `True` → исключение **подавлено** (как будто его не было)
- Если `False` или `None` → исключение пробрасывается дальше

```python
class SuppressKeyError:
    def __enter__(self): return self
    def __exit__(self, exc_type, exc_val, exc_tb):
        return exc_type is KeyError  # подавляем только KeyError

with SuppressKeyError():
    d = {}
    x = d["nonexistent"]  # KeyError подавлен
    print("This still runs!")  # напечатается
```

### Где в бэкенде применяются

| Ресурс | Контекстный менеджер |
|--------|---------------------|
| Сессия БД | `async with async_session() as db:` |
| Файл | `with open(...) as f:` |
| HTTP-клиент | `async with httpx.AsyncClient() as client:` |
| Блокировка | `async with asyncio.Lock():` |
| Семафор | `async with asyncio.Semaphore(10):` |
| RabbitMQ | `async with connection:` |
| Таймер | `with Timer() as t: ...; print(t.elapsed)` |

---

## 5. Data Classes (Python 3.7+)

### Что решают data classes?

**Проблема:** до dataclasses для простого DTO нужно было писать `__init__`, `__repr__`, `__eq__` — тонны boilerplate.

**Решение:** одна строка `@dataclass` генерирует всё это автоматически (и опционально: `__hash__`, `__lt__`, `__gt__`, `__le__`, `__ge__`, `__slots__`).

```python
# До dataclasses (~15 строк boilerplate)
class UserOld:
    def __init__(self, id: int, name: str, email: str):
        self.id = id
        self.name = name
        self.email = email
    def __repr__(self):
        return f"UserOld(id={self.id}, name={self.name!r}, email={self.email!r})"
    def __eq__(self, other):
        if not isinstance(other, UserOld):
            return NotImplemented
        return (self.id, self.name, self.email) == (other.id, other.name, other.email)

# С dataclasses — 4 строки:
@dataclass
class User:
    id: int
    name: str
    email: str
```

### Параметры `@dataclass`

| Параметр | По умолчанию | Что делает | Пример использования |
|----------|-------------|-----------|---------------------|
| `init=True` | ✅ | Генерирует `__init__` | Выключить для frozen-only DTO |
| `repr=True` | ✅ | Генерирует `__repr__` | Выключить для объектов с секретами |
| `eq=True` | ✅ | Генерирует `__eq__` | Сравнение по id, а не всем полям |
| `order=False` | ❌ | `__lt__`, `__le__`, `__gt__`, `__ge__` | Сортировка списков DTO |
| `unsafe_hash=False` | ❌ | `__hash__` (даже с mutable полями) | Только если уверены |
| `frozen=False` | ❌ | Поля неизменяемы после `__init__` | Immutable DTO для кэша |
| `slots=False` (3.10+) | ❌ | `__slots__` вместо `__dict__` | Экономия памяти (~50%) |
| `kw_only=False` (3.10+) | ❌ | Все поля keyword-only | Явность вызова |

### `field()` — тонкая настройка полей

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Any

@dataclass(order=True, frozen=True)
class Event:
    # Порядок сравнения: timestamp → type → (data не сравнивается)
    timestamp: datetime
    type: str = field(compare=True)
    data: dict[str, Any] = field(compare=False, repr=False)  # не в repr, не в сравнении
    id: str = field(default_factory=lambda: str(uuid.uuid4()), compare=False)
    created_at: datetime = field(
        default_factory=lambda: datetime.now(timezone.utc),
        compare=False
    )

    def __post_init__(self):
        """Валидация после генерации __init__."""
        if not self.type.strip():
            raise ValueError("Event type must not be empty")
        # Для frozen нужно использовать object.__setattr__
        if not self.data:
            object.__setattr__(self, "data", {})  # default factory не сработал

e = Event(timestamp=datetime.now(timezone.utc), type="user.created", data={"user_id": 42})
```

### Сравнение: dataclass vs Pydantic vs TypedDict

| Критерий | dataclass | Pydantic | TypedDict |
|----------|-----------|----------|-----------|
| Валидация | `__post_init__` (ручная) | Автоматическая, декораторы | ❌ Нет |
| Сериализация | `dataclasses.asdict()` | `.model_dump()` / `.model_dump_json()` | ❌ Нет |
| Десериализация | ❌ Нет | `.model_validate()` | ❌ Нет |
| JSON Schema | ❌ Нет | `.model_json_schema()` | ❌ Нет |
| Производительность | Быстрая (нативный Python) | Медленнее (валидация в runtime) | Очень быстрая |
| FastAPI | Можно, но pydantic — родной | ✅ Обязателен для request/response | Можно (Body) |
| Когда использовать | Внутренние DTO, слой репозитория | API-слой (request/response) | Простые dict-подобные структуры |

**Практика:** dataclass для внутренних DTO, Pydantic для API, TypedDict для аннотации словарей с известной структурой.

---

## 6. Аннотации типов (typing)

### Protocol — структурная типизация (утиная типизация со статической проверкой)

```python
from typing import Protocol, runtime_checkable

@runtime_checkable  # позволяет isinstance() в runtime (Python 3.8+)
class Streamable(Protocol):
    async def read(self) -> bytes:
        """Читает данные."""
        ...
    async def close(self) -> None:
        """Закрывает поток."""
        ...

async def process(stream: Streamable) -> str:
    data = await stream.read()
    await stream.close()
    return data.decode()

# Любой класс с read() + close() подходит — БЕЗ наследования!
class FileStream:
    async def read(self) -> bytes:
        return await self._file.read()
    async def close(self) -> None:
        await self._file.close()

class NetworkStream:
    async def read(self) -> bytes:
        return await self._socket.recv(4096)
    async def close(self) -> None:
        self._socket.shutdown()

await process(FileStream())    # ✅ mypy доволен
await process(NetworkStream()) # ✅ mypy доволен

# С @runtime_checkable:
print(isinstance(FileStream(), Streamable))  # True
```

**Protocol vs ABC:**
- `ABC` требует явного наследования: `class FileStream(StreamableABC)`
- `Protocol` требует только совпадения сигнатуры — structural subtyping
- `Protocol` проверяется **статически** (mypy/pyright)
- `ABC` проверяется и **в runtime** (isinstance/issubclass)

### TypeAlias и сложные типы

```python
from typing import TypeAlias
from uuid import UUID

# Без TypeAlias — невозможно читать:
def process(
    data: dict[str, list[dict[str, int | str | float | bool | None]]]
) -> int:
    ...

# С TypeAlias — читаемо:
JSON: TypeAlias = dict[str, "JSON"] | list["JSON"] | str | int | float | bool | None
UserId: TypeAlias = int
OrderId: TypeAlias = UUID

type EventPayload = dict[str, JSON]  # Python 3.12+ — альтернативный синтаксис!

def process(data: JSON) -> int:
    ...
```

### Generic — переиспользуемые типы

```python
from typing import TypeVar, Generic

T = TypeVar("T")
TModel = TypeVar("TModel", bound="BaseModel")  # только наследники BaseModel

class Repository(Generic[TModel]):
    """Обобщённый репозиторий для любого типа модели."""

    async def get_by_id(self, id: int) -> TModel | None:
        ...

    async def list(self, offset: int = 0, limit: int = 20) -> list[TModel]:
        ...

    async def save(self, entity: TModel) -> TModel:
        ...

    async def delete(self, id: int) -> bool:
        ...

# Использование:
class UserRepository(Repository[User]):
    async def get_by_email(self, email: str) -> User | None:
        ...

class OrderRepository(Repository[Order]):
    async def get_by_user(self, user_id: int) -> list[Order]:
        ...
```

### Literal, Final, TypedDict

```python
from typing import Literal, Final, TypedDict

# Literal — ограниченный набор значений
def set_log_level(level: Literal["debug", "info", "warning", "error"]) -> None:
    ...

set_log_level("info")     # ✅
# set_log_level("verbose")  # ❌ mypy: error

# Final — константа (не переопределять)
MAX_RETRIES: Final = 3
# MAX_RETRIES = 5  # ❌ mypy: error

# TypedDict — dict с известной структурой
class UserDict(TypedDict, total=False):  # total=False = не все поля обязательны
    id: int
    name: str
    email: str
    age: int

def print_user(user: UserDict) -> None:
    print(f"{user.get('name', 'Unknown')} <{user.get('email', 'no email')}>")

print_user({"id": 1, "name": "Alice", "email": "alice@ex.com"})  # ✅
print_user({"id": 2, "name": "Bob"})                              # ✅ (total=False)
```

---

> **На собесе:** «Расскажите про типы в Python» — начните с mutability и почему это важно. Затем ООП и MRO (diamond problem). Затем декораторы (3 уровня). Затем Protocol vs ABC. Покажите, что вы понимаете не синтаксис, а семантику и trade-off.