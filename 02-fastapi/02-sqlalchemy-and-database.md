# FastAPI + SQLAlchemy 2.0 — async, N+1, транзакции, Alembic

> **Цель:** уверенно работать с async SQLAlchemy в FastAPI, избегать N+1, управлять транзакциями и connection pool.

---

## 1. Async engine и session — как это устроено

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost:5432/db"

engine = create_async_engine(
    DATABASE_URL,
    pool_size=5,         # постоянно открытых соединений
    max_overflow=10,     # доп. соединений при пике (итого до 15)
    pool_pre_ping=True,  # проверять живое ли соединение перед запросом
    pool_recycle=3600,   # пересоздавать соединение каждый час
    echo_pool=False,     # "debug" logic в прод — лучше логировать отдельно
)

async_session = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,  # объекты остаются "живыми" после commit
)
```

**`pool_size=5, max_overflow=10`:**
- 5 соединений всегда открыты
- При пике — ещё до 10 (итого 15)
- После пика лишние закрываются

**`expire_on_commit=False`:**
- По умолчанию после `commit()` поля объекта становятся "expired" — любое обращение к ним делает ещё один запрос
- `False` — в бэкенде почти всегда ставим, чтобы не плодить лишние запросы

---

## 2. Models (SQLAlchemy 2.0 style)

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship
from sqlalchemy import String, Integer, Float, ForeignKey, DateTime, func
from datetime import datetime

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    category_id: Mapped[int | None] = mapped_column(ForeignKey("categories.id"))
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )

    # Relationships
    category: Mapped["Category | None"] = relationship(back_populates="users")
    orders: Mapped[list["Order"]] = relationship(back_populates="user")

class Category(Base):
    __tablename__ = "categories"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True)
    users: Mapped[list["User"]] = relationship(back_populates="category")

class Order(Base):
    __tablename__ = "orders"
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    total: Mapped[float] = mapped_column(Float)
    user: Mapped["User"] = relationship(back_populates="orders")
    items: Mapped[list["OrderItem"]] = relationship(back_populates="order")

class OrderItem(Base):
    __tablename__ = "order_items"
    id: Mapped[int] = mapped_column(primary_key=True)
    order_id: Mapped[int] = mapped_column(ForeignKey("orders.id"))
    product_name: Mapped[str] = mapped_column(String(200))
    quantity: Mapped[int] = mapped_column(Integer)
    price: Mapped[float] = mapped_column(Float)
    order: Mapped["Order"] = relationship(back_populates="items")
```

---

## 3. Repository Pattern

```python
from typing import Generic, TypeVar

T = TypeVar("T")

class BaseRepository(Generic[T]):
    model: type[T]

    def __init__(self, session: AsyncSession):
        self.session = session

    async def get_by_id(self, id: int) -> T | None:
        return await self.session.get(self.model, id)

    async def list(self, offset: int = 0, limit: int = 20) -> list[T]:
        stmt = select(self.model).offset(offset).limit(limit)
        result = await self.session.execute(stmt)
        return list(result.scalars().all())

    async def save(self, entity: T) -> T:
        self.session.add(entity)
        await self.session.commit()
        await self.session.refresh(entity)
        return entity

    async def delete(self, id: int) -> bool:
        entity = await self.get_by_id(id)
        if not entity:
            return False
        await self.session.delete(entity)
        await self.session.commit()
        return True

class UserRepository(BaseRepository[User]):
    model = User

    async def get_by_email(self, email: str) -> User | None:
        stmt = select(User).where(User.email == email)
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def get_with_orders(self, user_id: int) -> User | None:
        stmt = (
            select(User)
            .options(selectinload(User.orders).selectinload(Order.items))
            .where(User.id == user_id)
        )
        result = await self.session.execute(stmt)
        return result.unique().scalar_one_or_none()
```

**Почему Repository?** Эндпоинт не должен знать, как устроен запрос: ORM или raw SQL, joinedload или selectinload. Repository инкапсулирует это.

---

## 4. N+1 — полный разбор

**Проблема:** делаем 1 запрос на список → N запросов при обращении к relationship.

