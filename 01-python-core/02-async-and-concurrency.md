# Async и конкурентность в Python — глубоко с внутренним устройством

> **Цель:** понять, как async/await работает под капотом на уровне Python/C,
> и обоснованно выбирать asyncio vs threading vs multiprocessing под задачу.

---

## 1. GIL — почему он есть и как его обойти

### Что такое GIL на уровне C CPython

CPython написан на C. Глобальный интерпретатор (Python-код) работает через `ceval.c` — цикл исполнения байткода. Без GIL каждое чтение/запись в `PyObject` (абстракция "питоновского объекта") требовала бы блокировки. Это замедлило бы однопоточный код на **2–4x**.

**GIL — это trade-off:**
- **+** однопоточный код быстр, C-расширения (numpy, pandas, Pillow) могут отпускать GIL и быть потокобезопасными бесплатно  
- **−** Python-потоки не дают прироста для CPU-bound кода

### Что GIL НЕ блокирует

| Операция | Блокирует? | Почему |
|----------|-----------|--------|
| Python-код | ✅ Каждый байткод под GIL | Защита структур CPython |
| `socket.recv()`, `file.read()` | ❌ GIL отпускается на I/O | I/O — вне Python, ждёт ОС |
| `time.sleep()` | ❌ GIL отпускается на сон | Таймер ядра ОС |
| `numpy.dot()` | ❌ GIL отпускается | C-расширение использует `Py_BEGIN_ALLOW_THREADS` |
| asyncio event loop | ❌ Один поток, нет GIL-проблем | Coroutine switching — кооперативный |

### Практическое правило

```python
# CPU-bound — не используй threading, используй multiprocessing
from multiprocessing import Pool

# I/O-bound — asyncio (или threading для синхронных библиотек)
import asyncio

# Смешанный — asyncio + ProcessPoolExecutor для CPU-кусков
```

---

## 2. asyncio vs threading vs multiprocessing — когда что

| Сценарий | asyncio | threading | multiprocessing |
|----------|---------|-----------|-----------------|
| Web-сервер (FastAPI) | ✅ | ❌ | ❌ |
| Парсинг 1000 сайтов | ✅ | ❌ | ❌ |
| Старая синхронная библиотека (requests) | ❌ | ✅ | ❌ |
| Обработка изображений (CPU) | ❌ | ❌ | ✅ |
| Микросервис с 10k WebSocket | ✅ | ❌ | ❌ |
| Научные расчёты | ❌ | ❌ | ✅ |

**Почему asyncio лучше threading для I/O-bound:**
- Корутина весит ~1KB, поток ~2MB (можно иметь 10k корутин, но не 10k потоков)
- Переключение корутин — 0.1μs, переключение потоков — 1μs (ОС-оверхед)
- Нет гонок за общие данные (нет Lock/RLock/Semaphore — код проще)

---

## 3. Event Loop — внутреннее устройство

### Что происходит при `await`

```python
async def fetch():
    # 1. Python создаёт объект coroutine (машина состояний)
    # 2. При await asyncio.sleep(1):
    #    a. Корутина доходит до await, вызывает `__await__`
    #    b. Создаётся Future/Task в event loop
    #    c. Event loop регистрирует таймер: через 1 секунду
    #    d. Управление возвращается event loop'у
    # 3. Event loop выбирает следующую корутину из _ready
    # 4. Через 1с event loop кладёт эту корутину обратно в _ready
    # 5. Корутина просыпается, получает результат from await
    await asyncio.sleep(1)
    return 42
```

### Типы event loop на разных ОС

| ОС | Механизм | Имплементация |
|----|----------|--------------|
| Linux | `epoll` | `SelectorEventLoop` (по умолчанию) |
| macOS | `kqueue` | `SelectorEventLoop` |
| Windows | `IOCP` | `ProactorEventLoop` (используется в asyncio.run()) |

### run_in_executor — мост между sync и async

```python
async def call_legacy_api(url: str) -> dict:
    loop = asyncio.get_running_loop()
    # Выполняет requests.get в ThreadPoolExecutor
    # GIL отпускается на время I/O, event loop не блокируется
    response = await loop.run_in_executor(
        None,  # default ThreadPoolExecutor
        requests.get, url
    )
    return response.json()
```

---

## 4. Корутины, Tasks, Futures — разница на практике

```python
coro = my_coro()                    # Coroutine (объект-генератор)
task = asyncio.create_task(coro)    # Task (планирует в event loop)
future = loop.create_future()       # Future (только результат)
```

| Объект | Что делает | Когда использовать |
|--------|-----------|-------------------|
| **Coroutine** | Вычисляется при `await` | Внутри async def, прячется за create_task |
| **Task** | Планирует, можно отменить, ждать | Для фоновых задач |
| **Future** | Низкоуровневое обещание | Для callback-based API (редко) |

```python
# Пример Future с callback
def on_done(future):
    print(future.result())

future = loop.create_future()
future.add_done_callback(on_done)
loop.call_later(1, future.set_result, 42)
```

---

## 5. Producer-Consumer с asyncio.Queue

Этот паттерн — основа любого queue-based микросервиса:

```python
async def worker(name: str, queue: asyncio.Queue):
    while True:
        item = await queue.get()
        if item is None:
            queue.task_done()
            break
        print(f"[{name}] processing {item}")
        await asyncio.sleep(0.1)
        queue.task_done()

async def main():
    queue = asyncio.Queue(maxsize=100)
    workers = [asyncio.create_task(worker(f"W{i}", queue)) for i in range(3)]
    
    for i in range(20):
        await queue.put(f"task-{i}")
    
    for _ in workers:
        await queue.put(None)  # сигнал остановки
    
    await asyncio.gather(*workers)
```

---

## 6. Типичные подводные камни (на собесе)

| ❌ Ошибка | Почему | ✅ Как правильно |
|-----------|--------|-----------------|
| `time.sleep(1)` в `async def` | Блокирует весь event loop на 1s! | `await asyncio.sleep(1)` |
| `requests.get()` в `async def` | Блокирует event loop на I/O | `httpx.AsyncClient()` или `run_in_executor` |
| `gather` без `return_exceptions` | Одна ошибка убивает все задачи | `gather(…, return_exceptions=True)` |
| `asyncio.run()` внутри `async def` | RuntimeError: другой loop запущен | `await task` |
| Создал корутину, но не `await` | RuntimeWarning, код не выполнен | `await` или `create_task` |

---

> **На собесе:** «Расскажите, как работает asyncio» —  
> «asyncio — это событийный цикл (event loop), который исполняет корутины.
> Корутина — функция, которая может приостанавливаться через `await`, отдавая
> управление event loop'у. Event loop использует epoll/kqueue/IOCP для
> неблокирующего I/O. В одном потоке могут работать тысячи корутин.
> GIL не проблема — он отпускается на время I/O.»