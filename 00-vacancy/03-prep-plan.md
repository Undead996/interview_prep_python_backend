# План подготовки — детально

Этот файл — твой маршрут. Не "что почитать", а **какие именно навыки прокачать за каждую тренировку**.

---

## 14 дней (полный план)

### День 1. Разбор вакансии и самооценка (1 ч)

**Цель:** понять, что спрашивают на собесе, и честно оценить свой уровень.

```
Что сделать:
1. Прочитать `00-vacancy/01-vacancy-breakdown.md` — расшифровка требований.
2. Заполнить `00-vacancy/02-self-assessment.md` — проставить оценки 1–5.
3. Выписать 3 темы с минимальными оценками — это твои «слабые места».
```

**Результат:** знаешь, на чём фокусироваться. Слабые места — в приоритет.

---

### День 2–3. Python core + async (4 ч)

**Цель:** уверенно отвечать на вопросы про GIL, async/await, декораторы, MRO.

#### Сессия 1 (2 ч) — язык
```
1. `01-language-deep.md` — прочитать разделы:
   - Типы и изменяемость (immutable vs mutable)
   - ООП и MRO (C3 linearization, super())
   - Декораторы (с аргументами, как класс)
   - Контекстные менеджеры (класс vs @contextmanager)
   - Data classes (frozen, field, __post_init__)
   - Аннотации типов (Protocol, TypeAlias, Literal)

2. `03-standard-library-deep.md` — прочитать:
   - collections (Counter, defaultdict, deque, ChainMap)
   - itertools (product, cycle, groupby, batched, chain)
   - functools (lru_cache, partial, singledispatch)
   - pathlib (Path, glob, read_text/write_text)

3. `04-pitfalls-and-tasks.md` — прочитать:
   - mutable defaults
   - late binding
   - is vs ==
   - задачи 1–4 (TTL cache, rate limiter, async bulk insert)
```

**Как проверять:** устно ответить на вопросы из `mock-interview/02-python-deep.md` (вопросы 1–10).

#### Сессия 2 (2 ч) — async
```
1. `02-async-and-concurrency.md` — прочитать разделы:
   - GIL (почему, как обойти)
   - asyncio vs threading vs multiprocessing (таблица)
   - Event loop (как работает, типы, run_in_executor)
   - Корутины, Tasks, Futures (жизненный цикл)
   - gather / TaskGroup / wait / as_completed
   - Async queues (producer-consumer)
   - Примитивы (Lock, Semaphore, Event, Condition)
   - Timeout и cancellation (wait_for, timeout, cancel)
   - Async-итераторы и генераторы
```

**Как проверять:** написать код producer-consumer с `asyncio.Queue` и `Semaphore`.

---

### День 4–5. FastAPI + SQLAlchemy (4 ч)

**Цель:** написать CRUD с async SQLAlchemy 2.0, DI, JWT, тестами.

#### Сессия 1 (2 ч) — FastAPI core
```
1. `02-fastapi/01-routing-and-dependency-injection.md`:
   - Path/Query параметры
   - Depends() — функция, класс, генератор
   - Middleware (как написать, CORS, TrustedHost)
   - Exception handlers (кастомные ошибки)
   - BackgroundTasks
   - Lifespan

2. `02-fastapi/03-auth-and-security.md`:
   - JWT encode/decode (exp, sub)
   - OAuth2PasswordBearer + OAuth2PasswordRequestForm
   - bcrypt hashing (passlib)
   - RBAC (require_role)
   - Rate limiting (in-memory + Redis)
```

#### Сессия 2 (2 ч) — SQLAlchemy + тесты
```
1. `02-fastapi/02-sqlalchemy-and-database.md`:
   - Async engine + async_sessionmaker
   - Repository pattern
   - N+1 (joinedload vs selectinload)
   - Транзакции (begin/commit/savepoint)
   - Connection pool
   - Alembic (init, autogenerate, upgrade/downgrade)

2. `02-fastapi/04-testing-and-deployment.md`:
   - TestClient with lifespan
   - Dependency overrides (in-memory SQLite)
   - Dockerfile (multi-stage)
   - CI/CD (GitHub Actions)
```

**Как проверять:** решить `tasks/02-fastapi-tasks.md` (3 задачи: CRUD, JWT, контрактный тест).

---

### День 6–7. PostgreSQL + MySQL (4 ч)

**Цель:** читать EXPLAIN ANALYZE, объяснить MVCC, написать оконные функции.

```
1. `03-postgresql-mysql/01-postgresql-deep.md`:
   - MVCC (xmin/xmax, dead tuples, VACUUM vs VACUUM FULL)
   - Индексы: B-tree / GIN / BRIN / partial / covering
   - EXPLAIN ANALYZE (Seq Scan vs Index Scan vs Index Only Scan)
   - Транзакции и уровни изоляции
   - Партиционирование
   - CTE и оконные функции
   - pg_stat_* — диагностика

2. `03-postgresql-mysql/02-mysql-deep-and-diff.md`:
   - Сравнительная таблица PG vs MySQL
   - Когда мигрируют с MySQL на PG
   - Различия в синтаксисе и транзакциях
   - Движки MySQL

3. `03-postgresql-mysql/03-sql-tasks-with-solutions.md`:
   - Решить все 8 задач
```

**Как проверять:** устно ответить на вопросы из `mock-interview/04-postgresql-mysql.md`.

---

### День 8–9. RabbitMQ (3 ч)

**Цель:** объяснить AMQP-модель, DLX, outbox pattern, написать consumer с retry.

