# Мок-интервью: FastAPI (20 вопросов) — расширенные ответы

---

## Q1. FastAPI vs Flask?

FastAPI — ASGI (async-native, uvicorn), автодокументация (OpenAPI/Swagger из коробки), Pydantic-валидация, DI через Depends. Flask — WSGI (sync), документация вручную. FastAPI быстрее, современнее. Flask — для legacy или очень простых проектов.

---

## Q2. Что такое Dependency Injection в FastAPI?

DI = зависимости приходят извне, эндпоинт не создаёт их сам. `Depends(get_db)` — FastAPI сам вызывает `get_db()` и передаёт результат. В тестах — `app.dependency_overrides[get_db] = mock`. Поддерживает функции, генераторы (cleanup), классы. Кэширование через `@lru_cache`.

---

## Q3. Как обрабатывать ошибки?

1. `HTTPException(status_code=404)` — стандартные
2. `@app.exception_handler(AppError)` — кастомные бизнес-ошибки
3. Pydantic `ValidationError` → FastAPI сам 422
4. Структура ответа: `{"error": {"code": "X", "message": "..."}}`

---

## Q4. Как работает lifespan?

`@asynccontextmanager`-функция: код до `yield` — startup (engine, Redis, Kafka), после `yield` — shutdown (dispose, close). Современная замена `@app.on_event("startup")`/`"shutdown"`. В lifespan — ресурсы уровня приложения (engine). Сессии — через Depends.

---

## Q5. Как протестировать FastAPI?

`httpx.AsyncClient` с `ASGITransport` (или TestClient): интеграционные тесты. `dependency_overrides` — подмена БД на in-memory SQLite. `pytest-asyncio` с `asyncio_mode="auto"`. `factory_boy` для тестовых данных. Contract tests — проверка `/openapi.json`. CI — с реальным PostgreSQL в Docker.

---

## Q6. N+1 в SQLAlchemy — решение?

`joinedload` для many-to-one (LEFT JOIN), `selectinload` для many-to-many (второй запрос WHERE id IN). В тестах — `raiseload` для отлова забытых eager-load.

---

## Q7. Как устроена аутентификация?

`OAuth2PasswordBearer` извлекает Bearer-токен. `Depends(get_current_user)` — decode JWT (jose), проверка exp, поиск пользователя по sub. bcrypt для хэширования паролей. Refresh token для продления сессии.

---

## Q8. Rate limiting — как сделать?

**In-memory:** dict[IP → list[timestamps]], sliding window. **Redis:** `INCR ratelimit:{IP}:{minute}`, `EXPIRE`, если > N → 429. Альтернатива: token bucket (Lua-скрипт для атомарности). Заголовок `Retry-After`.

---

## Q9. Что такое middleware?

Функция `request → call_next → response`, выполняемая до и после каждого запроса. Порядок: первый добавленный — первый выполняется до call_next. Для: CORS, TrustedHost, rate limiting, логирование, security headers.

---

## Q10. BackgroundTasks vs Celery/RabbitMQ?

`background_tasks.add_task()` — быстрые задачи (логирование, email), выполняются после отправки ответа. Не async. Для долгих/тяжёлых задач — RabbitMQ/Celery/Kafka (отдельный worker, retry, мониторинг).

---

## Q11. Как валидировать данные в FastAPI?

Pydantic модели: `BaseModel` с type hints. `Field(gt=0, min_length=1)` — ограничения. `model_validator` — кросс-полевая валидация. `field_serializer` — преобразование при сериализации. Вложенные модели.

---

## Q12. Что такое response_model?

Pydantic-модель для сериализации ответа. Фильтрует поля (только явно указанные), валидирует, конвертирует. `response_model_exclude={"password"}` — исключить поле. Ускоряет FastAPI (не сериализует лишнее).

---

## Q13. Как сделать pagination?

```python
class Pagination:
    def __init__(self, page: int = Query(1, ge=1), size: int = Query(20, ge=1, le=100)):
        self.offset = (page - 1) * size
        self.limit = size

@app.get("/users")
async def list_users(p: Pagination = Depends()):
    return await repo.list(p.offset, p.limit)
```

---

## Q14. Что такое CORS и зачем?

Cross-Origin Resource Sharing — механизм браузера, блокирующий запросы с другого домена. FastAPI middleware добавляет заголовки `Access-Control-Allow-Origin`. Нужен когда фронтенд на другом домене. Не нужен для server-to-server.

---

## Q15. Как работает `Depends()` с генератором?

```python
async def get_db():
    async with async_session() as session:
        yield session  # передаётся в эндпоинт
    # cleanup: сессия закрыта даже при ошибке
```

---

## Q16. Чем `Body()` отличается от `Query()`?

`Query()` — параметры из URL (?key=val), `Body()` — из тела запроса (JSON). `Body(embed=True)` — оборачивает в ключ. Можно комбинировать: path + query + body в одном эндпоинте.

---

## Q17. Что такое WebSocket в FastAPI?

`@app.websocket("/ws")` — обработчик. `await ws.receive_text()`, `await ws.send_text()`. FastAPI нативно поддерживает WebSocket через Starlette. Для масштабирования — Redis Pub/Sub или Kafka для синхронизации между инстансами.

---

## Q18. Как сделать file upload?

```python
@app.post("/upload")
async def upload(file: UploadFile = File(...)):
    content = await file.read()
    path = Path("uploads") / file.filename
    path.write_bytes(content)
    return {"filename": file.filename, "size": len(content)}
```

---

## Q19. Как работает uvicorn workers vs gunicorn?

Uvicorn с `--workers N` запускает N процессов (каждый со своим event loop). Gunicorn с uvicorn workers — то же самое, но Gunicorn управляет процессами (graceful restart, сигналы). На проде: `gunicorn -k uvicorn.workers.UvicornWorker -w 4`.

---

## Q20. Как сделать healthcheck?

```python
@app.get("/health")
async def health(request: Request):
    # ping БД:
    engine = request.app.state.engine
    async with engine.connect() as conn:
        await conn.execute(text("SELECT 1"))
    # ping Redis:
    await request.app.state.redis.ping()
    return {"status": "ok"}
```