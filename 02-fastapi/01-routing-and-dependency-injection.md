# FastAPI: Routing, Dependency Injection, Middleware — глубоко

> **Цель:** понять, как FastAPI работает под капотом (ASGI, Starlette, Pydantic), и осознанно использовать Depends, middleware, lifespan, exception handlers.

---

## 1. Как FastAPI обрабатывает запрос (ASGI-конвейер)

```
HTTP-запрос (TCP)
  ↓
Uvicorn (ASGI-сервер: uvloop + httptools)
  ↓
Starlette (ASGI-фреймворк: routing, middleware, static files)
  ↓
FastAPI (дополнения: Depends, Pydantic-валидация, OpenAPI)
  ↓
1. Парсинг URL → Path параметры (из шаблона маршрута)
2. Парсинг Query параметров (?key=val)
3. Парсинг тела запроса → десериализация → Pydantic-валидация (422 при ошибке)
4. Разрешение Depends (рекурсивно — зависимость может иметь свои зависимости)
5. Вызов эндпоинта (sync или async)
6. Сериализация ответа через response_model (Pydantic)
7. HTTP-ответ
```

Каждый шаг — точка, куда можно вставить свою логику.

---

## 2. Dependency Injection — почему, как, когда

### Проблема без DI (Flask-style)

```python
@app.get("/users/{user_id}")
def get_user(user_id: int):
    db = get_db()           # ❌ эндпоинт создаёт зависимость сам
    cache = get_cache()     # ❌ нельзя подменить в тестах
    user = db.query(User).get(user_id)
    return user
```

**Проблемы:**
- Нельзя подменить БД на in-memory SQLite без monkey-patching
- Нельзя протестировать эндпоинт изолированно
- DB-сессия создаётся даже в healthcheck

### Решение: Depends()

```python
@app.get("/users/{user_id}")
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db),         # ← DI: приходит извне
    cache: Redis = Depends(get_cache),          # ← DI
):
    user = await db.get(User, user_id)
    return user
```

### Depends может быть трёх видов

```python
# 1. Функция (простая зависимость без cleanup)
def get_settings() -> Settings:
    return Settings()

# 2. Генератор (для ресурсов с cleanup — БД, Redis, файлы)
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as session:
        yield session   # эндпоинт работает...
    # код после yield = cleanup (всегда выполняется)

# 3. Класс (зависимость с параметрами)
class Pagination:
    def __init__(
        self,
        page: int = Query(1, ge=1, description="Номер страницы"),
        size: int = Query(20, ge=1, le=100, description="Размер страницы"),
    ):
        self.page = page
        self.size = size
        self.offset = (page - 1) * size

@app.get("/users")
async def list_users(pagination: Pagination = Depends()):  # Depends() без аргументов!
    return await repo.list(offset=pagination.offset, limit=pagination.size)
```

### Кэширование зависимостей

```python
from functools import lru_cache

@lru_cache(maxsize=1)  # один раз прочитали settings — дальше из кэша
def get_settings() -> Settings:
    return Settings(_env_file=".env")

@app.get("/config")
async def config(settings: Settings = Depends(get_settings)):
    # get_settings() вызвалась только для первого запроса
    return {"debug": settings.DEBUG}
```

### Цепочки зависимостей

```python
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as s:
        yield s

async def get_user_repo(db: AsyncSession = Depends(get_db)) -> UserRepository:
    return UserRepository(db)

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    repo: UserRepository = Depends(get_user_repo),
) -> User:
    payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
    user = await repo.get_by_id(int(payload["sub"]))
    if not user:
        raise HTTPException(status_code=401)
    return user

@app.get("/me")
async def me(current_user: User = Depends(get_current_user)):
    return current_user
```

---

## 3. Middleware — до и после каждого запроса

### Базовый middleware

```python
import time

@app.middleware("http")
async def add_process_time(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)  # ← запрос уходит дальше по конвейеру
    elapsed = time.perf_counter() - start
    response.headers["X-Process-Time"] = f"{elapsed:.4f}s"
    return response
```

### Порядок middleware

```python
# Порядок добавления важен!
app.add_middleware(CORSMiddleware, allow_origins=["*"], ...)
app.add_middleware(TrustedHostMiddleware, allowed_hosts=["example.com"])
app.add_middleware(GZipMiddleware, minimum_size=1000)

# Request:  CORSMiddleware → TrustedHost → GZip → эндпоинт
# Response: GZip → TrustedHost → CORSMiddleware → клиент
```

