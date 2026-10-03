# Шпаргалка: FastAPI + SQLAlchemy — последний день

---

## TestClient

```python
from fastapi.testclient import TestClient
with TestClient(app) as client:
    resp = client.get("/api/health")
    resp.status_code; resp.json(); resp.headers
```

## Dependency Override

```python
app.dependency_overrides[get_db] = lambda: test_db
# cleanup:
app.dependency_overrides.clear()
```

## Depends

```python
async def get_db(): ...
def endpoint(db=Depends(get_db)): ...
# Для кэша: @lru_cache на get_db()
```

## SQLAlchemy async

```python
engine = create_async_engine(URL, pool_size=5, max_overflow=10)
async_session = async_sessionmaker(engine, expire_on_commit=False)
async with async_session() as s:
    result = await s.execute(select(User))
    user = result.scalar_one_or_none()

# N+1 — joinedload / selectinload
st = select(User).options(joinedload(User.category))
st = select(Order).options(selectinload(Order.items))
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
token = jwt.encode({"sub": "1", "exp": ...}, SECRET, algorithm="HS256")
data = jwt.decode(token, SECRET, algorithms=["HS256"])
```

## Rate limiting

```python
await redis.incr(f"rl:{ip}:{minute}")
await redis.expire(f"rl:{ip}:{minute}", 61)
```

## Helthcheck

```python
@app.get("/health")
async def health():
    return {"status": "ok"}
```