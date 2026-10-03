# План подготовки — пошаговый маршрут

> **Цель:** не «что почитать», а **какие именно навыки прокачать** за каждую тренировку, с проверочными заданиями.

---

## Как пользоваться этим планом

Каждый день содержит:
- **Цель** — что должно быть в голове к концу дня
- **Что сделать** — конкретные файлы, разделы, задачи
- **Как проверить** — конкретное действие: ответить вслух, написать код, решить задачу

После каждой тренировки возвращайся к `02-self-assessment.md` и обновляй оценки.

---

# План А: 14 дней (полный, оптимальный)

---

## День 1. Разбор вакансии и самооценка (1 час)

**Цель:** понять, что спрашивают, и честно оценить свой уровень.

```
Что сделать:
1. Прочитать `00-vacancy/01-vacancy-breakdown.md` — расшифровка требований
2. Заполнить `00-vacancy/02-self-assessment.md` — все оценки 1-5
3. Выписать 3-5 тем с оценкой ≤ 2 — это твой фокус
```

**Результат:** ты знаешь свой真实ный уровень и 3-5 слабых мест. План дальше строится с приоритетом на них.

**Проверка:** сможешь ли ты назвать 3 главных отличия PG от MySQL прямо сейчас?

---

## День 2. Python: типы, ООП, декораторы (2 часа)

**Цель:** уверенно объяснить модель памяти, MRO, написать декоратор с аргументами.

```
Что сделать:
1. `01-python-core/01-language-deep.md` — прочитать:
   §1 Типы и изменяемость (таблица, tuple с list, почему dict-key)
   §2 ООП и MRO (C3 linearization, super(), diamond problem)
   §3 Декораторы (3 уровня, @wraps, класс-декоратор)
   §4 Контекстные менеджеры (класс vs @contextmanager)
   §5 Data classes (field, frozen, slots)
   §6 Аннотации типов (Protocol vs ABC, TypeAlias, Generic)

2. Написать код:
   - Декоратор @retry(max_attempts=3, delay=0.5)
   - Контекстный менеджер ManagedSession (enter → commit/rollback → close)
   - Dataclass с __post_init__ валидацией
```

**Проверка:** устно ответь на Q1-Q7 из `mock-interview/02-python-deep.md`. Если плаваешь в MRO — перечитай и нарисуй дерево наследования.

---

## День 3. Python: async, GIL, asyncio (2 часа)

**Цель:** объяснить event loop, отличие Task от Future, написать producer-consumer.

```
Что сделать:
1. `01-python-core/02-async-and-concurrency.md` — прочитать:
   §1 GIL (почему, что блокирует, как обойти)
   §2 asyncio vs threading vs multiprocessing (таблица)
   §3 Event loop (epoll/kqueue/IOCP, как работает await)
   §4 Coroutine/Task/Future (разница, когда что)
   §5 Producer-Consumer с asyncio.Queue + Semaphore
   §6 Подводные камни (time.sleep в async def, gather без return_exceptions)

2. `01-python-core/03-standard-library-deep.md` — прочитать:
   §1 collections (defaultdict, Counter, deque, ChainMap)
   §2 itertools (product, cycle, groupby, chain, batched)
   §6 functools (lru_cache, partial, singledispatch)

3. Написать код:
   - Producer-consumer с asyncio.Queue(maxsize=100) и 3 workers
   - Rate limiter с asyncio.Semaphore
```

**Проверка:** напиши код, который конкурентно качает 10 URL через asyncio.Semaphore(3) и httpx.AsyncClient.

---

## День 4. FastAPI: routing, DI, middleware, lifespan (2 часа)

**Цель:** написать эндпоинт с Depends, middleware, exception handler, lifespan.

```
Что сделать:
1. `02-fastapi/01-routing-and-dependency-injection.md`:
   §1 Как FastAPI обрабатывает запрос (ASGI-конвейер)
   §2 Dependency Injection — Depends() 3 способа
   §3 Middleware — порядок, CORS, кастомный logging
   §4 Exception handlers — кастомные AppError
   §5 Lifespan — startup/shutdown для engine/Redis/Kafka

2. Написать код:
   - CRUD-эндпоинт с Depends(get_db)
   - Middleware для X-Process-Time
   - Custom exception handler для AppError
   - Lifespan с engine + Redis
```

