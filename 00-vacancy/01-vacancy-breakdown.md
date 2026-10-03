# Разбор вакансии Python Backend Developer — глубоко

> **Цель:** понять, что **на самом деле** проверяют за каждой строчкой вакансии, и какую глубину ответа ожидает интервьюер.

---

## Типичная вакансия — расшифровка

```
Компания N ищет backend-разработчика (Python).
Стек: Python 3.12+, FastAPI, SQLAlchemy 2.0, PostgreSQL,
RabbitMQ / Kafka, Redis, pytest.
Плюсом: Docker, Alembic, asyncio deep knowledge.

Ожидания: Middle / Senior.
```

Каждая строчка — это не просто технология. Это **набор компетенций + ожидаемая глубина**. Интервьюер не спрашивает «использовали ли вы FastAPI» — он проверяет, понимаете ли вы, **почему** он работает именно так.

---

## 1. Python 3.12+ — что реально проверяют

### Не «знаю синтаксис», а «понимаю семантику»

**Уровень Middle (достаточно для работы):**
- Типы и изменяемость: immutable vs mutable, что можно ключом dict
- ООП: наследование, MRO (C3 linearization), `super()` в множественном наследовании
- Декораторы: с аргументами и без, `@functools.wraps` (почему обязателен)
- Контекстные менеджеры: класс vs `@contextmanager`, когда какой
- Data classes: `@dataclass`, `field()`, `__post_init__`, `frozen=True`
- Аннотации типов: `TypeAlias`, `Protocol`, `Literal`, `Generic`

**Уровень Senior (что выводит на оффер):**
- Может объяснить, **почему** Python устроен так: GIL как trade-off, C3 linearization как решение diamond problem, почему `__new__` отделён от `__init__`
- Понимает модель памяти: ссылки, id(), intern-строки, кэширование малых int
- Знает, как работают генераторы и корутины на уровне байткода (yield как точка сохранения стека)
- Может написать свой декоратор с сохранением сигнатуры (ParamSpec)

### Типичные проверочные вопросы

| Вопрос | Что проверяют | Ожидаемая глубина |
|--------|--------------|-------------------|
| «Почему list нельзя ключом dict?» | hashable vs mutable | __hash__, id-неизменяемость |
| «Как работает MRO?» | C3 linearization | merge-алгоритм, diamond problem |
| «Зачем @wraps?» | Метапрограммирование | __name__, __doc__, __wrapped__ |
| «Чем `__new__` отличается от `__init__`?» | Жизненный цикл объекта | Singleton, immutable types |
| «Почему GIL существует?» | CPython internals | trade-off: однопоточная скорость vs многопоточность |

---

## 2. FastAPI — что реально проверяют

### Не «написал CRUD», а «понимаю архитектуру»

**Обязательный минимум:**
- **Dependency Injection**: Depends() как функция / класс / генератор, кэширование, переопределение в тестах
- **Pydantic-валидация**: `Field()`, `model_validator`, `field_serializer`, nested models
- **Middleware**: порядок выполнения, CORS, TrustedHost, кастомный logging
- **Lifespan**: startup/shutdown для БД, Redis, Kafka — правильный cleanup
- **Аутентификация**: JWT (структура токена, проверка подписи, exp/sub/iat), OAuth2 flow
- **SQLAlchemy 2.0 async**: async sessions, N+1, транзакции, connection pool

**Senior-уровень:**
- Понимает ASGI-спецификацию: scope, receive, send
- Знает, как Uvicorn запускает приложение (uvloop + httptools)
- Может написать кастомный ASGI-middleware
- Понимает разницу между `BackgroundTasks` и `asyncio.create_task`

### Типичные проверочные вопросы

| Вопрос | Что проверяют | Ожидаемая глубина |
|--------|--------------|-------------------|
| «FastAPI vs Flask?» | ASGI vs WSGI | event loop, конкурентность, производительность |
| «Как переопределить Depends в тестах?» | DI-контейнер | `dependency_overrides` |
| «Как работает lifespan?» | Управление ресурсами | startup/shutdown, `@asynccontextmanager` |
| «Как предотвратить N+1?» | SQLAlchemy ORM | `joinedload` vs `selectinload` |

---

## 3. SQLAlchemy 2.0 + Alembic — что реально проверяют

**Обязательный минимум:**
- **Async sessions**: `create_async_engine`, `async_sessionmaker`, правильный lifecycle
- **N+1 problem**: объяснить причину, показать 3 способа решения
- **Транзакции**: `begin()`, `commit()`, `rollback()`, `savepoint`, вложенные транзакции
- **Connection pool**: `pool_size`, `max_overflow`, `pool_pre_ping`, почему это важно
- **Alembic**: `init`, `revision --autogenerate`, `upgrade`/`downgrade`, разрешение конфликтов

