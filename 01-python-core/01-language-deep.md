# Python Language Deep Dive — расширенный разбор

> **Цель файла:** понять ***почему*** Python устроен так, а не иначе. Не просто "список immutable", а "почему tuple можно ключом dict, а list нет".

---

## 1. Типы данных и изменяемость (mutability)

### Почему это вообще важно на собесе?

Потому что mutability влияет на **всё**: от производительности до багов. На собесе проверяют не знание таблицы immutable vs mutable, а понимание **последствий**.

### Что такое mutability на уровне памяти

```python
a = [1, 2, 3]
b = a          # b — не копия, а ссылка на ТОТ ЖЕ объект
b.append(4)
print(a)       # [1, 2, 3, 4] — a изменился!
```

Когда вы пишете `b = a` для mutable-объекта, Python **не копирует данные**. Он копирует ссылку. Оба имени указывают на один и тот же объект в памяти.

Для immutable-объектов это не имеет значения, потому что объект нельзя изменить — любая "модификация" создаёт новый объект.

```python
a = "hello"
b = a
b += " world"  # создаётся НОВАЯ строка, a не изменилась
print(a)       # "hello"
```

### Детальная таблица

| Тип | Изменяемый? | Можно ключом dict? | Хранится в |
|-----|------------|-------------------|------------|
| `int` | ❌ | ✅ | Значение |
| `float` | ❌ | ✅ | Значение |
| `str` | ❌ | ✅ | Значение |
| `bool` | ❌ | ✅ | Значение (True=1, False=0!) |
| `tuple` | ❌ | ✅ (если все элементы immutable) | Ссылки на элементы |
| `frozenset` | ❌ | ✅ | Значение |
| `bytes` | ❌ | ✅ | Значение |
| `list` | ✅ | ❌ | Ссылка |
| `dict` | ✅ | ❌ | Ссылка |
| `set` | ✅ | ❌ | Ссылка |
| `bytearray` | ✅ | ❌ | Ссылка |

### Провальная зона: tuple с mutable элементами

```python
t = (1, [2, 3])   # tuple, но внутри него список
# t[1] = [4]      # ❌ TypeError — tuple immutable
t[1].append(4)     # ✅ работает — мы изменили список, а не tuple
# Такой tuple нельзя ключом dict!
d = {t: "value"}   # ❌ TypeError: unhashable type: 'list'
```

**Почему?** Потому что хэш tuple вычисляется на основе хэшей его элементов. Если элемент изменится — хэш перестанет быть актуальным. Python запрещает использовать unhashable типы как ключи dict.

### Практический пример: словарь со счётчиками

```python
# ❌ Плохо — mutable default (разберём ниже)
def count_words(text, counts={}):
    for word in text.split():
        counts[word] = counts.get(word, 0) + 1
    return counts

# ✅ Хорошо
def count_words(text, counts=None):
    if counts is None:
        counts = {}
    for word in text.split():
        counts[word] = counts.get(word, 0) + 1
    return counts
```

---

## 2. ООП и MRO (Method Resolution Order)

### Зачем МRO вообще нужно?

При единственном наследовании вопросов нет: `D(B)` → ищем в D, потом в B, потом в object. Но при **множественном наследовании** Python должен решить, **в каком порядке** искать методы, чтобы:
1. Ни один класс не был проверен дважды (избежать цикла)
2. Порядок был предсказуемым
3. Ребёнок имел приоритет над родителем

### C3 Linearization — как это работает

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
# [D, B, C, A, object]
# Почему B перед C? Потому что D(B, C) — B указан первым.
# Почему C перед A? Потому что A — родитель C, но C указан раньше,
#   чем A появился бы в MRO через B.
```

**Формула:** MRO — это `[D] + merge(MRO(B), MRO(C), [B, C])`.

`merge` работает так:
1. Берёт первый элемент первого списка.
2. Если он не встречается в хвостах других списков — помещает в результат.
3. Иначе — переходит к следующему списку.

### super() + MRO = кооперативное наследование

```python
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

D()  # A → C → B
```

**Почему A → C → B, а не A → B → C?** Потому что MRO(D) = [D, B, C, A, object]. `super()` в B вызывает следующий по MRO класс, то есть C, а не A.

**Ключевой вывод:** Если вы используете множественное наследование, **все классы должны вызывать `super().__init__()`**, иначе цепочка прервётся.

### `__new__` vs `__init__`

```python
class Singleton:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self, value):
        # Будет вызван КАЖДЫЙ раз при Singleton(value)
        self.value = value
