# Async и конкурентность в Python — от GIL до event loop

> **Цель:** понять, как работает async/await под капотом CPython, и **обоснованно** выбирать asyncio vs threading vs multiprocessing под задачу.

---

## 1. GIL — Global Interpreter Lock

### Что такое GIL на уровне C в CPython

**Простыми словами:** GIL — это «ключ», который разрешает только одному потоку исполнять Python-код в каждый момент времени. Это как касса в супермаркете: сколько бы ни было покупателей (потоков), обслуживается всегда только один.

**Технически:** CPython написан на C. Основной цикл исполнения байткода расположен в `ceval.c`. Без GIL каждое чтение/запись полей `PyObject` требовало бы отдельной блокировки, что замедлило бы однопоточный код в 2–4 раза. GIL — это **trade-off**:

| Плюсы GIL | Минусы GIL |
|-----------|-----------|
| Однопоточный код быстр (нет per-object блокировок) | Python-потоки не дают прироста для CPU-bound |
| C-расширения (numpy, pandas) могут просто отпустить GIL (`Py_BEGIN_ALLOW_THREADS`) и быть потокобезопасными | Нагрузка на CPU в одном процессе не масштабируется |
| Упрощает GC (нет гонок при подсчёте ссылок) | — |

### Что GIL блокирует, а что — нет

| Операция | Блокирует GIL? | Почему |
|----------|---------------|--------|
| Python-код (`a = b + c`) | ✅ Да | Исполняется в `ceval.c` |
| `socket.recv()` | ❌ Нет | GIL отпускается перед системным вызовом |
| `file.read()` | ❌ Нет | GIL отпускается на время I/O |
| `time.sleep()` | ❌ Нет | GIL отпускается, таймер — в ядре ОС |
| `numpy.dot()` | ❌ Нет | C-расширение использует `Py_BEGIN_ALLOW_THREADS` |
| asyncio event loop | ❌ Нет (один поток!) | Корутинное переключение — кооперативное, без GIL |
| `json.loads()` (чистый Python) | ✅ Да | Парсинг — Python-код |

### Практическое правило выбора

```
CPU-bound   (нагрузка на процессор)      → multiprocessing
I/O-bound   (сеть, диск, ожидание)      → asyncio (или threading для legacy-библиотек)
Смешанное   (CPU + I/O)                 → asyncio + ProcessPoolExecutor
```

### Как обойти GIL

```python
# 1. Multiprocessing — каждый процесс имеет СВОЙ GIL
from multiprocessing import Pool

def cpu_intensive(n: int) -> int:
    return sum(i * i for i in range(n))

with Pool(processes=4) as pool:
    results = pool.map(cpu_intensive, [1_000_000] * 8)

# 2. C-расширения — отпускают GIL
import numpy as np
# numpy работает на C и отпускает GIL во время вычислений
a = np.random.randn(10000, 10000)
b = a @ a.T  # GIL отпущен, могут работать другие потоки

# 3. asyncio — не использует потоки вообще
import asyncio

async def io_bound(url: str):
    # httpx.AsyncClient работает на asyncio — GIL не проблема
    async with httpx.AsyncClient() as client:
        return await client.get(url)
```

---

## 2. asyncio vs threading vs multiprocessing

### Детальная сравнительная таблица

| Характеристика | asyncio | threading | multiprocessing |
|---------------|---------|-----------|-----------------|
| **Единица исполнения** | Корутина (~1KB) | Поток (~2MB стека) | Процесс (отдельный GIL) |
| **Максимум одновременно** | ~100 000 (в одном потоке) | ~100 (ограничение ОС) | ~N CPU ядер |
| **Переключение** | Кооперативное (по `await`), ~0.1μs | Вытесняющее (ОС), ~1μs | Вытесняющее (ОС), ~100μs |
| **Общая память** | ✅ Один поток — нет гонок | ✅ Но нужны Lock/RLock/Semaphore | ❌ Каждый процесс — своя память |
| **GIL** | Не проблема (1 поток) | Проблема для CPU-bound | Не проблема (свой GIL) |
| **Обработка ошибок** | `try/except` вокруг `await` | `try/except` в run() | `try/except` в target |

### Когда что выбирать — практические примеры

```python
# ✅ asyncio — web-сервер (FastAPI), 10k WebSocket, парсинг сайтов
@app.get("/users")
async def get_users(db: AsyncSession = Depends(get_db)):
    return await db.execute(select(User))

# ✅ threading — старая синхронная библиотека без async-версии
def call_legacy_soap(url: str) -> dict:
    # библиотека не-async, но I/O-bound
    return legacy_soap_client.call(url)

async def wrapper(url: str):
    loop = asyncio.get_running_loop()
    return await loop.run_in_executor(None, call_legacy_soap, url)

# ✅ multiprocessing — обработка изображений, ML-инференс
def resize_image(path: str) -> bytes:
    img = Image.open(path)
    return img.resize((256, 256)).tobytes()

with ProcessPoolExecutor(max_workers=4) as executor:
    futures = [executor.submit(resize_image, path) for path in images]
    results = [f.result() for f in futures]
```