```
1. `04-rabbitmq/01-amqp-concepts.md`:
   - AMQP-модель (Producer → Exchange → Queue → Consumer)
   - Типы exchange (direct, fanout, topic, headers)
   - Свойства очередей (durable, auto_delete, exclusive)
   - DLX (когда срабатывает, как настроить)
   - VHosts и Channels

2. `04-rabbitmq/02-reliability-and-patterns.md`:
   - Publisher confirms
   - Consumer ack/nack/reject
   - Outbox pattern (почему нужен, как работает)
   - TTL
   - Quorum queues

3. `04-rabbitmq/03-python-integration-tasks.md`:
   - aio-pika (producer/consumer)
   - FastAPI + RabbitMQ (lifespan)
   - Задачи 1–2 (retry + DLX, outbox)
```

**Как проверять:** решить `tasks/03-queue-tasks.md` (задача 1 — outbox).

---

### День 10–11. Apache Kafka (3 ч)

**Цель:** объяснить partition, offset, rebalancing, гарантии доставки.

```
1. `05-kafka/01-kafka-concepts.md`:
   - Topic → Partition → Consumer Group → Offset
   - Producer (key-hash, round-robin)
   - Kafka vs RabbitMQ (таблица)
   - Когда выбирать что

2. `05-kafka/02-reliability-and-consumption.md`:
   - Гарантии: at-most-once, at-least-once, exactly-once
   - Offset management (auto vs manual commit)
   - Rebalancing (что происходит, как уменьшить)
   - Idempotent producer

3. `05-kafka/03-python-integration-tasks.md`:
   - aiokafka (producer/consumer)
   - confluent-kafka
   - FastAPI + Kafka
   - Задачи 1–2
```

**Как проверять:** устно ответить на вопросы из `mock-interview/05-rabbitmq-kafka.md`.

---

### День 12. Redis (2 ч)

**Цель:** написать rate limiter, distributed lock, cache aside.

```
1. `06-redis/01-data-structures-and-caching.md`:
   - Strings, Lists, Sets, Hashes, Sorted Sets, Streams
   - Стратегии кэширования (Cache Aside, Write Through, Write Behind)

2. `06-redis/02-persistence-and-advanced.md`:
   - RDB vs AOF (trade-off)
   - Sentinel vs Cluster
   - Rate limiting (sliding window)
   - Distributed lock (setnx + Lua)
   - Pub/Sub vs Streams

3. `06-redis/03-python-integration-tasks.md`:
   - redis-py vs redis.asyncio
   - FastAPI + Redis
   - Задачи 1–2
```

**Как проверять:** написать rate limiter middleware с Redis.

---

### День 13. Мок-интервью вслух (3 ч)

**Цель:** проговорить ответы, чтобы на собесе не "зажевывать".

```
1. `mock-interview/01-hr-screening.md` — рассказ о себе (1–2 мин)
2. `mock-interview/02-python-deep.md` — 30 вопросов вслух
3. `mock-interview/03-fastapi.md` — 10 вопросов вслух
4. `mock-interview/04-postgresql-mysql.md` — 6 вопросов вслух
5. `mock-interview/05-rabbitmq-kafka.md` — 6 вопросов вслух
6. `mock-interview/06-redis.md` — 6 вопросов вслух
7. `mock-interview/07-live-coding.md` — решить 3 задачи на бумаге/доске
8. `mock-interview/08-behavioral-star.md` — рассказать 3 STAR-истории
```

**Важно:** не читай с экрана. Закрой файл и расскажи своими словами. Если плаваешь — перечитай материал.

---

### День 14. Cheatsheets + STAR (2 ч)

**Цель:** освежить всё перед собеседованием.

```
1. `cheatsheets/*` — пролистать все 6 файлов (5 мин каждый)
2. `mock-interview/07-live-coding.md` — ещё раз решить задачи (30 мин)
3. `mock-interview/08-behavioral-star.md` — рассказать истории (30 мин)
```

**Не учи новое!** Только повторение.

---

## 7 дней (интенсив)

Если времени мало — сжимаем до самого важного:

| День | Что делаем | Почему |
|------|-----------|--------|
| 1 | Python core + async | База, без неё никуда |
| 2 | FastAPI + SQLAlchemy | Основной инструмент |
| 3 | PostgreSQL + MySQL | SQL на собесе — 40% времени |
| 4 | RabbitMQ + Kafka базово | Очереди — 20% времени |
| 5 | Mock-interview: Python + FastAPI вслух | Проговорить ответы |
| 6 | Решить задачи: Python + FastAPI | Практика |
| 7 | Cheatsheets + Live coding + STAR | Финальный прогон |

---

## 3 дня (SOS)

Режим "пожар":

| День | Что успеваем | Формат |
|------|-------------|--------|
| 1 | Python core (типы, MRO, декораторы, async) + FastAPI core (DI, middleware, lifespan) | Читать + сразу отвечать вслух |
| 2 | PostgreSQL (MVCC, индексы, EXPLAIN) + RabbitMQ (AMQP, DLX, ack) | Только теория, без кода |
| 3 | Mock-interview: Python + FastAPI + БД вслух + cheatsheets | Проговорить, не учить новое |

---

## День перед собесом — чеклист

```
□ 08:00 — Пройти cheatsheets (30 мин)
□ 09:00 — Live-coding (2 задачи "на доске", 30 мин)
□ 10:00 — STAR-истории вслух (30 мин)
□ 20:00 — Спать
□ Батарейка ноутбука заряжена
□ Ссылка на созвон открыта
□ Документы (паспорт/СНИЛС) под рукой
□ Стакан воды на столе
```