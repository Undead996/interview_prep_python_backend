# Async и конкурентность в Python — глубоко

---

## 1. GIL (Global Interpreter Lock)

> **Простыми словами:** GIL — это «охраник», который стоит у двери Python-кода.
> Он пропускает только одного посетителя за раз. Даже если у вас 8 ядер,
> только один поток может исполнять Python-код в любой момент времени.
> Это защищает внутренние структуры CPython от race condition, но ценой
> производительности на многоядерных системах.

### Почему GIL существует

```python
# Без GIL такой код мог бы привести к race condition:
a = [1, 2, 3]
# Два потока одновременно:
b = a[0] + a[1]        # thread 1
a.append(4)             # thread 2
# Если бы они исполнялись одновременно — a мог бы измениться между чтением и записью
```

**GIL решает:** не нужно писать блокировки на каждую операцию с памятью.  
**Цена:** CPU-bound код не масштабируется на много ядер.

### Что GIL НЕ блокирует

| Операция | GIL блокирует? | Комментарий |
|---|---|---|
| Python-код (bytecode) | ✅ Да | Каждый байткод исполняется под GIL |
| I/O операции (socket, file) | ❌ Нет | GIL отпускается на время I/O |
| C-расширения (numpy, pandas) | ❌ Нет | Могут отпускать GIL (Py_BEGIN_ALLOW_THREADS) |
| `time.sleep()` | ❌ Нет | GIL отпускается на время сна |
| `asyncio` event loop | ❌ Нет | Однопоточный, GIL не мешает |

### Как обойти GIL для CPU-bound задач

```python
# 1. Multiprocessing — каждый процесс имеет свой GIL
from multiprocessing import Pool
with Pool(4) as pool:
    results = pool.map(cpu_intensive_fn, data)

# 2. concurrent.futures.ProcessPoolExecutor
from concurrent.futures import ProcessPoolExecutor
with ProcessPoolExecutor(max_workers=4) as executor:
    results = list(executor.map(cpu_intensive_fn, data))

# 3. C-расширения (numpy, numba, cython) — могут отпускать GIL
import numpy as np
# numpy отпускает GIL на время вычислений

# 4. asyncio + loop.run_in_executor (ProcessPoolExecutor)
loop = asyncio.get_running_loop()
result = await loop.run_in_executor(ProcessPoolExecutor(), cpu_fn, arg)
```

### GIL-free Python (free-threading)

```bash
# Python 3.13+ — экспериментальный режим без GIL
PYTHON_GIL=0 python my_script.py
```

---

## 2. Сравнение: asyncio vs threading vs multiprocessing

```
                     asyncio         threading       multiprocessing
                   ┌──────────┐   ┌──────────┐    ┌──────────┐
   Потоков/процес. │    1     │   │    N     │    │    N     │
                   ├──────────┤   ├──────────┤    ├──────────┤
   Использование   │   I/O    │   │   I/O    │    │   CPU    │
                   ├──────────┤   ├──────────┤    ├──────────┤
   GIL-проблемы    │    Нет   │   │   Есть   │    │   Нет    │
                   ├──────────┤   ├──────────┤    ├──────────┤
   Память          │  общая   │   │  общая   │    │ раздельн.│
                   ├──────────┤   ├──────────┤    ├──────────┤
   Сложность       │ средняя  │   │  низкая  │    │  высокая │
                   └──────────┘   └──────────┘    └──────────┘
```

### Когда что выбирать

| Сценарий | asyncio | threading | multiprocessing | Почему |
|---|---|---|---|---|
| Web-сервер (FastAPI) | ✅ | ❌ | ❌ | asyncio — событийный, не блокирует event loop |
| Парсинг 1000 сайтов | ✅ | ❌ | ❌ | I/O-bound, asyncio быстрее и легче |
| Блокирующие I/O (старая библиотека) | ❌ | ✅ | ❌ | run_in_executor не всегда спасает |
| CPU-intensive (обработка изображений) | ❌ | ❌ | ✅ | multiprocessing обходит GIL |
| Микросервис с 10k соединений | ✅ | ❌ | ❌ | asyncio — одно соединение ~ корутина (KB) |
| Научные расчёты | ❌ | ❌ | ✅ | ProcessPoolExecutor + numpy отпускает GIL |

---

## 3. Event Loop — сердце asyncio