```

- `__new__` — создаёт объект (вызывается **до** `__init__`), **обязан** вернуть экземпляр
- `__init__` — инициализирует поля, **не возвращает** ничего

**Когда переопределять `__new__`?**
- Singleton (выше)
- Кастомные метаклассы
- Иммутабельные типы (int, str, tuple) — только `__new__`, `__init__` не вызывается

---

## 3. Декораторы

### Как работает декоратор (уровень байткода)

```python
@decorator
def func():
    pass

# Это РАВНОСИЛЬНО:
func = decorator(func)
```

Декоратор — это просто синтаксический сахар для "применить функцию к функции". В байткоде `@decorator` превращается в `func = decorator(func)`.

### Декоратор с аргументами — почему три уровня?

```python
def retry(max_attempts=3, delay=0.1):
    # УРОВЕНЬ 1: принимает аргументы декоратора
    def decorator(func):
        # УРОВЕНЬ 2: принимает функцию
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            # УРОВЕНЬ 3: принимает аргументы функции
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except Exception:
                    if attempt == max_attempts:
                        raise
                    time.sleep(delay)
        return wrapper
    return decorator

# Использование:
@retry(max_attempts=3, delay=0.5)
def unstable_call():
    ...

# Эквивалент:
unstable_call = retry(max_attempts=3, delay=0.5)(unstable_call)
```

### @functools.wraps — почему он обязателен?

Без него декорированная функция теряет `__name__`, `__doc__`, `__module__`:

```python
def bare_decorator(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@bare_decorator
def greet(name):
    """Says hello"""
    print(f"Hello {name}")

print(greet.__name__)  # "wrapper" — ❌ потеряли имя!
print(greet.__doc__)   # None — ❌ потеряли документацию!
```

`@functools.wraps(func)` копирует `__name__`, `__doc__`, `__module__`, `__dict__` с исходной функции на wrapper.

### Декоратор как класс

```python
class CountCalls:
    def __init__(self, func):
        functools.update_wrapper(self, func)
        self.func = func
        self.calls = 0

    def __call__(self, *args, **kwargs):
        self.calls += 1
        print(f"Call {self.calls} of {self.func.__name__}")
        return self.func(*args, **kwargs)
```

**Когда класс лучше функции?**
- Когда нужно сохранять **состояние** между вызовами (как выше)
- Когда декоратор сложный и требует нескольких методов

### Где в реальном бэкенде применяют декораторы?

| Место | Декоратор | Зачем |
|-------|----------|-------|
| Эндпоинты | `@app.get("/path")` | FastAPI регистрирует функцию |
| Rate limiting | `@rate_limit(10, 60)` | Ограничение вызовов |
| Кэширование | `@lru_cache(maxsize=256)` | Кэш результатов функции |
| Retry | `@retry(max_attempts=3)` | Повтор при ошибках |
| Auth | `@require_role("admin")` | Проверка прав |
| Logging | `@log_execution_time` | Замер времени |
| Transaction | `@transactional` | Оборачивание в транзакцию |

---

## 4. Контекстные менеджеры

### Зачем они нужны в бэкенде?

Любая операция с ресурсом (БД, файл, HTTP-соединение, блокировка) должна:
1. Открыть/создать ресурс
2. **Гарантированно** закрыть/освободить, даже при исключении

Без контекстного менеджера:

```python
conn = create_connection()
try:
    conn.execute("...")
finally:
    conn.close()  # гарантированно
```

С контекстным менеджером:

```python
with create_connection() as conn:
    conn.execute("...")
# close() гарантированно вызван
```

### Как реализовать через класс

```python
class ManagedSession:
    def __enter__(self):
        self.session = create_session()
        return self.session

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type:
            self.session.rollback()
        else:
            self.session.commit()
        self.session.close()
        # Если вернуть True — исключение будет подавлено
        return False  # не подавлять
```

**Что делает `__exit__`:**
- Если `exc_type is not None` — было исключение, можно его обработать
- Если вернуть `True` — исключение будет **подавлено**
- Если вернуть `False` (или None) — исключение пробрасывается дальше

### Как реализовать через @contextmanager

```python
from contextlib import contextmanager

@contextmanager
def db_session():
    conn = create_connection()
    try:
        yield conn  # __enter__ возвращает это
    except Exception:
        conn.rollback()
        raise
    else:
        conn.commit()
    finally:
        conn.close()
```

**Важно:** `yield` может быть ровно один. Если внутри `with` произошло исключение, оно выбрасывается из `yield`. Можно поймать `try/except/yield`, чтобы обработать.

### Вложенные контекстные менеджеры

```python
with open("file.txt") as f, open("out.txt", "w") as out:
    for line in f:
        out.write(line.upper())

# Или
with open("file.txt") as f:
    with open("out.txt", "w") as out:
        for line in f:
            out.write(line.upper())
```

Оба файла будут гарантированно закрыты, даже при ошибке.

---

## 5. Data Classes (Python 3.7+)

### Что решают data classes?

До dataclasses:

```python
class User:
    def __init__(self, id, name, email):
        self.id = id
        self.name = name
        self.email = email

    def __repr__(self):
        return f"User(id={self.id}, name={self.name!r})"

    def __eq__(self, other):
        if not isinstance(other, User):
            return NotImplemented
        return self.id == other.id
```

После dataclasses:

```python
@dataclass
class User:
    id: int
    name: str
    email: str
```

Одна строчка `@dataclass` генерирует `__init__`, `__repr__`, `__eq__` (и опционально `__hash__`, `__lt__`, `__gt__`, `__le__`, `__ge__`).

### Параметры @dataclass

| Параметр | Что делает | Когда нужен |
|----------|-----------|-------------|
| `order=True` | Генерирует `__lt__`, `__le__`, `__gt__`, `__ge__` | Когда нужно сортировать |
| `frozen=True` | Все поля readonly после `__init__` | Immutable DTO |
| `slots=True` (3.10+) | `__slots__` вместо `__dict__` | Экономия памяти |
| `kw_only=True` (3.10+) | Все поля — keyword-only | Явные аргументы |

### field() — тонкая настройка

```python
@dataclass
class User:
    id: int
    name: str = field(compare=False)  # не участвует в сравнении
    email: str
    created_at: datetime = field(default_factory=datetime.now)
    tags: list[str] = field(default_factory=list, repr=False)  # не показывать в repr
    metadata: dict = field(default_factory=dict, hash=False)  # не участвует в hash

    def __post_init__(self):
        """Валидация после __init__"""
        if not self.email.count("@"):
            raise ValueError("Invalid email")
```

**`default_factory` vs `default`:** `default_factory` принимает **вызываемый объект** (функцию), который вызывается каждый раз при создании нового экземпляра. `default` — одно и то же значение, **общее** для всех экземпляров (mutable default trap!).

### Когда dataclass, а когда pydantic?

| Критерий | Data class | Pydantic |
|----------|-----------|----------|
| Валидация | `__post_init__` ручками | Авто, декораторы валидаторов |
| Сериализация | `dataclasses.asdict()` | `.model_dump()`, `.model_dump_json()` |
| Десериализация | Нет (ручками) | `.model_validate()` |
| Производительность | Быстрее | Медленнее (валидация) |
| FastAPI | Можно, но pydantic родной | Обязателен для request/response |

**Практика:** data class для внутренних DTO (слой репозитория), pydantic — для API-слоя.

---

## 6. Аннотации типов (typing)

### Зачем они на собесе?

Потому что Python стал **строже** с типами в 3.10–3.12. Код без типов на senior-позиции — красный флаг.

### Protocol — утиная типизация с проверкой

```python
from typing import Protocol

class Streamable(Protocol):
    async def read(self) -> bytes: ...

async def process_stream(stream: Streamable):
    data = await stream.read()
    ...

# Любой объект с async def read(self) -> bytes подойдёт
class FileStream:
    async def read(self) -> bytes:
        return await self.file.read()

await process_stream(FileStream())  # ✅
```

**`Protocol` отличается от `ABC` (AbstractBaseClass):**
- `ABC` требует явного наследования: `class FileStream(StreamableABC)`
- `Protocol` требует **только** совпадения сигнатуры — "утиная типизация для mypy"

### TypeAlias — читаемость

```python
from typing import TypeAlias

# Без алиаса:
def process(data: dict[str, "dict[str, int | str | float | bool | None] | list["dict[str, ...]"]):
    ...

# С алиасом:
JSON: TypeAlias = dict[str, "JSON"] | list["JSON"] | str | int | float | bool | None

def process(data: JSON):
    ...
```

### Generic — переиспользуемые типы

```python
from typing import TypeVar, Generic

T = TypeVar("T", bound="BaseModel")

class Repository(Generic[T]):
    async def get_by_id(self, id: int) -> T | None:
        ...

class UserRepo(Repository[User]):
    async def get_by_email(self, email: str) -> User | None:
        ...
```

### Literal — константы

```python
from typing import Literal

def set_mode(mode: Literal["sync", "async", "batch"]):
    ...
# set_mode("async") — ✅
# set_mode("parallel") — ❌ mypy warning
```

**На собесе:** "Какие модули из стандартной библиотеки используете?" — это вопрос не про память, а про **осознанность выбора**. Если вы используете `defaultdict`, вы должны сказать, почему он лучше обычного dict (не нужно проверять ключ). Если `lru_cache` — упомяните, что он thread-safe, но не для async-функций.