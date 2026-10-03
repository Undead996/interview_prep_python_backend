# FastAPI + SQLAlchemy 2.0 — глубокий разбор

> **Цель:** понять, как работать с async SQLAlchemy в FastAPI, избегать N+1,
> управлять транзакциями и connection pool'ом.

---

## 1. Async engine и session — как это устроено

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost:5432/db"

engine = create_async_engine(
    DATABASE_URL,
    pool_size=5,         # соединений в пуле (постоянно открыты)
    max_overflow=10,     # доп. соединений сверх pool_size
    pool_pre_ping=True,  # проверять соединение перед запросом
    echo_pool=True,      # логировать создание/закрытие соединений
)
async_session = async_sessionmaker(engine, expire_on_commit=False)
```

**Что значит `pool_size=5, max_overflow=10`:**
- Держим 5 соединений всегда открытыми
- При пике — до 15 (5 + 10)
- После пика лишние закрываются

**`expire_on_commit=False`:**
- По умолчанию: после `commit()` все поля объекта становятся "expired" — при обращении к ним делается новый запрос
- `False`: объект остаётся "живым" — поля доступны без запроса
- В бэкенде почти всегда ставим `False`

---

## 2. Repository Pattern

**Почему Repository?** Потому что эндпоинт не должен знать, как устроен запрос:

```python
# ❌ Без Repository — эндпоинт знает про SQLAlchemy
@app.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    stmt = select(User).where(User.id == user_id)
    result = await db.execute(stmt)
    return result.scalar_one_or_none()

# ✅ С Repository — эндпоинт вызывает бизнес-метод
@app.get("/users/{user_id}")
async def get_user(user_id: int, repo: UserRepository = Depends(get_user_repo)):
    return await repo.get_by_id(user_id)
```

**Repository инкапсулирует:**
- Тип запроса (ORM vs raw SQL)
- Стратегию загрузки (joinedload, selectinload)
- Логику повторных попыток

---

## 3. N+1 проблема — детально

**Проблема:** делаем 1 запрос на список + N запросов на каждую строку.

```python
# ❌ N+1 — на каждую категорию отдельный запрос
users = await session.execute(select(User))
for user in users.scalars():
    print(user.category.name)  # ← ещё один запрос!
```

**Решение:** сказать SQLAlchemy, какие отношения загрузить сразу.

```python
from sqlalchemy.orm import joinedload, selectinload

# Для many-to-one (User → Category) — joinedload (LEFT JOIN)
stmt = select(User).options(joinedload(User.category))

# Для one-to-many / many-to-many (Order → Items) — selectinload (второй запрос с WHERE IN)
stmt = select(Order).options(selectinload(Order.items))
```

**Когда какой:**
| Отношение | Рекомендация | SQL |
|-----------|-------------|-----|
| Many-to-one (User.category) | `joinedload` | `LEFT JOIN` |
| One-to-many (Order.items) | `selectinload` | `WHERE order_id IN (...)` |
| Many-to-many (Student.courses) | `selectinload` | `WHERE course_id IN (...)` через join-таблицу |

---

## 4. Транзакции — как управлять

```python
async def transfer_money(db: AsyncSession, from_id: int, to_id: int, amount: float):
    # begin() — начало транзакции
    async with db.begin():
        stmt1 = text("UPDATE accounts SET balance = balance - :a WHERE id = :f")
        stmt2 = text("UPDATE accounts SET balance = balance + :a WHERE id = :t")
        await db.execute(stmt1, {"a": amount, "f": from_id})
        await db.execute(stmt2, {"a": amount, "t": to_id})
    # commit() автоматически при выходе из async with
    # rollback() — если исключение
```

**Savepoints — вложенные транзакции:**

```python
async with db.begin():
    user = await create_user(db, ...)
    # Точка сохранения
    async with db.begin_nested():
        order = await create_order(db, user.id, ...)
        if order.total > user.limit:
            # rollback только order, user остаётся
            raise ValueError("Limit exceeded")
    # user.commit() всё равно произойдёт
```

---

> **Технически:** SQLAlchemy 2.0 — ORM и Core в одном. Async требует asyncpg
> (для PostgreSQL) или aiosqlite (для SQLite). Пул соединений — `QueuePool`
> для sync, `AsyncAdaptedQueuePool` для async.