**Проверка:** ответь на Q1-Q5 из `mock-interview/03-fastapi.md`.

---

## День 5. FastAPI: SQLAlchemy 2.0, транзакции, N+1 (2 часа)

**Цель:** написать async-запрос с joinedload, транзакцию с savepoint.

```
Что сделать:
1. `02-fastapi/02-sqlalchemy-and-database.md`:
   §1 Async engine + session (pool_size, max_overflow, pool_pre_ping)
   §2 Repository pattern
   §3 N+1 problem (4 решения: joinedload, selectinload, subqueryload, raiseload)
   §4 Транзакции (begin/commit/savepoint/rollback)
   §5 Connection pool, Alembic (init, autogenerate, upgrade/downgrade)

2. `02-fastapi/03-auth-and-security.md`:
   §1 JWT (header.payload.signature, exp/sub/iat)
   §2 OAuth2 Password Flow
   §3 Rate limiting (in-memory + Redis)

3. Написать код:
   - Repository с get_by_id / list / create
   - Запрос с selectinload для Order.items
   - Транзакцию с savepoint (begin_nested)
```

**Проверка:** напиши запрос к БД, который загружает User + его Orders + Items каждого Order. Сколько запросов? Как уменьшить?

---

## День 6. FastAPI: pytest, TestClient, Docker (2 часа)

**Цель:** написать интеграционный тест с TestClient + dependency override.

```
Что сделать:
1. `02-fastapi/04-testing-and-deployment.md`:
   §1 TestClient (with lifespan, async tests)
   §2 Dependency override (in-memory SQLite)
   §3 Contract tests (OpenAPI schema)
   §4 Docker multi-stage + docker-compose
   §5 CI/CD (GitHub Actions)

2. Написать код:
   - Фикстуру test_db (in-memory SQLite + create_all)
   - Тест на создание + получение пользователя
   - Тест на 404 (несуществующий пользователь)
   - Contract test: все эндпоинты имеют summary
```

**Проверка:** запусти `pytest --cov` и убедись, что coverage > 80% для твоего тестового приложения.

---

## День 7–8. PostgreSQL: MVCC, индексы, EXPLAIN (2 × 2 часа)

**Цель:** читать EXPLAIN ANALYZE, объяснить MVCC до dead tuples, написать 3 индекса под задачу.

```
Что сделать:
1. `03-postgresql-mysql/01-postgresql-deep.md`:
   §1 MVCC (xmin/xmax, dead tuples, VACUUM vs VACUUM FULL)
   §2 Индексы (B-tree / GIN / BRIN / HASH / partial / covering) — с синтаксисом
   §3 EXPLAIN ANALYZE — примеры вывода, как читать cost/rows/time
   §4 CTE + оконные функции
   §5 Уровни изоляции
   §6 Партиционирование

2. `03-postgresql-mysql/02-mysql-deep-and-diff.md`:
   §1 Сравнительная таблица (20+ пунктов)
   §2 Движки MySQL (InnoDB vs MyISAM)
   §3 Когда мигрировать с MySQL на PG

3. `03-postgresql-mysql/03-sql-tasks-with-solutions.md`:
   Решить задачи 1-8 (минимум 5)
```

**Проверка:** напиши CREATE INDEX для полнотекстового поиска по полю body (JSONB). Какой индекс? Почему GIN?

---

## День 9. RabbitMQ: AMQP, DLX, outbox (2 часа)

**Цель:** объяснить AMQP-модель, написать consumer с retry + DLX.

```
Что сделать:
1. `04-rabbitmq/01-amqp-concepts.md`:
   §1 AMQP-модель (Producer → Exchange → Queue → Consumer)
   §2 Типы exchange (direct/fanout/topic/headers)
   §3 DLX (когда срабатывает, настройка)

2. `04-rabbitmq/02-reliability-and-patterns.md`:
   §1 Publisher confirms
   §2 Consumer ack/nack/reject
   §3 Outbox pattern (полная реализация)

3. `04-rabbitmq/03-python-integration-tasks.md`:
   aio-pika producer/consumer код
```

