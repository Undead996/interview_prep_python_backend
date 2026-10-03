# Мок-интервью: FastAPI (20 вопросов) — расширенные ответы

---

## Q1. FastAPI vs Flask?

```
✅ Ответ:
FastAPI — ASGI (async), автодокументация (OpenAPI/Swagger), Pydantic-валидация.
Flask — WSGI (sync), ручная документация.

FastAPI быстрее (uvicorn) и современнее. Для нового проекта — FastAPI.
Flask — для legacy или очень простых проектов.

Дополнительно: FastAPI поддерживает lifespan, dependency injection,
background tasks, WebSocket.
```

---

## Q2. Что такое Dependency Injection?

```
✅ Ответ:
DI — механизм, при котором эндпоинт получает зависимости извне, а не создаёт
их внутри. В FastAPI — Depends().

Преимущества:
- Тестируемость (dependency_overrides для мока)
- Переиспользование
- Изоляция логики

Пример: @app.get("/users") db: AsyncSession = Depends(get_db)
```

---

## Q3. Как обрабатывать ошибки?

```
✅ Ответ:
1. HTTPException — для стандартных HTTP-ошибок (raise HTTPException(status_code=404))
2. @app.exception_handler(MyError) — для кастомных
3. Pydantic ValidationError → FastAPI сам возвращает 422
4. Для бизнес-ошибок — AppError(code, message, status)
```

---

## Q4. Как работает Lifespan?

```
✅ Ответ:
@asynccontextmanager lifespan(app): код до yield — startup (engine, Redis, Kafka),
код после yield — shutdown (dispose, close, stop).

Современный способ (вместо @app.on_event("startup")/"shutdown").
```

---

## Q5. Как протестировать FastAPI?

```
✅ Ответ:
1. TestClient(app) — для интеграционных тестов
2. Dependency overrides — для изоляции БД и auth
3. In-memory SQLite — для тестовой БД
4. Contract tests — jsonschema против /openapi.json
```

---

## Q6. N+1 в SQLAlchemy — решение?

```
✅ Ответ:
joinedload для many-to-one (LEFT JOIN),
selectinload для many-to-many (второй запрос с WHERE IN).
st = select(User).options(joinedload(User.category))
```

---

## Q7. Как устроена аутентификация в FastAPI?

```
✅ Ответ:
OAuth2PasswordBearer — получение Bearer-токена.
Depends get_current_user — decode JWT (jose.jwt.decode).
Проверка exp, sub, role. При ошибке — 401.
Для RBAC — Depends с проверкой роли.
```

---

## Q8. Rate limiting — как сделать?

```
✅ Ответ:
In-memory счётчик (dict[IP → list[timestamps]]) или Redis INCR + EXPIRE.
Key = {user_id|IP}:{minute}, incr, если > N — 429.
Retry-After заголовок.
```

---

## Q9. Что такое middleware?

```
✅ Ответ:
Функция, которая выполняется до и после каждого запроса.
@app.middleware("http") request → call_next → response.
Используется для: CORS, TrustedHost, rate limiting, логирование.
```

---

## Q10. Что делает background_tasks.add_task?

```
✅ Ответ:
Добавляет фоновую задачу, которая выполнится после отправки ответа.
Не async — синхронная. Для быстрых задач (логирование, email).
Для долгих — Celery/RabbitMQ.
```