```
  ┌────────────────────────────────────────────────────┐
  │                Event Loop                           │
  │                                                     │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
  │  │ coroutine│  │ coroutine│  │ coroutine│          │
  │  │ (waiting)│  │ (running)│  │ (done)   │          │
  │  └──────────┘  └──────────┘  └──────────┘          │
  │       │             │             │                 │
  │       ▼             ▼             ▼                 │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
  │  │ asyncio. │  │ asyncio. │  │ asyncio. │          │
  │  │  Queue   │  │  Queue   │  │  Queue   │          │
  │  └──────────┘  └──────────┘  └──────────┘          │
  │       │             │             │                 │
  │       ▼─────────────▼─────────────▼────             │
  │  ┌────────────────────────────────────────┐         │
  │  │         I/O (await)                    │         │
  │  │  socket ready? timer fired? signal?    │         │
  │  └────────────────────────────────────────┘         │
  └────────────────────────────────────────────────────┘
```

### Как работает event loop (упрощённо)

```python
import asyncio
import select

# Псевдо-код работы event loop
class SimpleEventLoop:
    def __init__(self):
        self._ready = []          # готовые корутины
        self._waiting = {}        # {fd: coroutine}

    def run_forever(self):
        while self._ready or self._waiting:
            # 1. Исполнить готовые корутины
            for coro in self._ready:
                try:
                    coro.send(None)  # запустить до await
                except StopIteration:
                    continue
                except Exception as e:
                    # если корутина упала — уведомить
                    coro.throw(e)

            # 2. Ждать I/O (select)
            readable, _, _ = select.select(
                [fd for fd in self._waiting],
                [], [], timeout=0.01
            )
            for fd in readable:
                coro = self._waiting.pop(fd)
                self._ready.append(coro)
```

Настоящий asyncio использует `selectors` (или `epoll`/`kqueue`/`iocp` на разных ОС).

### Типы event loop

```python
import asyncio

# Default (выбирается автоматически)
loop = asyncio.new_event_loop()
asyncio.set_event_loop(loop)

# Просмотр текущего loop
loop = asyncio.get_running_loop()   # внутри async def (RuntimeError если нет)
loop = asyncio.get_event_loop()     # вне async def (deprecated, но работает)

# Платформенные реализации
# Linux:   asyncio.SelectorEventLoop (epoll) — по умолчанию
# Windows: asyncio.ProactorEventLoop (IOCP) — для asyncio.run()
# macOS:   asyncio.SelectorEventLoop (kqueue)
```

### run_in_executor — мост между sync и async

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

# Глобальный executor (переиспользуется)
_IO_EXECUTOR = ThreadPoolExecutor(max_workers=4)
_CPU_EXECUTOR = ProcessPoolExecutor(max_workers=os.cpu_count())

async def sync_to_async(func, *args, executor=_IO_EXECUTOR):
    """Запустить синхронную функцию в executor (не блокирует event loop)"""
    loop = asyncio.get_running_loop()
    return await loop.run_in_executor(executor, func, *args)

# Использование
async def fetch_legacy():
    data = await sync_to_async(legacy_requests.get, "https://api.example.com")
    return data.json()
```

---

## 4. Корутины, Tasks, Futures

```python
import asyncio

async def my_coroutine():
    """Корутина — специальная функция, которая может приостанавливаться"""
    await asyncio.sleep(1)
    return 42

# Корутина ≠ Task
coro = my_coroutine()          # <coroutine> — объект корутины
task = asyncio.create_task(coro)  # Task — обёртка, планирует выполнение

# Task — это Future + управление event loop
# Future — обещание результата (низкоуровнево)

async def future_example():
    loop = asyncio.get_running_loop()
    future = loop.create_future()

    # Через 1 секунду установить результат
    loop.call_later(1, future.set_result, 42)

    # Ждать результат (не блокируя event loop)
    result = await future
    print(result)  # 42
```

### Жизненный цикл Task

```
  Created (coroutine)     →   Scheduled (create_task)
        │                           │
        ▼                           ▼
   Pending (ждёт в event loop queue)
        │
        ▼
   Running (выполняется)
        │
   ┌────┴────┐
   ▼         ▼
Done     Cancelled / Exception
```

### Создание задач

```python
# create_task — прямолинейно
async def main():
    task = asyncio.create_task(my_coroutine())
    await task