**Проверка:** нарисуй схему: FastAPI → outbox-таблица → poller → RabbitMQ → consumer → DB. Где что может упасть и как восстанавливаемся?

---

## День 10. Kafka: partitions, offsets, exactly-once (2 часа)

**Цель:** объяснить partition, offset management, гарантии доставки.

```
Что сделать:
1. `05-kafka/01-kafka-concepts.md`:
   §1 Topic → Partition → Consumer Group → Offset
   §2 Producer (key-hash, round-robin)
   §3 Kafka vs RabbitMQ (таблица сравнения)

2. `05-kafka/02-reliability-and-consumption.md`:
   §1 At-least-once / exactly-once (acks + idempotent)
   §2 Offset management (manual commit)
   §3 Rebalancing

3. `05-kafka/03-python-integration-tasks.md`:
   aiokafka producer/consumer код
```

**Проверка:** ответь на Q1-Q6 из `mock-interview/05-rabbitmq-kafka.md`.

---

## День 11. Redis: структуры, кэш, persistence (2 часа)

**Цель:** написать rate limiter и distributed lock с Redis.

```
Что сделать:
1. `06-redis/01-data-structures-and-caching.md`:
   §1 8 структур (таблица «когда что»)
   §2 Кэш-стратегии (Cache Aside, Write Through, Write Behind)
   §3 TTL, инвалидация

2. `06-redis/02-persistence-and-advanced.md`:
   §1 RDB vs AOF (trade-off)
   §2 Sentinel vs Cluster
   §3 Rate limiting (sliding window)
   §4 Distributed lock (SETNX + Lua)

3. `06-redis/03-python-integration-tasks.md`:
   redis.asyncio, pipeline, transaction, lifespan
```

**Проверка:** напиши rate limiter middleware для FastAPI с Redis: 100 запросов в минуту по IP. Какой ключ? Какой TTL?

---

## День 12. Задачи: Python + FastAPI + очереди (2 часа)

**Цель:** прорешать задачи, чтобы закрыть пробелы.

```
Что сделать:
1. `tasks/01-python-tasks.md` — решить задачи 1-5
2. `tasks/02-fastapi-tasks.md` — решить задачи 1-5
3. `tasks/03-queue-tasks.md` — решить задачи 1-5

Решай на бумаге или в редакторе. Потом сверяй с решением.
```

**Проверка:** если какая-то задача заняла > 10 минут — перечитай соответствующий раздел.

---

## День 13. Мок-интервью вслух (3 часа)

**Цель:** проговорить ответы на все вопросы так, как будто ты на собесе.

```
Порядок (каждый блок — 20-30 минут):
1. mock-interview/01-hr-screening.md — рассказ о себе (2 мин)
2. mock-interview/02-python-deep.md — 30 вопросов вслух
3. mock-interview/03-fastapi.md — 20 вопросов вслух
4. mock-interview/04-postgresql-mysql.md — 20 вопросов вслух
5. mock-interview/05-rabbitmq-kafka.md — 20 вопросов вслух
6. mock-interview/06-redis.md — 15 вопросов вслух
7. mock-interview/07-live-coding.md — решить 5 задач на бумаге
8. mock-interview/08-behavioral-star.md — рассказать 5 STAR-историй
```

**Критически важно:** не читай с экрана! Закрой файл, расскажи своими словами. Если где-то плаваешь — открой материал, перечитай и расскажи ещё раз.

---

## День 14. Cheatsheets + финальный прогон (2 часа)

**Цель:** освежить всё, не учить новое.

```
План:
1. cheatsheets/* — пролистать все 6 файлов (5 мин каждый = 30 мин)
2. mock-interview/07-live-coding.md — решить 2 задачи (30 мин)
3. mock-interview/08-behavioral-star.md — рассказать 3 истории (30 мин)
4. 00-vacancy/02-self-assessment.md — обновить оценки (30 мин)
```

