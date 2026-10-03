# Self-assessment — самооценка с детальными критериями

> **Цель:** честно оценить свой уровень по каждому требованию вакансии и понять, куда направить усилия.

---

## Как пользоваться этим файлом

1. Поставь оценку 1–5 в каждой строке
2. Там где ≤ 3 — открой указанный материал и прочитай / напиши код
3. Через неделю перепроверь: можешь ли объяснить **вслух** без заглядывания?
4. В день перед собесом пройди ещё раз — оценки должны быть ≥ 4

---

## Единая шкала оценки

| Оценка | Что означает | Как проверить |
|--------|-------------|---------------|
| **1** | Не знаю / никогда не использовал | Даже термин незнаком |
| **2** | Знаю теорию поверхностно, не применял в работе | Могу дать определение, но не написать код |
| **3** | Применял, могу написать код, но плаваю в сложных вопросах | Пишу, но на «почему» отвечаю неуверенно |
| **4** | Уверенно применяю, могу объяснить trade-off и «почему» | Объясню джуниору без подготовки |
| **5** | Могу учить других, знаю внутреннее устройство, пишу статьи / доклады | Могу написать свою реализацию event loop |

> **Важно:** не ставь себе 5, если не можешь объяснить тему джуниору или написать работающий код без гугла. 4 — это уже отличный результат для собеса.

---

## Python — ядро

| # | Тема | Оценка (1–5) | Что нужно уметь для оценки 4+ | Материал |
|---|------|-------------|-------------------------------|----------|
| 1 | Типы и изменяемость | `__/5` | immutable vs mutable на уровне памяти, hashable-контракт, почему tuple с list внутри — не ключ | `01-python-core/01` |
| 2 | ООП: наследование, MRO | `__/5` | C3 linearization (алгоритм merge), `super()` в множественном наследовании, diamond problem, `__new__` vs `__init__` | `01-python-core/01` |
| 3 | Декораторы | `__/5` | Функция vs класс, с аргументами (3 уровня), `@functools.wraps`, ParamSpec, класс как декоратор с состоянием | `01-python-core/01` |
| 4 | Контекстные менеджеры | `__/5` | `__enter__`/`__exit__`, `@contextmanager`, обработка исключений в `__exit__`, вложенные | `01-python-core/01` |
| 5 | Data classes | `__/5` | `@dataclass`, `field()`, `__post_init__`, `frozen`, `slots` (3.10+), `kw_only` | `01-python-core/01` |
| 6 | Аннотации типов | `__/5` | `Protocol`, `TypeAlias`, `Literal`, `Generic`, `TypeVar`, `ParamSpec`, `TypedDict` | `01-python-core/01` |
| 7 | GIL | `__/5` | Почему существует, что блокирует, что нет, как обойти: asyncio / multiprocessing / C-extensions | `01-python-core/02` |
| 8 | Async/await | `__/5` | Event loop (epoll/kqueue/IOCP), Coroutine vs Task vs Future, `gather` / `TaskGroup` / `wait` / `as_completed` | `01-python-core/02` |
| 9 | Примитивы asyncio | `__/5` | `Semaphore`, `Lock`, `Event`, `Condition`, `Queue` — producer-consumer паттерн | `01-python-core/02` |
| 10 | threading vs multiprocessing | `__/5` | Когда что: таблица сравнения (CPU vs I/O, GIL), `ThreadPoolExecutor` vs `ProcessPoolExecutor` | `01-python-core/02` |
| 11 | collections | `__/5` | `defaultdict`, `Counter`, `deque`, `ChainMap` — какая проблема решается каждым | `01-python-core/03` |
| 12 | itertools | `__/5` | `product`, `cycle`, `groupby` (сортировка!), `chain`, `batched` (3.12+) | `01-python-core/03` |
| 13 | functools + pathlib | `__/5` | `lru_cache` (ограничения для async!), `partial`, `singledispatch`, `Path` vs `os.path` | `01-python-core/03` |
| 14 | Подводные камни | `__/5` | Mutable defaults, late binding, `is` vs `==`, замыкания, циклический import, intern-строки | `01-python-core/04` |

**Суммарная оценка Python: `___/70` → средняя: `__/5`**

---

## FastAPI + pytest