# ensure_future — совместимость (принимает корутину или Future)
task = asyncio.ensure_future(my_coroutine())

# asyncio.run — создаёт loop, запускает главную корутину, закрывает loop
result = asyncio.run(my_coroutine())
```

---

## 5. Запуск конкурентных задач

### asyncio.gather — конкурентный запуск

```python
async def fetch_url(url: str, delay: float = 1.0) -> dict:
    await asyncio.sleep(delay)  # имитация I/O
    return {"url": url, "status": 200}

async def main():
    # Без gather — последовательно
    r1 = await fetch_url("https://a.com", 1)  # 1s
    r2 = await fetch_url("https://b.com", 1)  # ещё 1s → всего 2s

    # С gather — конкурентно
    r1_task = fetch_url("https://a.com", 1)
    r2_task = fetch_url("https://b.com", 1)
    results = await asyncio.gather(r1_task, r2_task)
    # ~1s (обе параллельно)

    # return_exceptions — не убивать остальные при ошибке
    results = await asyncio.gather(
        fetch_url("ok.com"),
        fetch_url("fail.com"),  # упадёт с ошибкой
        return_exceptions=True,  # ошибка вернётся как Exception, а не убьёт gather
    )
    for r in results:
        if isinstance(r, Exception):
            print(f"Task failed: {r}")
        else:
            print(f"Success: {r}")
```

### asyncio.TaskGroup (Python 3.11+)

```python
async def main():
    # TaskGroup — structured concurrency
    async with asyncio.TaskGroup() as tg:
        t1 = tg.create_task(fetch_url("a.com", 1))
        t2 = tg.create_task(fetch_url("b.com", 2))

    # Здесь все задачи завершены (или отменены при ошибке)
    # Если t1 упал → t2 автоматически отменяется
    # Аналог: structured concurrency из trio

    # Результаты доступны:
    print(t1.result())
```

### asyncio.wait — более гибкий контроль

```python
async def main():
    task1 = asyncio.create_task(fetch_url("slow.com", 5))
    task2 = asyncio.create_task(fetch_url("fast.com", 1))

    # Ждать первую завершённую (FIRST_COMPLETED)
    done, pending = await asyncio.wait(
        [task1, task2],
        return_when=asyncio.FIRST_COMPLETED,
    )
    # done — завершённые, pending — ожидающие
    for t in done:
        print(t.result())

    # Отменить оставшиеся
    for t in pending:
        t.cancel()

    # Ждать с таймаутом
    done, pending = await asyncio.wait(
        [task1, task2],
        timeout=2.0,
    )
    print(f"Completed: {len(done)}, Pending: {len(pending)}")
```

### asyncio.as_completed — по мере завершения

```python
async def main():
    tasks = [fetch_url(f"https://site{i}.com", delay=i*0.5) for i in range(5)]

    # Обрабатывать по мере завершения (не по порядку)
    for coro in asyncio.as_completed(tasks):
        result = await coro
        print(f"Got: {result['url']}")
```

---

## 6. Async-очереди (producer-consumer)

```python
import asyncio
import random

async def producer(queue: asyncio.Queue, num_items: int):
    for i in range(num_items):
        item = f"item-{i}"
        await queue.put(item)
        print(f"Produced: {item}")
        await asyncio.sleep(random.random())
    # Сигнал окончания
    await queue.put(None)

async def consumer(queue: asyncio.Queue, name: str):
    while True:
        item = await queue.get()
        if item is None:
            # Последний consumer получает None — завершение
            await queue.put(None)  # для следующего consumer
            break
        print(f"Consumer {name}: processing {item}")
        await asyncio.sleep(random.random())
        queue.task_done()

async def main():
    queue = asyncio.Queue(maxsize=10)

    # Создаём producer и два consumer
    producer_task = asyncio.create_task(producer(queue, 10))
    consumer1 = asyncio.create_task(consumer(queue, "A"))
    consumer2 = asyncio.create_task(consumer(queue, "B"))

    await asyncio.gather(producer_task, consumer1, consumer2)

# asyncio.run(main())
```

### Queue с приоритетом

```python
import asyncio
from dataclasses import dataclass, field

@dataclass(order=True)
class PrioritizedItem:
    priority: int
    data: str = field(compare=False)

async def priority_consumer(queue: asyncio.PriorityQueue):
    while True:
        item = await queue.get()
        if item.data is None:
            break
        print(f"Processing priority {item.priority}: {item.data}")
        queue.task_done()
