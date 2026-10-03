# FastAPI: Testing (pytest) и Deployment — глубоко

> **Цель:** научиться тестировать FastAPI так, чтобы тесты были быстрыми, изолированными и надёжными. Писать меньше тестов, покрывать больше кода.

---

## 1. pytest + pytest-asyncio — настройка

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"  # автоматически определяет async-тесты
testpaths = ["tests"]

[tool.coverage.run]
source = ["app"]
omit = ["*/migrations/*", "*/tests/*"]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "if TYPE_CHECKING:",
    "raise NotImplementedError",
]
```

```python
# conftest.py — общие фикстуры
import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from app.main import app
from app.db import Base, get_db

TEST_DATABASE_URL = "sqlite+aiosqlite:///./test.db"

@pytest_asyncio.fixture(scope="session")
async def test_engine():
    engine = create_async_engine(TEST_DATABASE_URL, echo=False)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()

@pytest_asyncio.fixture
async def db_session(test_engine):
    async_session = async_sessionmaker(test_engine, expire_on_commit=False)
    async with async_session() as session:
        yield session
        await session.rollback()

@pytest_asyncio.fixture
async def client(db_session: AsyncSession):
    async def override_get_db():
        yield db_session
    app.dependency_overrides[get_db] = override_get_db

    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac

    app.dependency_overrides.clear()
```

---

## 2. Тесты — от простого к сложному

### Проверка healthcheck

```python
@pytest.mark.asyncio
async def test_healthcheck(client: AsyncClient):
    response = await client.get("/health")
    assert response.status_code == 200
    assert response.json() == {"status": "ok"}
```

### CRUD-тесты (состояние между тестами!)

```python
@pytest.mark.asyncio
async def test_create_and_get_user(client: AsyncClient):
    # Создать
    create_resp = await client.post(
        "/api/users",
        json={"name": "Alice", "email": "alice@test.com"},
    )
    assert create_resp.status_code == 201
    user = create_resp.json()
    assert user["name"] == "Alice"
    assert user["email"] == "alice@test.com"
    assert "id" in user

    # Получить по id
    get_resp = await client.get(f"/api/users/{user['id']}")
    assert get_resp.status_code == 200
    assert get_resp.json()["name"] == "Alice"

@pytest.mark.asyncio
async def test_get_user_not_found(client: AsyncClient):
    response = await client.get("/api/users/99999")
    assert response.status_code == 404
    assert "not found" in response.json()["detail"].lower()

@pytest.mark.asyncio
async def test_create_user_invalid_email(client: AsyncClient):
    response = await client.post(
        "/api/users",
        json={"name": "Bob", "email": "not-an-email"},
    )
    assert response.status_code == 422  # Pydantic ValidationError
    errors = response.json()["detail"]
    assert any("email" in str(e["loc"]) for e in errors)
```

### Тест с авторизацией

```python
import pytest_asyncio

@pytest_asyncio.fixture
async def auth_headers(client: AsyncClient):
    """Создаём пользователя, логинимся, возвращаем заголовки."""
    await client.post("/api/users", json={
        "name": "Test", "email": "auth@test.com", "password": "secret123",
    })
    login_resp = await client.post("/api/auth/login", data={
        "username": "auth@test.com", "password": "secret123",
    })
    token = login_resp.json()["access_token"]
    return {"Authorization": f"Bearer {token}"}

@pytest.mark.asyncio
async def test_profile_authenticated(client: AsyncClient, auth_headers: dict):
    response = await client.get("/api/me", headers=auth_headers)
    assert response.status_code == 200
    assert response.json()["email"] == "auth@test.com"

@pytest.mark.asyncio
async def test_profile_no_token(client: AsyncClient):
    response = await client.get("/api/me")
    assert response.status_code == 401
```

---

## 3. Factories — не плоди ручные данные

```python
import factory
from app.db import User

class UserFactory(factory.alchemy.SQLAlchemyModelFactory):
    class Meta:
        model = User
        sqlalchemy_session_persistence = "commit"

    name = factory.Faker("name")
    email = factory.Faker("email")
    is_active = True

# В тесте:
user1 = await UserFactory.create()  # создан и записан в БД
user2 = await UserFactory.build()   # создан, но НЕ записан
users = await UserFactory.create_batch(10)  # 10 пользователей
```

---

## 4. Mock vs Real — что мокать

**НЕ мокай:**
- Свою бизнес-логику
- Репозитории (используй in-memory SQLite)
- Pydantic-модели

**Мокай:**
- Внешние API (через `respx` или `httpx.MockTransport`)
- Email/sms/push-сервисы
- Платёжные шлюзы

```python
import respx
from httpx import Response

@respx.mock
@pytest.mark.asyncio
async def test_external_api_call(client: AsyncClient):
    # Мокаем внешний API
    respx.get("https://api.external.com/users/1").mock(
        return_value=Response(200, json={"id": 1, "name": "External User"})
    )

    response = await client.get("/api/sync-external/1")
    assert response.status_code == 200
    assert response.json()["name"] == "External User"
```

---

## 5. Contract Tests — проверка OpenAPI-схемы

```python
@pytest.mark.asyncio
async def test_all_endpoints_have_summary(client: AsyncClient):
    schema_resp = await client.get("/openapi.json")
    schema = schema_resp.json()

    errors = []
    for path, methods in schema["paths"].items():
        for method in methods:
            if "summary" not in methods[method]:
                errors.append(f"{method.upper()} {path}")

    assert not errors, f"Endpoints missing summary: {', '.join(errors)}"

@pytest.mark.asyncio
async def test_response_matches_schema(client: AsyncClient):
    """Проверить, что ответ /api/users соответствует OpenAPI-схеме."""
    schema_resp = await client.get("/openapi.json")
    schema = schema_resp.json()

    # Получаем схему ответа для GET /api/users/{user_id}
    response_schema = schema["paths"]["/api/users/{user_id}"]["get"]["responses"]["200"]

    # Можно валидировать через jsonschema.validate()
    # или просто проверить наличие нужных полей
    assert "content" in response_schema
```

---

## 6. Docker — multi-stage build

```dockerfile
# Stage 1: сборка зависимостей
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Stage 2: финальный образ (без сборочных инструментов)
FROM python:3.12-slim
WORKDIR /app

# Копируем установленные пакеты из builder
COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH

# Копируем код
COPY . .

# Не запускать от root
RUN useradd --create-home appuser && chown -R appuser:appuser /app
USER appuser

# Healthcheck
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"

EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

### docker-compose.yml для разработки

```yaml
version: "3.9"
services:
  app:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+asyncpg://user:pass@db:5432/app
      - REDIS_URL=redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    volumes:
      - .:/app  # hot-reload при разработке

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: app
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d app"]
      interval: 5s
      timeout: 3s
      retries: 5
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```

---

## 7. CI/CD — GitHub Actions

```yaml
name: Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test
        options: >-
          --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt
      - run: pytest --cov --cov-report=xml
      - run: ruff check . && mypy app/
```

---

> **На собесе:** «Как тестируете FastAPI?» —
> «Интеграционные тесты через `httpx.AsyncClient` с ASGI-transport (или TestClient для sync). БД — in-memory SQLite с `create_all`/`drop_all` в сессионной фикстуре. `dependency_overrides` для подмены. Мокаю только внешние API (respx), свою логику — никогда. `pytest --cov` с порогом ≥ 80%. Fixtures через factory_boy для тестовых данных. В CI — с реальным PostgreSQL в Docker.»