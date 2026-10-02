# FastAPI: Routing, Dependency Injection, Middleware

---

## База: эндпоинты и Depends

```python
from fastapi import FastAPI, Depends, HTTPException, Query, Path
from fastapi import status as http_status
from pydantic import BaseModel, Field

app = FastAPI(title="User Service", version="1.0.0")


# --- Dependency ---

def get_current_user(token: str = Depends(HTTPBearer())):
    if token.credentials != "valid":
        raise HTTPException(status_code=401)
    return {"user_id": 1, "role": "admin"}


# --- Эндпоинт ---

@app.get("/api/users/{user_id}", response_model=UserResponse)
def get_user(
    user_id: int = Path(ge=1),
    db: AsyncSession = Depends(get_db),
    current_user: dict = Depends(get_current_user),
):
    ...
```

### Path и Query параметры

```python
@app.get("/api/items")
def list_items(
    category: str | None = Query(default=None, max_length=50),
    page: int = Query(default=1, ge=1),
    size: int = Query(default=20, ge=1, le=100),
):
    ...
```

---

## Middleware

```python
import time
from fastapi import Request

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    elapsed = time.perf_counter() - start
    response.headers["X-Process-Time"] = f"{elapsed:.4f}"
    return response


# CORSMiddleware
from fastapi.middleware.cors import CORSMiddleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://frontend.example.com"],
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)

# TrustedHostMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
app.add_middleware(TrustedHostMiddleware, allowed_hosts=["api.example.com"])
```

---

## Exception Handlers

```python
from fastapi import Request
from fastapi.responses import JSONResponse

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

@app.exception_handler(ValidationError)
async def validation_handler(request: Request, exc: ValidationError):
    return JSONResponse(status_code=422, content={"detail": str(exc)})
```

---

## Background Tasks

```python
from fastapi import BackgroundTasks

def send_notification(user_id: int):
    """Фоновая задача — не await внутри"""
    ...

@app.post("/api/orders")
async def create_order(
    payload: OrderCreate,
    tasks: BackgroundTasks,
    db: AsyncSession = Depends(get_db),
):
    order = await create_order_in_db(db, payload)
    tasks.add_task(send_notification, order.user_id)
    return order
```

---

## Lifespan (startup/shutdown)

```python
from contextlib import asynccontextmanager
from sqlalchemy.ext.asyncio import create_async_engine

@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup
    engine = create_async_engine(DATABASE_URL)
    app.state.db = engine
    yield
    # shutdown
    await engine.dispose()

app = FastAPI(lifespan=lifespan)
```

---

## Подводные камни

| ❌ Ошибка | ✅ Правильно |
|---|---|
| `async def` для эндпоинта без I/O | Избыточно — обычный `def` быстрее |
| Depends() каждый раз новый объект | Для кэша — `@lru_cache` на зависимость |
| Путь: `@app.get("/users/{id}")` перекроет `/users/me` | Порядок важен! `/users/me` выше |
| Не обрабатывать HTTPException | Всегда raise через `HTTPException(status_code, detail)` |

---

> **Технически:** FastAPI — ASGI-фреймворк на Starlette + Pydantic.
> Depends — внедрение зависимостей (может быть функция, класс, генератор).
> Support async и sync эндпоинты (sync — в thread pool).