```

### Queue с таймаутом

```python
async def get_with_timeout(queue: asyncio.Queue, timeout: float):
    try:
        return await asyncio.wait_for(queue.get(), timeout=timeout)
    except asyncio.TimeoutError:
        return None  # таймаут — ничего не ждём
```

---

## 7. Примитивы синхронизации

### Lock

```python
import asyncio

lock = asyncio.Lock()

async def update_resource():
    # Защита общего ресурса (но в async обычно не нужно — нет гонок)
    async with lock:
        # Только одна корутина за раз
        await modify_shared_state()
```

### Semaphore — ограничение конкурентности

```python
import asyncio

# Не более 10 одновременных соединений к БД
db_sem = asyncio.Semaphore(10)

async def query_db(query):
    async with db_sem:
        return await execute_query(query)

# BoundedSemaphore — не может превысить начальное значение при release
sem = asyncio.BoundedSemaphore(5)
```

### Event — сигнал между корутинами

```python
event = asyncio.Event()

async def waiter():
    print("Ждём события...")
    await event.wait()
    print("Событие получено!")

async def setter():
    await asyncio.sleep(1)
    print("Устанавливаем событие")
    event.set()

# Очистка
event.clear()
# Проверка без ожидания
if event.is_set():
    ...
```

### Condition — сигнал + блокировка

```python
condition = asyncio.Condition()

async def consumer():
    async with condition:
        await condition.wait()  # отпускает Lock, ждёт notify
        print("Consumer просыпается")

async def producer():
    async with condition:
        # Изменение общего состояния
        await condition.notify(1)  # разбудить одного ждущего
```

---

## 8. Timeout и Cancellation

### asyncio.wait_for — таймаут

```python
async def main():
    try:
        # Ждать не более 5 секунд
        result = await asyncio.wait_for(
            slow_operation(),
            timeout=5.0,
        )
    except asyncio.TimeoutError:
        print("Operation timed out, task cancelled")
```

### asyncio.timeout (Python 3.11+)

```python
async def main():
    # asyncio.timeout — контекстный менеджер
    async with asyncio.timeout(5.0):
        result = await slow_operation()
    # Если таймаут — TimeoutError
```

### Отмена задач

```python
async def main():
    task = asyncio.create_task(long_running())

    await asyncio.sleep(1)
    # Принудительная отмена
    task.cancel()

    try:
        await task  # может бросить CancelledError
    except asyncio.CancelledError:
        print("Task was cancelled")

    # Проверка отмены
    if task.cancelled():
        print("Task is cancelled")
```

### Graceful cancellation

```python
async def graceful_worker():
    try:
        while True:
            await do_work()
    except asyncio.CancelledError:
        # Чистим ресурсы перед завершением
        await cleanup()
        raise  # обязательный re-raise
```

---

## 9. Async-итераторы и генераторы

### Async-итератор (класс)

```python
class AsyncRange:
    def __init__(self, n: int):
        self.n = n
        self.i = 0

    def __aiter__(self):
        return self

    async def __anext__(self):
        if self.i >= self.n:
            raise StopAsyncIteration
        await asyncio.sleep(0.1)
        val = self.i
        self.i += 1
        return val

async def main():
    async for x in AsyncRange(5):
        print(x)  # 0 1 2 3 4 (с паузой ~0.1s)
```

### Async-генератор

```python
async def read_slow_stream(url: str):
    """Async-генератор — читает страницу порциями"""
    async with httpx.AsyncClient() as client:
        async with client.stream("GET", url) as response:
            async for chunk in response.aiter_bytes():
                yield chunk  # yield внутри async def = async generator
                # yield возвращает значение, await приостанавливает

# Использование
async def process():
    async for chunk in read_slow_stream("https://example.com"):
        print(f"Got {len(chunk)} bytes")
```

### Async list comprehension

```python
# Python 3.6+
results = [await fetch(i) async for i in async_range(10)]

# Python 3.11+ — async for в list comprehension без дополнительных скобок
```

---

## 10. Async subprocess

```python
import asyncio

