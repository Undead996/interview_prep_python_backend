# Разбор вакансии Python Backend Developer — глубоко

---

## Типичная вакансия — что мы видим

```
Компания N ищет backend-разработчика (Python).
Стек: Python 3.12+, FastAPI, SQLAlchemy 2.0, PostgreSQL,
RabbitMQ / Kafka, Redis.
Плюсом: async/await, Docker, Alembic.

Ожидания: Middle / Senior.
```

Каждая строчка такой вакансии — это не просто список технологий. За ней стоит **набор компетенций**, который работодатель хочет проверить. Ниже — расшифровка того, **что именно** проверяют под каждым пунктом, и **какую глубину** ожидают увидеть.

---

## Детальная расшифровка требований

### 1. Python 3.12+

**Что написано:** знаешь Python 3, работаешь с новыми версиями.

**Что реально проверяют:**
- Типы данных и изменяемость: immutable vs mutable, что можно ключом dict, что нельзя
- ООП: наследование, MRO (C3 linearization), кооперативное наследование через `super()`
- Декораторы: зачем нужны, как писать с аргументами и без, `@functools.wraps`
- Контекстные менеджеры: `__enter__`/`__exit__`, `@contextmanager`, когда применять
- Data classes: `@dataclass`, `field()`, `__post_init__`, `frozen=True`
- Аннотации типов: `TypeAlias`, `Protocol`, `Literal`, `Generic`
- Подводные камни: mutable defaults, late binding, `is` vs `==`

**Как глубоко нужно знать:**
> Middle: уверенно пользуется перечисленным, понимает подвохи.
> Senior: может объяснить, **почему** Python так устроен (например, почему GIL существует, почему MRO именно C3 linearization), и какие trade-off у каждого выбора.

---

### 2. FastAPI

**Что написано:** умеешь писать API на FastAPI.

**Что реально проверяют:**
- **Dependency Injection**: не просто `Depends()`, а понимание, почему это DI, как переопределять в тестах, как кэшировать (`@lru_cache` на зависимость)
- **Pydantic модели**: валидация, `Field()`, `model_validator`, `field_serializer`
- **Middleware**: как написать свой, порядок выполнения, CORS
- **Lifespan**: правильный startup/shutdown для engine, Redis, Kafka
- **Аутентификация**: JWT (не просто копировать код, а понимать, как работает HMAC-SHA256, зачем `exp`, `sub`, `iat`), OAuth2 flow
- **SQLAlchemy 2.0 async**: async sessions, N+1, транзакции, connection pool

**Частый вопрос:** "Чем FastAPI отличается от Flask?"
> FastAPI — ASGI (async), автодокументация (OpenAPI), Pydantic-валидация. Flask — WSGI (sync), ручная документация. Для нового проекта — FastAPI.

---

### 3. SQLAlchemy 2.0

**Что написано:** знаком с ORM.

**Что реально проверяют:**
- **Async sessions** с `async_sessionmaker` — написать правильно, с lifecycle
- **N+1 problem**: объяснить, почему возникает, как лечить (`joinedload`, `selectinload`)
- **Транзакции**: `db.begin()`, `commit()`, `rollback()`, `savepoint`, вложенные транзакции
- **Connection pool**: зачем, как настроить (`pool_size`, `max_overflow`, `pool_pre_ping`)
- **Alembic**: автогенерация миграций, `upgrade`/`downgrade`, как избежать конфликтов

---

### 4. PostgreSQL / MySQL

**Что написано:** работал с SQL-базами.

**Что реально проверяют:**
- **MVCC**: как работает в PG (xmin/xmax, dead tuples, VACUUM), как в MySQL (undo log)
- **Индексы**: B-tree, GIN, BRIN, partial index, covering index — когда что выбирать
- **EXPLAIN ANALYZE**: читать план запроса, понимать cost/rows/Execution Time
- **JOIN vs EXISTS vs IN**: когда что быстрее
- **Оконные функции**: `ROW_NUMBER()`, `SUM() OVER`, `LAG()` — написать на доске
- **CTE vs подзапросы**: читаемость, производительность, рекурсивные CTE

**Ключевой вопрос:** "Чем PostgreSQL отличается от MySQL?"
> Это вопрос из разряда "покажи, что понимаешь разницу на уровне архитектуры, а не синтаксиса". Сравнивают MVCC, JSON, DDL-транзакции, строгость GROUP BY.

---

### 5. RabbitMQ

**Что написано:** работал с брокерами.

