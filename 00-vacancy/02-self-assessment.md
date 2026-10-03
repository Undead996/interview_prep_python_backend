# Self-assessment — самооценка с детальным разбором

Оцени себя 1–5 по каждому пункту. К каждому минусу — ссылка на материал.
Ниже — **критерии** для каждой оценки, чтобы ты понимал, что именно означает "3" или "5".

---

## Критерии оценки (единая шкала)

| Оценка | Что означает |
|--------|-------------|
| 1 | Не знаю / никогда не использовал |
| 2 | Знаю теорию, но не применял в работе |
| 3 | Применял, могу написать код, но плаваю в сложных вопросах |
| 4 | Уверенно применяю, могу объяснить trade-off |
| 5 | Могу учить других, знаю внутреннее устройство |

---

## Python

| Тема | Оценка (1–5) | Что именно нужно уметь для 4+ | Материал |
|------|-------------|------------------------------|----------|
| Типы, ООП, MRO | `__/5` | immutable vs mutable, `__new__` vs `__init__`, C3 linearization, `super()` в множественном наследовании, абстрактные классы, протоколы | `01-python-core/01-language-deep.md` |
| Async/await, asyncio | `__/5` | Event loop, корутина vs Task vs Future, `gather` vs `wait` vs `as_completed`, `TaskGroup`, timeout, cancellation, async-генераторы, `run_in_executor` | `01-python-core/02-async-and-concurrency.md` |
| Стандартная библиотека | `__/5` | `collections` (Counter, defaultdict, deque, ChainMap), `itertools` (product, cycle, groupby, batched, chain), `typing.Protocol`, `dataclasses`, `pathlib`, `functools` (lru_cache, partial, singledispatch) | `01-python-core/03-standard-library-deep.md` |
| Подводные камни | `__/5` | mutable defaults, late binding, `is` vs `==`, замыкания, цикл import, изменяемый ключ dict, кэширование маленьких int | `01-python-core/04-pitfalls-and-tasks.md` |

---

## FastAPI

| Тема | Оценка (1–5) | Что нужно для 4+ | Материал |
|------|-------------|-----------------|----------|
| Routing, Depends, middleware | `__/5` | Path/Query параметры, `Depends()` (функция/класс/генератор), кэширование DI, middleware как фабрика, exception handlers, BackgroundTasks, lifespan | `02-fastapi/01-routing-and-dependency-injection.md` |
| SQLAlchemy 2.0, Alembic | `__/5` | Async engine, async_sessionmaker, Repository pattern, N+1 (joinedload vs selectinload), транзакции (begin/commit/rollback/savepoint), connection pool, Alembic autogenerate | `02-fastapi/02-sqlalchemy-and-database.md` |
| JWT, OAuth2, hashing | `__/5` | JWT encode/decode, `exp`, `sub`, `iat`, bcrypt/argon2, OAuth2PasswordBearer, RBAC, CORS, rate limiting in Redis | `02-fastapi/03-auth-and-security.md` |
| TestClient, pytest, deploy | `__/5` | TestClient with lifespan, dependency_overrides, in-memory SQLite for tests, GitHub Actions, Docker multi-stage, uvicorn workers | `02-fastapi/04-testing-and-deployment.md` |

---

## Базы данных

| Тема | Оценка (1–5) | Что нужно для 4+ | Материал |
|------|-------------|-----------------|----------|
| PostgreSQL: MVCC, EXPLAIN, индексы | `__/5` | MVCC (xmin/xmax, dead tuples), VACUUM vs VACUUM FULL, B-tree/GIN/BRIN (когда что), partial/covering indexes, EXPLAIN (Seq Scan vs Index Scan vs Index Only Scan), CTE, оконные функции, партиционирование | `03-postgresql-mysql/01-postgresql-deep.md` |
| MySQL vs PostgreSQL | `__/5` | Различия в MVCC (xmin vs undo log), DDL-транзакции, JSON (JSONB vs TEXT), строгость GROUP BY, FULL OUTER JOIN, движки InnoDB vs MyISAM | `03-postgresql-mysql/02-mysql-deep-and-diff.md` |
| SQL-задачи | `__/5` | Вторая зарплата, пользователи без заказов, топ-3 за месяц, дубликаты, running total, оконные функции с JOIN | `03-postgresql-mysql/03-sql-tasks-with-solutions.md` |

---

## Очереди

| Тема | Оценка (1–5) | Что нужно для 4+ | Материал |
|------|-------------|-----------------|----------|
| RabbitMQ: AMQP, exchanges, DLX | `__/5` | AMQP-модель, типы exchange, durable/auto_delete, DLX (зачем, как настроить), publisher confirms, consumer ack/nack/reject, vhosts, channels | `04-rabbitmq/*` |
| Kafka: topics, partitions, consumer groups | `__/5` | Topic → Partition → Consumer Group → Offset, key-hash, rebalancing, at-least-once/exactly-once, idempotent producer, offset management | `05-kafka/*` |
| RabbitMQ vs Kafka — trade-off | `__/5` | Когда выбирать каждую, модели (smart broker vs dumb broker), RPC vs event sourcing | `05-kafka/01-kafka-concepts.md` |

---

## Redis

| Тема | Оценка (1–5) | Что нужно для 4+ | Материал |
|------|-------------|-----------------|----------|
| Структуры данных | `__/5` | Strings (set/get/incr), Lists (lpush/rpop), Sets (sadd/sinter), Hashes (hset/hgetall), Sorted Sets (zadd/zrange), Streams (xadd/xgroup) — когда что выбирать | `06-redis/01-data-structures-and-caching.md` |
| Persistence, Cluster, Sentinel | `__/5` | RDB vs AOF (trade-off), Sentinel vs Cluster, Lua-скриптинг | `06-redis/02-persistence-and-advanced.md` |
| Rate limiting, locks, pub/sub | `__/5` | Sliding window rate limiter, distributed lock (setnx + Lua), Pub/Sub (не для надёжной доставки) vs Streams | `06-redis/03-python-integration-tasks.md` |

---

## Как заполнять

1. По каждому пункту поставь оценку.
2. Где ≤ 3 — открой материал и прочитай/напиши код.
3. Через неделю перепроверь: можешь ли объяснить **вслух** без заглядывания?

> **Совет:** не ставь себе 5, если не можешь объяснить это джуниору. Если плаваешь в ответе — это 3 или ниже.