async def run_cmd(cmd: str, timeout: float = 10.0) -> str:
    """Запустить команду, получить stdout (с таймаутом)"""
    proc = await asyncio.create_subprocess_shell(
        cmd,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )

    try:
        stdout, stderr = await asyncio.wait_for(
            proc.communicate(), timeout=timeout,
        )
    except asyncio.TimeoutError:
        proc.kill()
        raise TimeoutError(f"Command timed out: {cmd}")

    if proc.returncode != 0:
        raise RuntimeError(f"Command failed: {stderr.decode()}")
    return stdout.decode()

async def main():
    output = await run_cmd("echo Hello, World!")
    print(output)  # "Hello, World!\n"
```

---

## 11. Отладка async-кода

### Включить режим отладки

```python
import asyncio

# Глобально
asyncio.run(main(), debug=True)

# Через loop
loop = asyncio.new_event_loop()
loop.set_debug(True)
loop.run_until_complete(main())

# Окружение
# PYTHONASYNCIODEBUG=1 python my_script.py
```

### «Забытый await» — предупреждение

```python
import warnings

# Раньше: создал корутину, но не await — warning
async def forget():
    coro = some_async()  # ❌ не await
    # RuntimeWarning: coroutine was never awaited
```

### asyncio.all_tasks — просмотр активных задач

```python
async def debug_tasks():
    tasks = asyncio.all_tasks(asyncio.get_running_loop())
    for t in tasks:
        print(f"Task: {t.get_name()}, done={t.done()}, cancelled={t.cancelled()}")
        try:
            print(f"  Exception: {t.exception()}")
        except asyncio.InvalidStateError:
            pass
```

### timeout для всей программы

```python
async def main():
    try:
        await asyncio.wait_for(
            asyncio.gather(task1, task2, task3),
            timeout=30.0,  # общий таймаут
        )
    except asyncio.TimeoutError:
        print("Overall timeout — force stop")
        # Отмена оставшихся
        for t in asyncio.all_tasks():
            t.cancel()
```

---

## 12. FastAPI + async — интеграционные паттерны

### Пул соединений (connection pool)

```python
from sqlalchemy.ext.asyncio import (
    create_async_engine, async_sessionmaker, AsyncSession
)

class DatabasePool:
    """Пул асинхронных соединений с lifecycle"""

    def __init__(self, url: str, pool_size: int = 5, max_overflow: int = 10):
        self.engine = create_async_engine(
            url,
            pool_size=pool_size,
            max_overflow=max_overflow,
            pool_pre_ping=True,
            echo_pool=True,
        )
        self.session_factory = async_sessionmaker(
            self.engine, expire_on_commit=False,
        )

    async def get_session(self) -> AsyncSession:
        async with self.session_factory() as session:
            yield session

    async def close(self):
        await self.engine.dispose()
```

### Graceful shutdown

```python
from contextlib import asynccontextmanager
import signal

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    db = DatabasePool(DATABASE_URL)
    app.state.db = db
    kafka = AIOKafkaProducer(bootstrap_servers=KAFKA_URL)
    await kafka.start()
    app.state.kafka = kafka

    yield  # приложение работает

    # Shutdown (graceful)
    logger.info("Shutting down...")
    app.state.kafka.flush()  # дождаться отправки
    await app.state.kafka.stop()
    await db.close()
    logger.info("Shutdown complete")

app = FastAPI(lifespan=lifespan)
```

### Streaming response

```python
from fastapi.responses import StreamingResponse
from typing import AsyncGenerator

async def generate_report() -> AsyncGenerator[bytes, None]:
    """Генерация большого отчёта стримом (без памяти)"""
    async for row in db.fetch_stream("SELECT * FROM large_table"):
        yield f"{row}\n".encode()

@app.get("/api/report")
async def get_report():
    return StreamingResponse(
        generate_report(),
        media_type="text/csv",
        headers={"Content-Disposition": "attachment; filename=report.csv"},
    )
```

### WebSocket

```python
from fastapi import WebSocket

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_text()
            # Асинхронная обработка
            result = await process_message(data)
            await websocket.send_json(result)
    except WebSocketDisconnect:
        logger.info("Client disconnected")
```

---

## 13. Retry с экспоненциальной задержкой (async)

```python
import asyncio
import functools
from typing import Awaitable, Callable, TypeVar

T = TypeVar("T")