```python
# ❌ N+1: 1 + N запросов
users = await session.execute(select(User))
for user in users.scalars():
    print(user.category.name)  # ← ещё один запрос на КАЖДОГО пользователя!
```

**4 способа решения:**

| Способ | SQL | Когда |
|--------|-----|-------|
| `joinedload` | `LEFT JOIN` | many-to-one (User → Category) |
| `selectinload` | `WHERE id IN (...)` (второй запрос) | one-to-many (Order → Items), many-to-many |
| `subqueryload` | Подзапрос в JOIN | Устарел, лучше selectinload |
| `raiseload` | Запрещает lazy-load | Для отлова N+1 в тестах |

```python
# ✅ joinedload для many-to-one:
stmt = select(User).options(joinedload(User.category))
users = (await session.execute(stmt)).unique().scalars().all()
# 1 запрос с LEFT JOIN!

# ✅ selectinload для one-to-many:
stmt = select(Order).options(selectinload(Order.items))
orders = (await session.execute(stmt)).unique().scalars().all()
# 2 запроса: orders + items WHERE order_id IN (...)

# ✅ raiseload для тестов (ловит забытые eager-load):
stmt = select(User).options(raiseload(User.category))
# Если код обратится к user.category → InvalidRequestError!
```

---

## 5. Транзакции

```python
# Базовый паттерн:
async def transfer_money(db: AsyncSession, from_id: int, to_id: int, amount: float):
    async with db.begin():  # автоматический commit/rollback
        await db.execute(text("UPDATE accounts SET balance = balance - :a WHERE id = :f"), {"a": amount, "f": from_id})
        await db.execute(text("UPDATE accounts SET balance = balance + :a WHERE id = :t"), {"a": amount, "t": to_id})

# Savepoint (вложенная транзакция):
async with db.begin():
    user = await create_user(db, name="Alice")
    async with db.begin_nested():  # savepoint
        try:
            order = await create_order(db, user.id, total=500.0)
            if order.total > user.limit:
                raise ValueError("Limit exceeded")
        except ValueError:
            pass  # order откатился, user — нет
    # user закоммитится
```

**Уровни изоляции:**
```python
engine = create_async_engine(
    DATABASE_URL,
    isolation_level="REPEATABLE READ",  # для строгой изоляции
)
```

---

## 6. Alembic — миграции

```bash
# Инициализация
alembic init alembic

# Создать миграцию
alembic revision --autogenerate -m "add users table"

# Применить
alembic upgrade head

# Откатить на 1
alembic downgrade -1

# Посмотреть историю
alembic history
```

```python
# alembic/env.py — настройка async:
from app.models import Base
from sqlalchemy.ext.asyncio import create_async_engine

target_metadata = Base.metadata

def run_migrations_online():
    connectable = create_async_engine(DATABASE_URL)
    with connectable.connect() as connection:
        context.configure(connection=connection, target_metadata=target_metadata)
        with context.begin_transaction():
            context.run_migrations()
```

**⚠️ autogenerate не ловит всё:** переименования колонок, изменения nullable с интерпретацией — проверяй сгенерированный код вручную.

---

## 7. Диагностика connection pool

```python
# Статистика пула:
pool = engine.pool
print(f"Size: {pool.size()}, Checked in: {pool.checkedin()}, Overflow: {pool.overflow()}")

# Типичная ошибка — пул исчерпан:
# asyncpg.exceptions.TooManyConnectionsError
# → Увеличить pool_size + max_overflow или поставить таймаут
engine = create_async_engine(DATABASE_URL, pool_size=10, max_overflow=20, pool_timeout=30)
```

---

> **На собесе:** «N+1 — что это и как решаете?» —
> «N+1 возникает, когда после загрузки списка объектов мы обращаемся к relationship — каждый доступ делает отдельный запрос. Решения: `joinedload` (LEFT JOIN) для many-to-one, `selectinload` (WHERE id IN) для one-to-many. В тестах — `raiseload` чтобы отловить забытые eager-load. Также можно `apply_loads` для кастомных стратегий.»