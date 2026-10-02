# FastAPI + SQLAlchemy 2.0

---

## SQLAlchemy 2.0 — async engine

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import String, Integer, select, text

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost:5432/db"

engine = create_async_engine(DATABASE_URL, pool_size=5, max_overflow=10)
async_session = async_sessionmaker(engine, expire_on_commit=False)


# --- Модели ---

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
```

---

## Repository pattern с async session

```python
from sqlalchemy import select, delete
from sqlalchemy.ext.asyncio import AsyncSession

class UserRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def get_by_id(self, user_id: int) -> User | None:
        stmt = select(User).where(User.id == user_id)
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def create(self, name: str, email: str) -> User:
        user = User(name=name, email=email)
        self.session.add(user)
        await self.session.commit()
        await self.session.refresh(user)
        return user

    async def list(self, skip: int = 0, limit: int = 100) -> list[User]:
        stmt = select(User).offset(skip).limit(limit)
        result = await self.session.execute(stmt)
        return list(result.scalars().all())
```

---

## Dependency Injection в FastAPI

```python
async def get_db():
    async with async_session() as session:
        yield session

async def get_user_repo(db: AsyncSession = Depends(get_db)):
    return UserRepository(db)

@app.get("/api/users/{user_id}")
async def get_user(
    user_id: int,
    repo: UserRepository = Depends(get_user_repo),
):
    user = await repo.get_by_id(user_id)
    if not user:
        raise HTTPException(status_code=404)
    return user
```

---

## N+1 проблема

```python
# ❌ N+1 — на каждую категорию отдельный запрос
users = await session.execute(select(User))
for user in users.scalars():
    print(user.category.name)  # ❌ Каждый раз запрос!

# ✅ Eager loading — joinedload или selectinload
from sqlalchemy.orm import joinedload, selectinload

stmt = select(User).options(joinedload(User.category))
result = await session.execute(stmt)
```

---

## Транзакции

```python
async def transfer_money(db: AsyncSession, from_id: int, to_id: int, amount: float):
    async with db.begin():
        # Если ошибка — обе таблицы не изменятся
        stmt1 = text("UPDATE accounts SET balance = balance - :a WHERE id = :f")
        stmt2 = text("UPDATE accounts SET balance = balance + :a WHERE id = :t")
        await db.execute(stmt1, {"a": amount, "f": from_id})
        await db.execute(stmt2, {"a": amount, "t": to_id})
```

---

## Alembic — миграции

```bash
alembic init alembic
alembic revision --autogenerate -m "add users table"
alembic upgrade head
alembic downgrade -1  # откат
```

```python
# alembic/env.py
from models import Base  # Импорт моделей для autogenerate
target_metadata = Base.metadata
```

---

## Connection pool

```python
engine = create_async_engine(
    DATABASE_URL,
    pool_size=10,       # кол-во соединений в пуле
    max_overflow=20,    # доп. соединений сверх pool_size
    pool_pre_ping=True, # проверка соединения перед запросом
    pool_recycle=1800,  # переиспользовать через 30 мин
)
```

---

## Подводные камни

| ❌ Ошибка | ✅ Правильно |
|---|---|
| `session.commit()` без `refresh()` | После commit объект detached — `await session.refresh(obj)` |
| `expire_on_commit=True` (default) | После commit — доступ к полям вызывает запрос. `False` — безопаснее |
| N+1 с async | `selectinload` для many-to-many, `joinedload` для many-to-one |
| Не закрывать engine | `await engine.dispose()` в lifespan shutdown |

---

> **Технически:** SQLAlchemy 2.0 — ORM и Core. Async требует asyncpg (для PG)
> или aiosqlite (для SQLite). Пул соединений — `QueuePool` для sync, `AsyncAdaptedQueuePool`
> для async. `expire_on_commit=False` — объект жив после commit.