def async_retry(
    max_attempts: int = 3,
    base_delay: float = 0.1,
    max_delay: float = 5.0,
    exponential_base: float = 2.0,
    retryable_exceptions: tuple = (Exception,),
):
    """Декоратор с exponential backoff для async функций"""
    def decorator(func: Callable[..., Awaitable[T]]) -> Callable[..., Awaitable[T]]:
        @functools.wraps(func)
        async def wrapper(*args, **kwargs) -> T:
            last_exception = None
            for attempt in range(1, max_attempts + 1):
                try:
                    return await func(*args, **kwargs)
                except retryable_exceptions as e:
                    last_exception = e
                    if attempt == max_attempts:
                        raise

                    # Вычисляем задержку: base * exp^(attempt-1) + jitter
                    delay = min(
                        base_delay * (exponential_base ** (attempt - 1)),
                        max_delay,
                    )
                    jitter = delay * 0.1 * (hash(str(e)) % 20 / 10)  # небольшой random
                    total_delay = delay + jitter

                    logger.warning(
                        f"Attempt {attempt}/{max_attempts} failed: {e}. "
                        f"Retrying in {total_delay:.2f}s"
                    )
                    await asyncio.sleep(total_delay)

            raise last_exception  # noqa
        return wrapper
    return decorator


# Использование
@async_retry(max_attempts=3, base_delay=0.5)
async def fetch_unstable_api(url: str) -> dict:
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, timeout=3.0)
        resp.raise_for_status()
        return resp.json()
```

---

## 14. Threading — когда он всё же нужен

```python
import threading
import time
from queue import Queue

# Ситуация: библиотека блокирующая (не asyncio, не выпускает GIL)
# Пример: Pillow, OpenCV, некоторые драйверы

def worker(input_queue: Queue, output_queue: Queue):
    while True:
        item = input_queue.get()
        if item is None:
            break
        # Блокирующая операция (но GIL отпускается на C-вызовах)
        result = pillow_process(item)
        output_queue.put(result)

# Запуск потоков
in_q = Queue()
out_q = Queue()
threads = [threading.Thread(target=worker, args=(in_q, out_q)) for _ in range(4)]
for t in threads:
    t.start()
```

### ThreadPoolExecutor + asyncio

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

async def main():
    loop = asyncio.get_running_loop()
    with ThreadPoolExecutor(max_workers=4) as pool:
        # Запустить 10 задач в thread pool
        tasks = [
            loop.run_in_executor(pool, sync_io_operation, i)
            for i in range(10)
        ]
        results = await asyncio.gather(*tasks)
```

---

## 15. Multiprocessing + asyncio

```python
import asyncio
from multiprocessing import Process, Queue as MPQueue
import time

def worker_process(input_queue: MPQueue, output_queue: MPQueue):
    """CPU-bound работа в отдельном процессе (свои GIL, своя память)"""
    while True:
        data = input_queue.get()
        if data is None:
            break
        # Много CPU
        result = expensive_calculation(data)
        output_queue.put(result)

async def main():
    input_queue = MPQueue()
    output_queue = MPQueue()

    proc = Process(target=worker_process, args=(input_queue, output_queue))
    proc.start()

    # Асинхронно отправляем данные
    for i in range(10):
        await asyncio.to_thread(input_queue.put, i)  # Python 3.9+
        # Или loop.run_in_executor

    # Читаем результаты
    for _ in range(10):
        result = await asyncio.to_thread(output_queue.get)
        print(result)

    input_queue.put(None)
    proc.join()
```

---

## 16. Таблица: все инструменты конкурентности

| Инструмент | Импорт | I/O-bound | CPU-bound | Потоков/Процессов | GIL |
|---|---|---|---|---|---|
| **asyncio** | `import asyncio` | ✅ | ❌ | 1 | не блокирует I/O |
| **threading** | `from threading import Thread` | ✅ | ❌ | N | GIL на Python-код |
| **multiprocessing** | `from multiprocessing import Pool` | ✅ | ✅ | N | свой GIL на процесс |
| **concurrent.futures** | `from concurrent.futures import ...` | ✅ (TPE) | ✅ (PPE) | N | зависит |
| **asyncio + run_in_executor** | `loop.run_in_executor(...)` | ✅ | ✅ | N | executor решает |

---

## 17. Подводные камни (подробно)

### Камень 1. Создал корутину — забыл await