### CORS — что это и когда нужно

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://frontend.example.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)
```

**Нужен когда:** фронтенд на другом домене обращается к API (браузер блокирует cross-origin запросы).

**Не нужен когда:** server-to-server, мобильное приложение, Postman/curl.

### Кастомный middleware (логирование, метрики)

```python
@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    # До запроса
    request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
    logger.info("Request started", extra={
        "request_id": request_id,
        "method": request.method,
        "path": request.url.path,
    })

    try:
        response = await call_next(request)
    except Exception:
        logger.exception("Request failed", extra={"request_id": request_id})
        raise

    # После запроса
    logger.info("Request completed", extra={
        "request_id": request_id,
        "status": response.status_code,
    })
    response.headers["X-Request-ID"] = request_id
    return response
```

---

## 4. Exception Handlers — централизованная обработка

```python
class AppError(Exception):
    def __init__(self, code: str, message: str, status_code: int = 400):
        self.code = code
        self.message = message
        self.status_code = status_code

class NotFoundError(AppError):
    def __init__(self, entity: str, id: int):
        super().__init__(
            code=f"{entity.upper()}_NOT_FOUND",
            message=f"{entity} with id={id} not found",
            status_code=404,
        )

class UnauthorizedError(AppError):
    def __init__(self, reason: str = "Invalid credentials"):
        super().__init__(code="UNAUTHORIZED", message=reason, status_code=401)

# Обработчик
@app.exception_handler(AppError)
async def app_error_handler(request: Request, exc: AppError):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": {"code": exc.code, "message": exc.message}
        },
    )

# Валидация Pydantic (422) — FastAPI обрабатывает автоматически
# HTTPException — тоже автоматически

# Использование в эндпоинте:
@app.get("/users/{user_id}")
async def get_user(user_id: int, repo: UserRepo = Depends(get_user_repo)):
    user = await repo.get_by_id(user_id)
    if not user:
        raise NotFoundError("User", user_id)
    return user
```

---

## 5. Lifespan — startup и shutdown

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # === STARTUP ===
    engine = create_async_engine(DATABASE_URL, pool_size=5, max_overflow=10)
    app.state.engine = engine

    redis = aioredis.Redis(host="localhost", decode_responses=True)
    app.state.redis = redis

    logger.info("Application started", extra={"db": DATABASE_URL})

    yield  # ← приложение работает

    # === SHUTDOWN ===
    logger.info("Shutting down...")
    await engine.dispose()
    await redis.aclose()
    logger.info("Shutdown complete")

app = FastAPI(lifespan=lifespan)

# Доступ к ресурсам в эндпоинтах:
@app.get("/health")
async def health(request: Request):
    engine = request.app.state.engine
    # ping БД...
    return {"status": "ok"}
```

**Что в lifespan:** engine, Redis, RabbitMQ connection, Kafka producer.

**Что НЕ в lifespan (через Depends):** Settings, репозитории, сессии БД.

---

## 6. BackgroundTasks

```python
from fastapi import BackgroundTasks

@app.post("/orders")
async def create_order(
    payload: OrderCreate,
    background_tasks: BackgroundTasks,
    db: AsyncSession = Depends(get_db),
):
    order = await create_order_in_db(db, payload)
    # Добавляем фоновую задачу — выполнится ПОСЛЕ отправки ответа
    background_tasks.add_task(send_confirmation_email, order.email, order.id)
    background_tasks.add_task(invalidate_cache, f"user:{order.user_id}")
    return order

# ⚠️ BackgroundTasks — для быстрых задач (логирование, email).
# Для долгих — RabbitMQ / Celery / Kafka.
```

---

## 7. Кастомный роутер с префиксом и тегами

```python
from fastapi import APIRouter

router = APIRouter(
    prefix="/api/v1/users",
    tags=["users"],
    responses={404: {"description": "User not found"}},
)

@router.get("/")
async def list_users(
    pagination: Pagination = Depends(),
    repo: UserRepo = Depends(get_user_repo),
):
    return await repo.list(pagination.offset, pagination.size)

@router.get("/{user_id}")
async def get_user(user_id: int, repo: UserRepo = Depends(get_user_repo)):
    user = await repo.get_by_id(user_id)
    if not user:
        raise NotFoundError("User", user_id)
    return user

app.include_router(router)
```

---

> **На собесе:** «Расскажите про Dependency Injection в FastAPI» —
> «FastAPI использует `Depends()` для DI. Зависимости объявляются в параметрах — FastAPI рекурсивно разрешает их перед вызовом эндпоинта. Поддерживаются функции, генераторы (с cleanup) и классы. В тестах — `app.dependency_overrides` для подмены. Кэширование — `@lru_cache` на фабрике. Lifespan — для ресурсов уровня приложения.»