| # | Тема | Оценка (1–5) | Что нужно уметь для оценки 4+ | Материал |
|---|------|-------------|-------------------------------|----------|
| 1 | Routing, Query/Path/Body | `__/5` | Path-параметры, Query-валидация, Body-nested models, response_model | `02-fastapi/01` |
| 2 | Dependency Injection | `__/5` | `Depends()` — функция / класс / генератор, кэширование, `dependency_overrides` для тестов | `02-fastapi/01` |
| 3 | Middleware | `__/5` | `@app.middleware("http")`, порядок выполнения, CORS, TrustedHost, кастомный logging | `02-fastapi/01` |
| 4 | Exception handlers | `__/5` | `@app.exception_handler`, кастомные бизнес-ошибки, единый формат ответа | `02-fastapi/01` |
| 5 | Lifespan | `__/5` | `@asynccontextmanager`, startup/shutdown, engine/Redis/Kafka cleanup | `02-fastapi/01` |
| 6 | SQLAlchemy 2.0 async | `__/5` | `create_async_engine`, `async_sessionmaker`, `expire_on_commit=False`, pool_size/max_overflow | `02-fastapi/02` |
| 7 | N+1 problem | `__/5` | `joinedload` vs `selectinload` vs `subqueryload` vs `raiseload` — когда каждый | `02-fastapi/02` |
| 8 | Транзакции | `__/5` | `begin()`/`commit()`/`rollback()`, `savepoint`, вложенные транзакции, `SERIALIZABLE` | `02-fastapi/02` |
| 9 | Alembic | `__/5` | `init`, `revision --autogenerate`, `upgrade`/`downgrade`, разрешение конфликтов миграций | `02-fastapi/02` |
| 10 | JWT + OAuth2 | `__/5` | JWT-структура (header.payload.signature), `exp`/`sub`/`iat`, HMAC-проверка, refresh token | `02-fastapi/03` |
| 11 | RBAC + rate limiting | `__/5` | `require_role` dependency, sliding window rate limiter (in-memory + Redis) | `02-fastapi/03` |
| 12 | pytest + TestClient | `__/5` | `TestClient(app)`, `dependency_overrides`, fixtures, `@pytest.mark.asyncio`, `AsyncMock` | `02-fastapi/04` |
| 13 | Factories + coverage | `__/5` | factory_boy / свои factory fixtures, `pytest --cov`, что покрывать обязательно | `02-fastapi/04` |
| 14 | Docker deploy | `__/5` | Multi-stage Dockerfile, docker-compose, uvicorn workers, healthcheck | `02-fastapi/04` |

**Суммарная оценка FastAPI: `___/70` → средняя: `__/5`**

---

## Базы данных: PostgreSQL + MySQL

| # | Тема | Оценка (1–5) | Что нужно уметь для оценки 4+ | Материал |
|---|------|-------------|-------------------------------|----------|
| 1 | MVCC в PostgreSQL | `__/5` | xmin/xmax, dead tuples, VACUUM vs VACUUM FULL, autovacuum, bloat | `03-postgresql-mysql/01` |
| 2 | Индексы | `__/5` | B-tree / GIN / BRIN / HASH / partial / covering — когда каждый, синтаксис CREATE INDEX | `03-postgresql-mysql/01` |
| 3 | EXPLAIN ANALYZE | `__/5` | Seq Scan vs Index Scan vs Index Only Scan vs Bitmap Heap Scan, cost, rows, actual time | `03-postgresql-mysql/01` |
| 4 | Оконные функции | `__/5` | `ROW_NUMBER` / `RANK` / `DENSE_RANK` / `LAG` / `LEAD` / `SUM() OVER` — написать без гугла | `03-postgresql-mysql/01` |
| 5 | CTE | `__/5` | Обычные CTE, рекурсивные CTE (WITH RECURSIVE), MATERIALIZED vs NOT MATERIALIZED | `03-postgresql-mysql/01` |
| 6 | Уровни изоляции | `__/5` | READ COMMITTED → REPEATABLE READ → SERIALIZABLE, фантомы, snapshot isolation | `03-postgresql-mysql/01` |
| 7 | PostgreSQL vs MySQL | `__/5` | 6+ архитектурных отличий: MVCC, DDL, JSON, GROUP BY, FULL JOIN, оконные функции | `03-postgresql-mysql/02` |
| 8 | Движки MySQL | `__/5` | InnoDB (транзакции, MVCC) vs MyISAM (нет транзакций, быстрее чтение), когда что | `03-postgresql-mysql/02` |
| 9 | SQL Injection | `__/5` | Параметризованные запросы, экранирование, ORM-санирование, принцп минимальных привилегий | `03-postgresql-mysql/02` |
| 10 | SQL-задачи | `__/5` | JOIN + GROUP BY + оконная функция, написать без гугла за 5 минут | `03-postgresql-mysql/03` |