**Что реально проверяют:**
- **AMQP-модель**: Producer → Exchange → Queue → Consumer. Типы exchange.
- **Гарантии доставки**: publisher confirms, consumer ack, DLX, outbox pattern
- **Ретраи**: как организовать (DLX + retry queue)
- **Мониторинг**: `rabbitmqctl list_queues`, как увидеть размер очереди

---

### 6. Apache Kafka

**Что написано:** знаком с Kafka.

**Что реально проверяют:**
- **Архитектура**: Topic → Partition → Consumer Group → Offset
- **Гарантии**: at-most-once, at-least-once, exactly-once — как настроить
- **Rebalancing**: что происходит, как минимизировать
- **Offset management**: auto vs manual commit

**Ключевой вопрос:** "Kafka vs RabbitMQ — когда что?"
> Ответ должен показать, что ты понимаешь разницу между хранением (Kafka хранит, RabbitMQ удаляет), моделью потребления (Kafka — smart consumer, RabbitMQ — smart broker), use case (event sourcing vs task queue).

---

### 7. Redis

**Что написано:** работал с Redis.

**Что реально проверяют:**
- **Структуры данных**: String, List, Set, Hash, Sorted Set, Stream — когда что
- **Кэш-стратегии**: Cache Aside, Write Through, Write Behind, TTL, инвалидация
- **Persistence**: RDB vs AOF, trade-off по скорости/надёжности
- **Sentinel vs Cluster**: HA vs шардирование

---

### 8. Docker (бонус)

**Что написано:** "плюсом: Docker".

**Что реально проверяют:**
- `docker-compose.yml` для тестового окружения (app + db + redis)
- Multi-stage build для уменьшения образа
- Понимание разницы между Docker-образом и контейнером

---

## Типичные этапы собеса — что происходит на каждом

```
HR-скрининг
  ↓
Online-задача (SQL + Python)
  ↓
Техническое интервью (Python, БД, очереди)
  ↓
System design (проектирование сервиса)
  ↓
Оффер
```

### 1. HR-скрининг (15–30 мин)
- Рассказ о себе (1–2 мин, хронология)
- Почему ушёл с прошлого места (не "плохой начальник", а "хочу расти")
- Ожидания по зарплате (назвать диапазон ±20%)
- **Любимый баг** — подготовь 1 историю по STAR

### 2. Online-задача (1–2 часа)
- SQL: JOIN, GROUP BY, оконная функция
- Python: функция с обработкой ошибок, async, класс с методами
- **Что проверяют:** не "напиши идеально", а "покажи ход мыслей, обработай граничные случаи"

### 3. Техническое интервью (1–1.5 часа)
- Python: GIL, async/await, MRO, декораторы, mutable defaults
- FastAPI: DI, middleware, lifespan, SQLAlchemy N+1
- БД: MVCC, индексы, EXPLAIN, PostgreSQL vs MySQL
- Очереди: RabbitMQ vs Kafka, гарантии доставки

### 4. System design (45–60 мин)
- "Как спроектировать сервис нотификаций?"
- "Как сделать онлайн-табло заказов?"
- **Что проверяют:** не конкретные технологии, а умение комбинировать: FastAPI → БД → очередь → кэш

---

## Три любимых вопроса (prepare for these)

### Вопрос 1. PostgreSQL vs MySQL
```
"У нас MySQL, переезжаем на PostgreSQL. Что изменится?"
```
**Что хотят услышать:**
1. MVCC: в PG — xmin/xmax + VACUUM, в MySQL — undo log (InnoDB)
2. DDL: в PG — в транзакции (можно ROLLBACK ALTER TABLE), в MySQL — неявный COMMIT
3. JSON: в PG — JSONB (бинарный, с GIN-индексами), в MySQL — TEXT
4. GROUP BY: в PG — строгий (все неагрегированные поля в GROUP BY), в MySQL — можно не все
5. FULL OUTER JOIN: в PG есть, в MySQL нет (до 8.0)

### Вопрос 2. RabbitMQ vs Kafka
```
"Когда что выбирать?"
```
**Коротко:** RabbitMQ — задачи, RPC, уведомления (1 раз доставить). Kafka — стримы, event sourcing, аудит (много раз прочитать).

### Вопрос 3. Напиши SQL
```
"Напиши SQL-запрос с тремя JOIN и оконной функцией"
```
Готовься написать `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)` с JOIN трёх таблиц. См. `03-postgresql-mysql/03-sql-tasks-with-solutions.md`.