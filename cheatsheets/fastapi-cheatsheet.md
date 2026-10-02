# Шпаргалка: FastAPI

---

## TestClient

```python
from fastapi.testclient import TestClient
client = TestClient(app)
resp = client.get("/api/health")
resp.status_code; resp.json(); resp.headers
```

## Dependency Override

```python
app.dependency_overrides[get_db] = lambda: test_db
# после теста:
app.dependency_overrides.clear()
```

## Depends

```python
async def get_db(): ...
def endpoint(db=Depends(get_db)): ...
```

## SQLAlchemy async

```python
engine = create_async_engine(URL, pool_size=5)
async_session = async_sessionmaker(engine)
async with async_session() as s:
    result = await s.execute(select(User))
    user = result.scalar_one_or_none()
```

## Lifespan

```python
@asynccontextmanager
async def lifespan(app):
    yield  # startup / shutdown
app = FastAPI(lifespan=lifespan)
```

## JWT

```python
from jose import jwt
token = jwt.encode({"sub": "1"}, SECRET, algorithm="HS256")
data = jwt.decode(token, SECRET, algorithms=["HS256"])
```

## Rate limit

```python
await redis.incr(f"rl:{ip}:{minute}")
await redis.expire(f"rl:{ip}:{minute}", 61)
```