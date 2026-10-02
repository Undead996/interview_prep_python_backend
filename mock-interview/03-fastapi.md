# Мок-интервью: FastAPI (20 вопросов)

---

## Q1. FastAPI vs Flask?

```
✅ Ответ: «FastAPI — ASGI, async, автодокументация, Pydantic-валидация.
Flask — WSGI, sync, ручная документация. FastAPI быстрее. Для нового
проекта — FastAPI. Для legacy — Flask.»
```

## Q2. Что такое Dependency Injection?

```
✅ Ответ: «Механизм FastAPI: Depends() автоматически создаёт и передаёт
зависимости в эндпоинт. Применение: БД-сессия, auth, конфиг.
В тестах — dependency_overrides для мока. Эндпоинт не создаёт свои
зависимости — он их получает (DI).»
```

## Q3. Как обрабатывать ошибки?

```
✅ Ответ: «HTTPException — для стандартных HTTP-ошибок.
app.exception_handler(Exception) — для кастомных.
ValidationError (Pydantic) — FastAPI сам возвращает 422.
Для бизнес-ошибок — AppError(code, message, status).»
```

## Q4. Как работает Lifespan?

```
✅ Ответ: «@asynccontextmanager lifespan(event) — код до yield (startup),
код после yield (shutdown). Старый способ: @app.on_event("startup").
Lifespan — современный. Используется для engine, Redis, Kafka.»
```

## Q5. Как протестировать FastAPI?

```
✅ Ответ: «TestClient + session-scoped фикстура. Dependency overrides
для изоляции БД и auth. Контрактные тесты — jsonschema против /openapi.json.
BackgroundTasks — мок через unittest.mock.patch.»
```

## Q6. N+1 в SQLAlchemy — решение?

```
✅ Ответ: «joinedload для many-to-one, selectinload для many-to-many.
В SQLAlchemy 2.0 — .options(joinedload(Model.relation)). До этого —
не lazy=True, а selectin или joined.»
```

## Q7. Как устроена аутентификация в FastAPI?

```
✅ Ответ: «OAuth2PasswordBearer — получение Bearer-токена.
Зависимость get_current_user — decode JWT (jose.jwt.decode).
Проверка exp, sub, role. При ошибке — 401. Для RBAC — Depends
с проверкой роли.»
```

## Q8. Rate limiting — как сделать?

```
✅ Ответ: «In-memory счётчик в Redis (INCR + EXPIRE) или middleware
с dict[IP → list[timestamps]]. По key = {user_id|IP}:{minute},
incr, если > N — 429. Retry-After заголовок.»
```

## Q9. Что такое middleware в FastAPI?

```
✅ Ответ: «Функция, которая выполняется до и после каждого запроса.
@app.middleware("http") request → call_next → response.
Использую для: CORS, TrustedHosts, Process-Time, rate limiting,
логирование запросов.»
```

## Q10. Что делает `background_tasks.add_task`?

```
✅ Ответ: «Добавляет фоновую задачу, которая выполнится после отправки
ответа. Не async — выполняется синхронно. Для долгих задач — Celery/
RabbitMQ. BackgroundTasks — для быстрых: логирование, отправка email.»
```