**Senior-уровень:**
- Понимает разницу между Core и ORM в SQLAlchemy 2.0
- Знает, когда использовать raw SQL вместо ORM
- Может объяснить, как `expire_on_commit` влияет на производительность
- Понимает, как работает `QueuePool` vs `AsyncAdaptedQueuePool`

---

## 4. PostgreSQL / MySQL — что реально проверяют

### Не «знаю SQL», а «понимаю, как БД работает под капотом»

**Обязательный минимум:**
- **MVCC**: xmin/xmax (PG) vs undo log (MySQL), зачем VACUUM
- **Индексы**: B-tree / GIN / BRIN / partial / covering — когда какой
- **EXPLAIN ANALYZE**: читать план запроса, понимать cost/rows/actual time
- **JOIN vs EXISTS vs IN**: когда что быстрее, план запроса
- **Оконные функции**: `ROW_NUMBER()`, `SUM() OVER`, `LAG()` — написать на доске
- **PostgreSQL vs MySQL**: 5+ архитектурных различий

**Senior-уровень:**
- Уровни изоляции и как они реализованы (snapshot isolation в PG)
- Блокировки: явные (FOR UPDATE) и неявные (MVCC)
- VACUUM FULL vs pg_repack
- Партиционирование: range / list / hash + как это влияет на планы запросов

### Ключевой вопрос: «PostgreSQL vs MySQL — что изменится при миграции?»

Это **самый частый вопрос** на middle/senior. Ожидается развёрнутый ответ:
1. **MVCC**: PG — xmin/xmax + VACUUM (может отставать), MySQL — undo log (прозрачно, но растёт)
2. **DDL**: PG — в транзакции (можно `ROLLBACK ALTER TABLE`), MySQL — неявный `COMMIT`
3. **JSON**: PG — `JSONB` (бинарный, GIN-индексы), MySQL — `JSON` (текстовый, без индексов)
4. **GROUP BY**: PG — строгий (все неагрегированные поля должны быть в GROUP BY), MySQL — нестрогий
5. **FULL OUTER JOIN**: PG — есть, MySQL — нет (до 8.0)
6. **Оконные функции**: PG — полная поддержка, MySQL — с 8.0, но хуже производительность

---

## 5. RabbitMQ — что реально проверяют

**Обязательный минимум:**
- **AMQP-модель**: Producer → Exchange → (binding) → Queue → Consumer
- **Типы exchange**: direct, fanout, topic, headers — когда каждый
- **Гарантии доставки**: publisher confirms, consumer ack, persistent messages
- **DLX**: что это, когда срабатывает, как настроить retry через DLX
- **Outbox pattern**: проблема, решение, реализация в коде

---

## 6. Apache Kafka — что реально проверяют

**Обязательный минимум:**
- **Архитектура**: Topic → Partition → Consumer Group → Offset
- **Producer**: key-hash vs round-robin, idempotent producer
- **Гарантии**: at-least-once, exactly-once, как настроить
- **Rebalancing**: что происходит, как минимизировать
- **Kafka vs RabbitMQ**: когда что выбирать (таблица сравнения)

---

## 7. Redis — что реально проверяют

**Обязательный минимум:**
- **Структуры данных**: String, List, Set, Hash, Sorted Set, Stream — когда каждую
- **Кэш-стратегии**: Cache Aside, Write Through, Write Behind, TTL, инвалидация
- **Persistence**: RDB vs AOF — trade-off по скорости и надёжности
- **Rate limiting**: sliding window, token bucket (реализация в коде)
- **Distributed lock**: SETNX + Lua для атомарного освобождения

---

## 8. pytest — что реально проверяют

**Обязательный минимум:**
- **Fixtures**: scope (function/class/module/session), yield, parametrize
- **Async tests**: `@pytest.mark.asyncio`, `pytest-asyncio`
- **Mocking**: `unittest.mock`, `AsyncMock`, `patch` — когда уместно, когда нет
- **TestClient**: `with TestClient(app)`, dependency overrides, in-memory SQLite
- **Coverage**: `pytest --cov`, что должно быть покрыто

**Senior-уровень:**
- Factory fixtures (factory_boy / свои)
- Contract tests (jsonschema vs OpenAPI)
- Property-based testing (Hypothesis)

---

## 9. Docker — что реально проверяют (бонус)

**Обязательный минимум:**
- `docker-compose.yml` для локального окружения (app + db + redis + rabbitmq)
- Multi-stage build для уменьшения образа
- Dockerfile best practices: `.dockerignore`, non-root user, healthcheck
- Понимание разницы между образом и контейнером

---

## Типичные этапы собеса — детально

```
HR-скрининг (15–30 мин)
  ↓
Техническое интервью: Python + БД + очереди (1–1.5 ч)
  ↓
Live-coding / Online-задача (1–2 ч)
  ↓
System design (45–60 мин)
  ↓
Финальное собеседование (team lead / CTO, 30–45 мин)
  ↓
Оффер
```

