# FastAPI: Routing, Dependency Injection, Middleware — глубоко

> **Цель:** понять, как FastAPI работает под капотом (ASGI, Starlette, Pydantic),
> и осознанно использовать Depends, middleware, lifespan, exception handlers.

---

## 1. Как FastAPI обрабатывает запрос (упрощённо)

```
HTTP-запрос → Uvicorn (ASGI-сервер) → Starlette (ASGI-фреймворк) → FastAPI (Starlette + дополнения)
    ↓
1. Парсинг URL → Path параметры (из маршрута)
2. Парсинг Query параметров (из ?key=val)
3. Парсинг тела запроса → Pydantic-валидация
4. Генерация Depends (рекурсивно)
5. Вызов эндпоинта
6. Сериализация ответа → Pydantic response_model
7. HTTP-ответ
```

Каждый шаг — это **конвейер**, в который можно вставить свою логику (middleware, depends, exception handlers).

---

## 2. Dependency Injection — почему это важно

### Без DI (как в Flask)

```python
# Flask-style: эндпоинт сам создаёт зависимости
@app.get("/users/{user_id}")
def get_user(user_id: int):
    db = get_db()          # ❌ создаёт внутри
    user = db.query(...)
    return user
```

**Проблемы:**
- Нельзя подменить БД на тестовую без monkey-patching
- Нельзя легко проверить эндпоинт изолированно
- db создаётся даже в healthcheck-endpoint

### С DI (FastAPI)

```python
# FastAPI-style: зависимости приходят извне
@app.get("/users/{user_id}")
def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db),  # ✅ DI
):
    user = await db.get(User, user_id)
    return user
```

**Преимущества:**
- Тесты: `app.dependency_overrides[get_db] = mock_db`
- Переиспользование: одна `get_db` на все эндпоинты
- Изоляция: эндпоинт не знает, как создаётся db

### Depends может быть функцией, классом, генератором

```python
# Функция
def get_settings() -> Settings:
    return Settings()

# Генератор (для ресурсов с cleanup)
async def get_db():
    async with sessionmaker() as session:
        yield session

# Класс
class Pagination:
    def __init__(self, page: int = Query(1, ge=1), size: int = Query(20, ge=1, le=100)):
        self.page = page
        self.size = size
        self.offset = (page - 1) * size

@app.get("/users")
async def list_users(pagination: Pagination = Depends()):
    ...
```

### Кэширование Depends

По умолчанию каждый `Depends()` — новый вызов. Если зависимость "тяжёлая" (например, чтение конфига), кэшируйте:

```python
from functools import lru_cache

@lru_cache
def get_settings() -> Settings:
    return Settings()

@app.get("/config")
async def config(settings: Settings = Depends(get_settings)):
    # get_settings() вызовется 1 раз, потом — кэш
    ...
```

---

## 3. Middleware — обработка до и после запроса

```python
@app.middleware("http")
async def process_time_header(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)  # запрос идёт дальше
    elapsed = time.perf_counter() - start
    response.headers["X-Process-Time"] = f"{elapsed:.4f}"
    return response
```

**Порядок middleware:** первый добавленный — первый выполняется **до** call_next, последний — первый выполняется **после** call_next.

```python
app.add_middleware(CORSMiddleware, ...)
app.add_middleware(TrustedHostMiddleware, ...)

# Порядок:
# Request → CORSMiddleware → TrustedHostMiddleware → эндпоинт
# Response → TrustedHostMiddleware → CORSMiddleware → клиент
```

### CORS — что это и зачем

CORS (Cross-Origin Resource Sharing) — механизм браузера, который блокирует запросы с другого домена. FastAPI middleware добавляет заголовки `Access-Control-Allow-Origin`.

**Когда нужно:** фронтенд на `frontend.example.com` обращается к API на `api.example.com`.

**Когда не нужно:** мобильное приложение, server-to-server, Postman.

---

## 4. Exception Handlers — централизованная обработка ошибок

```python
class AppError(Exception):
    def __init__(self, code: str, message: str, status: int = 400):
        self.code = code
        self.message = message
        self.status = status

@app.exception_handler(AppError)
async def app_error_handler(request: Request, exc: AppError):
    return JSONResponse(
        status_code=exc.status,
        content={"error": exc.code, "detail": exc.message},
    )
```

**Почему это лучше, чем try/except в каждом эндпоинте:**
- Единый формат ошибок в ответе
- Логирование в одном месте
- Не нужно дублировать обработку

---

## 5. Lifespan — startup и shutdown

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    engine = create_async_engine(DATABASE_URL)
    app.state.db = engine
    logger.info("Application started")
    
    yield  # ← приложение работает
    
    # Shutdown
    logger.info("Shutting down...")
    await engine.dispose()
    logger.info("Shutdown complete")
```

**Что класть в lifespan:**
- DB engine (создать/закрыть)
- Redis connection
- Kafka producer (start/stop)
- RabbitMQ connection

**Что НЕ класть:** настройки (Settings), которые не требуют cleanup — их лучше через Depends.

---

> **На собесе:** «Расскажите про Dependency Injection в FastAPI» —  
> «FastAPI использует Depends() для внедрения зависимостей. Эндпоинт объявляет,
> что ему нужна БД-сессия, а FastAPI сам создаёт и передаёт её. В тестах
> можно переопределить зависимость через app.dependency_overrides.
> Depends поддерживает кэширование через @lru_cache и cleanup через генераторы.»