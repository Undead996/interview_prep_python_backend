# FastAPI + SQLAlchemy + pytest Cheatsheet

## TestClient / AsyncClient

```python
from httpx import AsyncClient, ASGITransport
async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
    resp = await client.get("/api/health")
    resp.status_code; resp.json(); resp.headers
```

## Depends

```python
async def get_db(): async with async_session() as s: yield s
def endpoint(db=Depends(get_db)): ...

# Кэш: @lru_cache на get_settings()
# Тесты: app.dependency_overrides[get_db] = override
```

## SQLAlchemy async

```python
engine = create_async_engine(URL, pool_size=5, max_overflow=10, pool_pre_ping=True)
async_session = async_sessionmaker(engine, expire_on_commit=False)

async with async_session() as s:
    result = await s.execute(select(User).where(User.id == id))
    user = result.scalar_one_or_none()

# N+1:
stmt = select(User).options(joinedload(User.category))       # many-to-one
stmt = select(Order).options(selectinload(Order.items))      # one-to-many
stmt = select(User).options(raiseload(User.category))        # запретить lazy (тесты)
```

## Lifespan

```python
@asynccontextmanager
async def lifespan(app):
    app.state.db = create_async_engine(URL)
    yield
    await app.state.db.dispose()
```

## JWT

```python
from jose import jwt
token = jwt.encode({"sub": "1", "exp": datetime.now(tz=utc) + timedelta(min=30)}, SECRET, algorithm="HS256")
data = jwt.decode(token, SECRET, algorithms=["HS256"])
```

## Rate limiting

```python
await redis.incr(f"rl:{ip}:{int(time.time()) // 60}")
await redis.expire(f"rl:{ip}:{int(time.time()) // 60}", 61)
```

## pytest + factories

```python
# conftest.py:
@pytest_asyncio.fixture
async def client():
    engine = create_async_engine("sqlite+aiosqlite://")
    async with engine.begin() as conn: await conn.run_sync(Base.metadata.create_all)
    async_session = async_sessionmaker(engine, expire_on_commit=False)
    app.dependency_overrides[get_db] = lambda: async_session()
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as ac:
        yield ac
    await engine.dispose()

# Factory:
class UserFactory(factory.alchemy.SQLAlchemyModelFactory):
    class Meta: model = User
    name = factory.Faker("name")
    email = factory.Faker("email")
```

## Docker

```dockerfile
FROM python:3.12-slim AS builder
RUN pip install --user -r requirements.txt
FROM python:3.12-slim
COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

## Alembic

```bash
alembic init alembic
alembic revision --autogenerate -m "add users"
alembic upgrade head
alembic downgrade -1
```