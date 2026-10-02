# Шпаргалка: Python

---

## Типы / структуры

```python
int, float, str, bool, list, dict, set, tuple, None, bytes
immutable: int, str, float, bool, tuple, frozenset
mutable:   list, dict, set
```

## Async

```python
async def fetch(url): ...
async with httpx.AsyncClient() as c: ...
await asyncio.gather(t1, t2, return_exceptions=True)

asyncio.run(main())
asyncio.create_task(coro)
loop.run_in_executor(None, sync_fn)
```

## Декоратор

```python
@functools.wraps(func)
def wrapper(*args, **kwargs): ...
```

## Контекстный менеджер

```python
class Ctx:
    def __enter__(self): return self
    def __exit__(self, typ, val, tb): ...
```

## Исключения

```python
try: ...
except ValueError as e: ...
except Exception: ...
else: ...  # нет ошибки
finally: ...  # всегда
```