### Этап 1. HR-скрининг (15–30 мин)
- Рассказ о себе (1–2 мин, хронология, не читать CV)
- Почему уходите (не «плохой начальник», а «хочу роста / другой домен»)
- Ожидания по зарплате (диапазон ±20% или попросить назвать бюджет)
- **Любимый баг** — подготовьте 1 историю по STAR

### Этап 2. Техническое интервью (1–1.5 ч)
- **Python (30%):** GIL, async/await, MRO, декораторы, mutable defaults, контекстные менеджеры
- **FastAPI (20%):** DI, middleware, lifespan, SQLAlchemy async, N+1
- **БД (30%):** MVCC, индексы, EXPLAIN, PostgreSQL vs MySQL, уровни изоляции
- **Очереди + Redis (20%):** RabbitMQ vs Kafka, гарантии доставки, rate limiting, кэш-стратегии

### Этап 3. Live-coding (1–2 ч)
- **SQL-задача:** JOIN, GROUP BY, оконная функция — написать на общем экране
- **Python-задача:** функция с обработкой ошибок, async-генератор, декоратор
- **Что проверяют:** ход мыслей, обработка краевых случаев, naming, тестируемость

### Этап 4. System design (45–60 мин)
- «Спроектируйте сервис нотификаций»
- «Как сделать ленту новостей с сортировкой по времени?»
- **Что проверяют:** умение комбинировать технологии + trade-off + масштабирование

### Этап 5. Финальное (team lead / CTO, 30–45 мин)
- Культурное соответствие (culture fit)
- Как решаете конфликты в команде
- Как оцениваете задачи
- Куда хотите расти через 2–3 года

---

## Три любимых вопроса (подготовьте развёрнутый ответ)

### Вопрос 1. «PostgreSQL vs MySQL — что изменится при миграции?»

**Полный ответ:**
1. MVCC: в PG — xmin/xmax + VACUUM (нужно следить), в MySQL — undo log (прозрачно, но может расти)
2. DDL: в PG — в транзакции (ROLLBACK ALTER TABLE работает), в MySQL — неявный COMMIT после каждого DDL
3. JSON: в PG — JSONB (бинарный, с GIN-индексами), в MySQL — JSON (текстовый, без индексов по ключам)
4. GROUP BY: в PG — строгий (все неагрегированные поля в GROUP BY), в MySQL — можно не все
5. FULL OUTER JOIN: в PG есть, в MySQL нет (до 8.0; в 8.0+ через UNION LEFT + RIGHT)
6. Оконные функции: в PG полная поддержка с 9.x, в MySQL с 8.0 (хуже производительность)

### Вопрос 2. «RabbitMQ vs Kafka — когда что выбираете?»

**Коротко:**
- RabbitMQ — для задач (сообщение доставить 1 раз, RPC, сложная маршрутизация)
- Kafka — для потоков (сообщение прочитают много раз, event sourcing, аудит)

**Детально:**
| Характеристика | RabbitMQ | Kafka |
|---------------|---------|-------|
| Модель | Smart broker, dumb consumer | Dumb broker, smart consumer |
| Хранение | Удаляется после ack | Хранится N дней |
| Порядок | В одной очереди | В одной partition |
| Пропускная способность | ~50K msg/s | ~1M+ msg/s |
| Ретраи | Встроенные (DLX) | Ручные (retry topic) |
| RPC | ✅ (reply-to) | ❌ |
| Типичное применение | Задачи, уведомления, RPC | Event sourcing, стриминг, Big Data |

### Вопрос 3. «Напишите SQL-запрос с JOIN и оконной функцией»

Готовьтесь написать что-то вроде:
```sql
SELECT
    c.name AS category,
    p.name AS product,
    SUM(s.amount) AS total,
    ROW_NUMBER() OVER (PARTITION BY c.id ORDER BY SUM(s.amount) DESC) AS rank
FROM sales s
JOIN products p ON p.id = s.product_id
JOIN categories c ON c.id = p.category_id
WHERE s.sale_date >= '2024-01-01'
GROUP BY c.id, c.name, p.id, p.name
ORDER BY c.name, rank;
```

---

## Что отличает хорошего кандидата от среднего

| Средний кандидат | Сильный кандидат |
|-----------------|------------------|
| Перечисляет технологии | Объясняет, **почему** выбрал эту технологию |
| Говорит «использовал JWT» | Объясняет структуру токена, как проверяется подпись |
| Знает синтаксис SQL | Читает EXPLAIN ANALYZE, предлагает индексы |
| Говорит «писал тесты» | Объясняет стратегию тестирования, что мокает, а что нет |
| Знает, что RabbitMQ и Kafka — брокеры | Объясняет, когда что, с цифрами и trade-off |

---

> **На собесе:** вас спрашивают не «какие технологии знаешь». Вас спрашивают «какие проблемы ты решал этими технологиями и **почему** именно ими, а не другими». Каждый ответ должен содержать **причину выбора**.