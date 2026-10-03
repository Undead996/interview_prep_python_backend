# FastAPI: Testing and Deployment — глубоко

> **Цель:** научиться тестировать FastAPI так, чтобы тесты были быстрыми,
> изолированными и надёжными.

---

## 1. TestClient — как тестировать ASGI-приложение

```python
from fastapi.testclient import TestClient

def test_healthcheck():
    with TestClient(app) as client:
        resp = client.get("/api/health")
        assert resp.status_code == 200
        assert resp.json() == {"status": "ok"}
```

**Почему `with TestClient(app) as client:`?**
- Это включает lifespan (startup/shutdown)
- Без `with` — lifespan не выполняется, engine не создаётся

### Что тестировать

```python
def test_get_user_not_found(client):
    resp = client.get("/api/users/99999")
    assert resp.status_code == 404
    assert resp.json()["detail"] == "User not found"

def test_create_user(client):
    resp = client.post("/api/users", json={"name": "Alice", "email": "alice@ex.com"})
    assert resp.status_code == 201
    data = resp.json()
    assert data["name"] == "Alice"
    assert "id" in data
```

---

## 2. Dependency Override — изоляция тестов

```python
from main import app, get_db

@pytest.fixture
async def test_db():
    # In-memory SQLite для тестов
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

**Почему это работает:** FastAPI при вызове `Depends(get_db)` смотрит в `dependency_overrides` — если там есть замена, использует её.

---

## 3. Contract Tests — проверка OpenAPI-схемы

```python
def test_all_endpoints_have_summary(client):
    schema = client.get("/openapi.json").json()
    for path, methods in schema["paths"].items():
        for method in methods:
            assert "summary" in methods[method], f"{method.upper()} {path} missing summary"
```

**Зачем:** чтобы документация была полной, и фронтенд-разработчик знал, что делает эндпоинт.

---

## 4. Docker — multi-stage build

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

**Зачем multi-stage:** финальный образ содержит только то, что нужно для запуска — без инструментов сборки. Размер ~120MB вместо ~300MB.

---

> **На собесе:** «Как деплоите FastAPI?» —  
> «Docker + docker-compose. Uvicorn с 4 workers (через gunicorn или --workers).
> Nginx как reverse-proxy. Healthcheck на /api/health. Alembic для миграций.
> CI/CD — GitHub Actions с pytest --cov.»