---

## 3. Event Loop — внутреннее устройство

### Что происходит при `await`

**Простыми словами:** `await` — это как нажать на «паузу» в видеоигре. Игра (корутина) ставится на паузу, а процессор идёт играть в другую игру. Через какое-то время первая игра «размораживается» и продолжается ровно с того же места.

```python
async def fetch_data():
    # 1. Python создаёт coroutine-object
    # 2. Дошли до await — корутина приостанавливается
    # 3. Event loop регистрирует таймер/сокет
    # 4. Управление возвращается event loop'у
    # 5. Event loop выбирает следующую готовую корутину
    # 6. Когда данные получены/таймер истёк — эту корутину кладут обратно в _ready
    # 7. Корутина продолжается со следующей после await строки
    data = await http_client.get("/api/data")
    return data

# Под капотом — это машина состояний:
# 1. __init__ → начальное состояние
# 2. __await__ → создаёт Future
# 3. send(None) → выполняет до первого await
# 4. send(result) → возобновляет после await с результатом
```

### Типы event loop на разных ОС

| ОС | Механизм | Event Loop | Примечание |
|----|----------|-----------|------------|
| Linux | `epoll` | `SelectorEventLoop` | По умолчанию |
| macOS | `kqueue` | `SelectorEventLoop` | По умолчанию |
| Windows | `IOCP` | `ProactorEventLoop` | Используется в `asyncio.run()` |

```python
# Можно выбрать явно:
import asyncio
import sys

if sys.platform == "win32":
    loop = asyncio.ProactorEventLoop()
    asyncio.set_event_loop(loop)
```

### run_in_executor — мост между sync и async

```python
import asyncio
import time
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

# Синхронная CPU-bound функция
def cpu_heavy(n: int) -> int:
    return sum(i * i for i in range(n))

async def main():
    loop = asyncio.get_running_loop()

    # ThreadPoolExecutor — для I/O-bound (GIL отпускается на I/O)
    result = await loop.run_in_executor(None, time.sleep, 1)  # default executor

    # ProcessPoolExecutor — для CPU-bound
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, cpu_heavy, 10_000_000)
```

---

## 4. Корутины, Tasks, Futures — что есть что

| Объект | Что это | Когда использовать |
|--------|--------|-------------------|
| **Coroutine** | Объект-генератор, созданный из `async def` | Почти никогда напрямую — всегда `await` или `create_task` |
| **Task** | Корутина, запланированная в event loop | Для фоновых задач, параллельного исполнения |
| **Future** | Низкоуровневое «обещание» результата | Для callback-based API, редко используется напрямую |

```python
import asyncio

async def worker(n: int) -> int:
    await asyncio.sleep(0.1)
    return n * 2

async def main():
    # Coroutine — сама не выполняется
    coro = worker(5)
    print(type(coro))  # <class 'coroutine'>

    # Task — выполняется в event loop
    task = asyncio.create_task(worker(3))
    print(type(task))  # <class 'Task'>
    result = await task  # дождаться Task
    print(result)       # 6

    # Future — низкоуровневое (редко используется напрямую)
    loop = asyncio.get_running_loop()
    future = loop.create_future()

    # Кто-то должен установить результат:
    loop.call_later(0.5, future.set_result, 42)
    result = await future
    print(result)  # 42

asyncio.run(main())
```

---

## 5. gather / TaskGroup / wait / as_completed

### gather — конкурентный запуск всех задач

```python
async def fetch_url(url: str) -> str:
    async with httpx.AsyncClient() as client:
        resp = await client.get(url)
        return f"{url}: {resp.status_code}"

async def main():
    urls = ["https://httpbin.org/get"] * 3

    # Без return_exceptions — одна ошибка убивает все задачи!
    results = await asyncio.gather(
        *[fetch_url(url) for url in urls],
        return_exceptions=True  # ← всегда ставь!
    )
    for i, result in enumerate(results):
        if isinstance(result, Exception):
            print(f"URL {i} failed: {result}")
        else:
            print(f"URL {i}: {result}")
```

### TaskGroup (Python 3.11+) — structured concurrency

```python
async def main():
    async with asyncio.TaskGroup() as tg:
        task1 = tg.create_task(worker(1))
        task2 = tg.create_task(worker(2))
        task3 = tg.create_task(worker(3))
    # Все задачи завершены (или одна упала → остальные отменены)
    print(task1.result(), task2.result(), task3.result())
```

**TaskGroup vs gather:**
- `TaskGroup` — structured concurrency: если одна задача упала → остальные отменяются
- `gather` — можно `return_exceptions=True` и обработать ошибки

### as_completed — обрабатывать по мере готовности

```python
async def main():
    tasks = [asyncio.create_task(worker(i)) for i in range(5)]

    for coro in asyncio.as_completed(tasks):
        result = await coro  # получаем результат ПЕРВОЙ завершившейся
        print(f"Got result: {result}")
```