```python
async def main():
    # ❌ Корутина создана, но не выполнена
    task = fetch_user(1)
    # → RuntimeWarning: coroutine was never awaited

    # ✅ Способы выполнить
    result = await task                    # 1. Await
    task = asyncio.create_task(fetch(1))   # 2. Запланировать
    await asyncio.gather(fetch(1))         # 3. Gather
```

### Камень 2. Блокирующий код внутри async def

```python
async def bad():
    time.sleep(1)    # ❌ БЛОКИРУЕТ ВЕСЬ EVENT LOOP!
    requests.get(url)  # ❌ БЛОКИРУЕТ!

async def good():
    await asyncio.sleep(1)                # ✅
    async with httpx.AsyncClient() as c:   # ✅
        await c.get(url)
    # Или run_in_executor:
    loop = asyncio.get_running_loop()
    data = await loop.run_in_executor(None, requests.get, url)
```

### Камень 3. gather без return_exceptions

```python
async def main():
    tasks = [will_fail(), will_succeed()]

    # ❌ Ошибка в will_fail убивает will_succeed!
    results = await asyncio.gather(*tasks)

    # ✅ return_exceptions сохраняет результаты
    results = await asyncio.gather(*tasks, return_exceptions=True)
```

### Камень 4. asyncio.run() внутри async def

```python
async def main():
    # ❌ asyncio.run() нельзя вызывать из async
    result = asyncio.run(other())  # RuntimeError!

    # ✅
    result = await other()
```

### Камень 5. Замыкание в лямбдах async

```python
# ❌ Late binding
tasks = [asyncio.create_task(fetch(i)) for i in range(10)]

# ✅ Если fetch использует i внутри — она захватится по ссылке
# fetch(i) — passé par valeur (i передаётся как аргумент) — ОК
```

### Камень 6. Event loop в разных потоках

```python
import asyncio
import threading

def thread_fn():
    # В новом потоке нет event loop!
    asyncio.run(main())  # Ошибка: no running event loop

    # Решение: создать loop в потоке
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    loop.run_until_complete(main())
```

### Камень 7. CancelledError

```python
async def worker():
    try:
        await long_task()
    except asyncio.CancelledError:
        # ❌ Не забудь re-raise или cleanup
        await cleanup()
        raise  # обязательно!
```

---

## 18. Вопросы на собесе

> **Q:** Чем asyncio отличается от threading?
> **A:** asyncio — кооперативная многозадачность (корутина сама отдаёт управление
> через await). threading — вытесняющая (ОС сама переключает потоки). asyncio
> легче (корутина ~KB, поток ~MB). asyncio обходит GIL для I/O. threading
> нужен для блокирующих библиотек.

> **Q:** Что такое GIL?
> **A:** Global Interpreter Lock — блокировка CPython, разрешающая исполнять
> байткод только одному потоку. Защищает внутренние структуры, но мешает
> CPU-bound параллелизму. I/O-bound (asyncio) — не страдает.

> **Q:** Как сделать retry в async-коде?
> **A:** Декоратор с exponential backoff: `await asyncio.sleep(delay)`
> между попытками. `asyncio.wait_for` для таймаута.

> **Q:** Что такое Structured Concurrency (TaskGroup)?
> **A:** Гарантия, что если одна задача упала — остальные отменяются.
> Аналог structured concurrency из trio. Python 3.11+, asyncio.TaskGroup.

> **Q:** Как ограничить количество одновременных запросов?
> **A:** asyncio.Semaphore(N). async with sem: await request().

> **Q:** Что будет, если в async def сделать time.sleep(10)?
> **A:** Весь event loop остановится на 10 секунд — никто не обработает
> другие запросы. Использовать await asyncio.sleep() или run_in_executor.

---

> **Технически:** asyncio — библиотека для конкурентного I/O на основе
> событийного цикла и корутин. async/await — синтаксис (PEP 492, Python 3.5).
> Event loop управляет переключением между корутинами. await приостанавливает
> текущую корутину до получения результата. GIL не блокирует I/O —
> поэтому asyncio эффективен для сетевых приложений.
>
> **На собесе:** «Объясните, как работает event loop» — «Event loop — это
> бесконечный цикл, который: 1) проверяет готовые корутины (с результатом),
> 2) запускает их, 3) ждёт I/O-события (socket ready, timer, signal).
> await — точка приостановки корутины: она говорит loop 'возобнови меня,
> когда будет готов результат'. После await loop переключается на другую
> корутину. Никакие потоки не нужны — всё в одном потоке.»