**Суммарная оценка БД: `___/50` → средняя: `__/5`**

---

## Очереди: RabbitMQ + Kafka

| # | Тема | Оценка (1–5) | Что нужно уметь для оценки 4+ | Материал |
|---|------|-------------|-------------------------------|----------|
| 1 | AMQP-модель | `__/5` | Producer → Exchange → (binding) → Queue → Consumer, типы exchange | `04-rabbitmq/01` |
| 2 | DLX | `__/5` | Dead Letter Exchange — когда срабатывает, как настроить retry через DLX | `04-rabbitmq/01` |
| 3 | Гарантии доставки (RMQ) | `__/5` | Publisher confirms, consumer ack, persistent messages, quorum queues | `04-rabbitmq/02` |
| 4 | Outbox pattern | `__/5` | Проблема, решение, реализация в коде (SQL + Python) | `04-rabbitmq/02` |
| 5 | Kafka: partitions, offsets | `__/5` | Topic → Partition → Consumer Group → Offset, key-hash vs round-robin | `05-kafka/01` |
| 6 | Гарантии доставки (Kafka) | `__/5` | `acks=0/1/all`, idempotent producer, manual vs auto commit, exactly-once | `05-kafka/02` |
| 7 | Rebalancing | `__/5` | Что происходит при rebalance, как минимизировать, CooperativeStickyAssignor | `05-kafka/02` |
| 8 | RabbitMQ vs Kafka | `__/5` | Таблица сравнения (10+ пунктов), когда что выбирать с примерами | `05-kafka/01` |
| 9 | Python-интеграция | `__/5` | aio-pika / aiokafka, lifespan, producer/consumer код без гугла | `04-rabbitmq/03`, `05-kafka/03` |

**Суммарная оценка Очереди: `___/45` → средняя: `__/5`**

---

## Redis

| # | Тема | Оценка (1–5) | Что нужно уметь для оценки 4+ | Материал |
|---|------|-------------|-------------------------------|----------|
| 1 | Структуры данных | `__/5` | String/List/Set/Hash/SortedSet/Stream/HyperLogLog/Bitmap — когда каждую | `06-redis/01` |
| 2 | Кэш-стратегии | `__/5` | Cache Aside / Write Through / Write Behind / Write Around — плюсы/минусы каждой | `06-redis/01` |
| 3 | TTL и инвалидация | `__/5` | `SETEX`, пассивная/активная очистка, versioned keys, event-driven инвалидация | `06-redis/01` |
| 4 | RDB vs AOF | `__/5` | Разница, trade-off, гибридный режим, fsync-политики | `06-redis/02` |
| 5 | Sentinel vs Cluster | `__/5` | HA vs шардирование, когда что, client library поддержка | `06-redis/02` |
| 6 | Rate limiting | `__/5` | Sliding window, token bucket — написать реализацию с Redis | `06-redis/02` |
| 7 | Distributed lock | `__/5` | SETNX + Lua для release, Redlock-алгоритм, TTL и watchdog | `06-redis/02` |
| 8 | Python-интеграция | `__/5` | redis-py / redis.asyncio, pipeline, transaction, lifespan | `06-redis/03` |

**Суммарная оценка Redis: `___/40` → средняя: `__/5`**

---

## Сводная таблица

| Область | Сумма | Средняя | Приоритет (где ≤ 3) |
|---------|-------|---------|---------------------|
| Python core | `___/70` | `__/5` | |
| FastAPI + pytest | `___/70` | `__/5` | |
| Базы данных | `___/50` | `__/5` | |
| Очереди | `___/45` | `__/5` | |
| Redis | `___/40` | `__/5` | |
| **ИТОГО** | `___/275` | `__/5` | |

---

## Карта пробелов — что учить в первую очередь

1. Выпиши все темы с оценкой ≤ 2 (красная зона) — это **обязательно** подтянуть
2. Оценки 3 (жёлтая зона) — подтянуть до 4, если есть время
3. Оценки 4+ (зелёная зона) — только повторение перед собесом

```
Красное (≤ 2):  обязательно → учить в первую очередь
Жёлтое (3):     желательно → подтянуть если есть время
Зелёное (4+):   повторение → прогнать вслух за день до собеса
```

---

> **Совет:** не пытайся поднять всё до 5. 4 везде — это уже отличный результат. Лучше 4 по всем темам, чем 5 по Python и 2 по Kafka. На собесе спросят всё.