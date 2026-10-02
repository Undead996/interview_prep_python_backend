# FastAPI: Testing and Deployment

---

## TestClient

```python
import pytest
from fastapi.testclient import TestClient
from main import app


@pytest.fixture(scope="session")
def client():
    with TestClient(app) as c:
        yield c


def test_healthcheck(client):
    resp = client.get("/api/health")
    assert resp.status_code == 200
    assert resp.json() == {"status": "ok"}
```

---

## Dependency Override для тестов

```python
from main import app, get_db

@pytest.fixture
async def test_db():
    """In-memory SQLite для тестов"""
    engine = create_async_engine("sqlite+aiosqlite://", echo=True)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    async_session = async_sessionmaker(engine, expire_on_commit=False)

    async def override_get_db():
        async with async_session() as session:
            yield session

    app.dependency_overrides[get_db] = override_get_db
    yield
    app.dependency_overrides.clear()
    await engine.dispose()
```

---

## Тестируем авторизацию

```python
from main import app, get_current_user

TEST_USER = {"user_id": 1, "role": "admin"}

@pytest.fixture(autouse=True)
def override_auth():
    app.dependency_overrides[get_current_user] = lambda: TEST_USER
    yield
    app.dependency_overrides.clear()

def test_admin_endpoint(client):
    resp = client.get("/api/admin/users")
    assert resp.status_code == 200
```

---

## Dockerfile

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
version: "3.9"
services:
  app:
    build: .
    ports: ["8000:8000"]
    depends_on:
      - db
      - redis
    environment:
      DATABASE_URL: postgresql+asyncpg://user:pass@db:5432/app

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: pass
```

---

## CI/CD (GitHub Actions)

```yaml
name: FastAPI CI
on: [push]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -r requirements.txt
      - run: pytest tests/ -n auto --timeout=30 --cov=src
```

---

## Подводные камни

| ❌ Ошибка | ✅ Правильно |
|---|---|
| TestClient без lifespan | `with TestClient(app) as client:` |
| dependency_overrides не очищен | autouse фикстура с `app.dependency_overrides.clear()` |
| Uvicorn без workers | `--workers 4` — на проде |
| Не настроен healthcheck | `GET /health` — всегда нужен |

---

> **На собесе:** «Как деплоите FastAPI?» — «Docker + docker-compose. Uvicorn с gunicorn
> (--workers 4). Nginx как reverse-proxy. Healthcheck на /api/health. Alembic для миграций.»