**Не учи новое!** Если наткнулся на незнакомое — запиши в «на будущее», не пытайся выучить за день.

---

# План Б: 7 дней (интенсив)

Если времени мало — сжимаем до самого важного. Каждый день = 4 часа (2 утром + 2 вечером).

| День | Утро (2 ч) | Вечер (2 ч) | Ключевая проверка |
|------|-----------|-------------|-------------------|
| **1** | Python: типы, MRO, декораторы, контекстные менеджеры, data classes | Python: GIL, asyncio, Task/Future, Queue, примитивы | Написать producer-consumer с asyncio.Queue |
| **2** | FastAPI: Depends, middleware, lifespan, exception handlers | FastAPI: SQLAlchemy 2.0 async, N+1, транзакции, JWT | Написать CRUD с Depends + joinedload |
| **3** | PostgreSQL: MVCC, индексы, EXPLAIN | MySQL diff + SQL-задачи (минимум 5) | Прочитать EXPLAIN ANALYZE запроса |
| **4** | RabbitMQ: AMQP, DLX, outbox | Kafka: partitions, offset, exactly-once | Нарисовать outbox-архитектуру |
| **5** | Redis: структуры, кэш-стратегии, rate limiting | Mock-interview: Python 30 вопросов вслух | Написать rate limiter с Redis |
| **6** | Mock-interview: FastAPI 20 + БД 20 вслух | Задачи: Python (5) + FastAPI (5) | Решить каждую задачу ≤ 10 мин |
| **7** | Mock-interview: очереди 20 + Redis 15 вслух | Cheatsheets + live-coding 5 задач + STAR | Проговорить все cheatsheets вслух |

---

# План В: 3 дня (SOS — «пожарный» режим)

Только самое критичное. Учим отвечать на вопросы, а не писать идеальный код.

| День | Что делаем (6 часов) | Формат |
|------|---------------------|--------|
| **1** | Python core (типы, MRO, декораторы, async) + FastAPI core (DI, middleware, lifespan, N+1) + JWT | Читать → сразу отвечать вслух. Писать минимальный код. |
| **2** | PostgreSQL (MVCC, индексы, EXPLAIN) + RabbitMQ/Kafka (только различия и когда что) + Redis (структуры, кэш, rate limiting) | Только теория и ready-made ответы. Без deep-dive. |
| **3** | Mock-interview ВСЛУХ (Python 30 + FastAPI 20 + БД 20) + Cheatsheets + Live-coding 2 задачи | Проговаривать, проговаривать, проговаривать. |

---

# План Г: 1 день (день перед собесом)

**Не учить новое! Только повторение и разогрев.**

```
08:00-08:30  Cheatsheets: python → fastapi → postgres → rabbitmq → kafka → redis
08:30-09:00  Live-coding: 2 задачи (SQL + Python) на бумаге
09:00-09:30  STAR-истории: 3 истории вслух (баг, инициатива, конфликт)
09:30-10:00  Рассказ о себе (2-3 варианта) + ответы на HR-вопросы

Весь день:
- Лёгкое повторение (cheatsheets)
- Проговаривание сложных тем вслух
- Проверка оборудования (камера, микрофон, интернет)

20:00  Спать. Серьёзно. Сон > ещё один файл.
```

---

## Чеклист на день собеса

```
Техническое:
□ Ноутбук заряжен (100% или на зарядке)
□ Интернет стабильный (проверь speedtest)
□ Камера и микрофон работают
□ Ссылка на созвон открыта
□ Редактор кода открыт (для live-coding)
□ Ручка + 2 листа бумаги

Документы:
□ Паспорт
□ СНИЛС / ИНН (если нужно)
□ Трудовая книжка / выписка (если просили)

Личное:
□ Стакан воды
□ Тихое помещение (без фона)
□ Телефон в беззвучном режиме
□ За 30 мин до — не учить, просто дышать
```

---

> **Главное правило:** на собесе важнее показать ход мыслей, чем идеальный код. Если не помнишь синтаксис — скажи «точно не помню, но идея такая», и объясни подход. Интервьюер оценит мышление выше, чем запоминание.