---

## 6. Producer-Consumer с asyncio.Queue + Semaphore

Этот паттерн — основа любого queue-based микросервиса:

```python
import asyncio
from asyncio import Queue, Semaphore

async def worker(name: str, queue: Queue, sem: Semaphore):
    """Обрабатывает задачи из очереди с ограничением конкурентности."""
    while True:
        item = await queue.get()
        if item is None:  # сигнал завершения
            queue.task_done()
            break
        async with sem:
            print(f"[{name}] processing {item}")
            await asyncio.sleep(0.1)  # имитация работы
        queue.task_done()

async def main():
    queue: Queue[str | None] = Queue(maxsize=100)
    sem = Semaphore(3)  # не более 3 одновременных обработок

    # Запускаем 5 workers
    workers = [asyncio.create_task(worker(f"W{i}", queue, sem)) for i in range(5)]

    # Producer: кладём задачи
    for i in range(20):
        await queue.put(f"task-{i}")

    # Сигнал остановки для каждого worker
    for _ in workers:
        await queue.put(None)

    # Ждём завершения всех workers
    await asyncio.gather(*workers)
    print("All done")

asyncio.run(main())
```

---

## 7. Примитивы синхронизации в asyncio

```python
import asyncio

# Lock — аналог threading.Lock (взаимное исключение)
lock = asyncio.Lock()

async def critical_section(data: dict):
    async with lock:
        data["counter"] += 1

# Semaphore — ограничение конкурентности
sem = asyncio.Semaphore(10)  # не более 10 одновременных вызовов

async def rate_limited_request(url: str):
    async with sem:
        return await http_client.get(url)

# Event — сигнал (один поток ждёт, другой подаёт)
event = asyncio.Event()

async def waiter():
    print("Waiting...")
    await event.wait()  # блокируется до event.set()
    print("Got signal!")

async def signaler():
    await asyncio.sleep(1)
    event.set()  # разблокировать всех waiter'ов

# Condition — сложная координация
cond = asyncio.Condition()
buffer: list[int] = []

async def producer():
    for i in range(5):
        async with cond:
            buffer.append(i)
            cond.notify()

async def consumer():
    while True:
        async with cond:
            await cond.wait_for(lambda: len(buffer) > 0)
            item = buffer.pop(0)
            print(f"Consumed: {item}")
```

---

## 8. Типичные подводные камни (с примерами и объяснением)

### ❌ Ошибка 1: `time.sleep()` в async-функции

```python
async def bad_polling():
    while True:
        data = await fetch()
        time.sleep(5)  # ❌ БЛОКИРУЕТ ВЕСЬ EVENT LOOP на 5 секунд!

async def good_polling():
    while True:
        data = await fetch()
        await asyncio.sleep(5)  # ✅ отпускает управление
```

### ❌ Ошибка 2: `requests.get()` в async-функции

```python
# ❌ requests — синхронный, блокирует event loop
async def bad_fetch(url: str):
    return requests.get(url).json()

# ✅ httpx — async-native
async def good_fetch(url: str):
    async with httpx.AsyncClient() as client:
        resp = await client.get(url)
        return resp.json()

# ✅ run_in_executor для legacy-библиотек
async def okay_fetch(url: str):
    loop = asyncio.get_running_loop()
    resp = await loop.run_in_executor(None, requests.get, url)
    return resp.json()
```

### ❌ Ошибка 3: `gather()` без `return_exceptions`

```python
async def may_fail(n: int) -> int:
    if n == 3:
        raise ValueError("Bad number!")
    return n * 2

# ❌ Одна ошибка — все остальные задачи отменяются
results = await asyncio.gather(*[may_fail(i) for i in range(5)])
# RuntimeError!

# ✅ Ошибки возвращаются как значения
results = await asyncio.gather(
    *[may_fail(i) for i in range(5)],
    return_exceptions=True
)
# [0, 2, 4, ValueError("Bad number!"), 8]
```

### ❌ Ошибка 4: не `await` корутину

```python
async def main():
    fetch_data()  # ❌ RuntimeWarning: coroutine was never awaited
    # Правильно:
    await fetch_data()         # ждать
    asyncio.create_task(fetch_data())  # или фоновая задача
```

### ❌ Ошибка 5: `asyncio.run()` внутри `async def`

```python
async def inner():
    return 42

async def outer():
    result = asyncio.run(inner())  # ❌ RuntimeError: event loop already running
    # Правильно:
    result = await inner()
```

---

> **На собесе:** «Расскажите, как работает asyncio» —
> «asyncio — это событийный цикл, который кооперативно переключает корутины.
> Корутина приостанавливается через `await`, отдавая управление event loop'у.
> Event loop использует epoll/kqueue/IOCP для неблокирующего I/O.
> В одном потоке работают тысячи корутин. GIL не проблема — он отпускается на I/O.
> Для CPU-bound используем multiprocessing или run_in_executor с ProcessPoolExecutor.»