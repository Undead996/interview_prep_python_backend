# Подготовка к собеседованию — Python Backend Developer

**Стек:** Python 3.12+ · FastAPI · PostgreSQL · MySQL · RabbitMQ · Apache Kafka · Redis  
**Тестирование:** pytest · pytest-asyncio · TestClient · factory_boy · coverage  
**Бонусом:** SQLAlchemy 2.0 · Alembic · Docker · SQL-задачи · asyncio

Эта папка — не просто «шпаргалка», а **полноценный учебный курс** для подготовки к собеседованию на позицию Python Backend Developer (middle/senior). Каждый файл содержит:

- **Теорию** — объяснение «почему» с уровнем детализации C (как под капотом)
- **Примеры кода** — рабочие, копируй и запускай
- **ASCII/Mermaid-диаграммы** — визуализация архитектуры и потоков данных
- **Мок-интервью** — как отвечать вслух, что спросят следом
- **Задачи** — с полным разбором и альтернативными решениями
- **Шпаргалки** — на последний день, сжато но полно

---

## Почему этот материал работает

### 1. Он построен от «почему», а не от «что»

Вместо «список immutable-типов» ты узнаешь, **почему** tuple можно ключом словаря, а list — нет. Вместо «как написать Depends()» — **почему** DI важен и как FastAPI использует его внутри.

### 2. Он готовит к реальному собесу

Каждый раздел содержит секцию **«На собесе»** — готовые фразы для ответа + подсказки, что спросят дальше.

### 3. Он заточен под middle/senior

Материал предполагает, что ты уже пишешь на Python. Мы не учим синтаксис — мы разбираем **глубину**: внутреннее устройство, trade-off, подводные камни.

---

## Структура папки

```
interview_prep_python_backend/
├── README.md                                         ← навигация + 3 плана подготовки
│
├── 00-vacancy/                                       Разбор вакансии и стратегия
│   ├── 01-vacancy-breakdown.md                       Что проверяют за строками вакансии + таблица глубин
│   ├── 02-self-assessment.md                         Требование → оценка 1–5 с критериями + карта пробелов
│   └── 03-prep-plan.md                               План на 14 / 7 / 3 / 1 день (пошаговый, почасовой)
│
├── 01-python-core/                                   Python — требование №1
│   ├── 01-language-deep.md                           Типы, память, ООП, MRO, C3 linearization, декораторы (3 уровня), контекстные менеджеры, data classes, аннотации типов — с level-C объяснением
│   ├── 02-async-and-concurrency.md                   GIL (почему, когда мешает, как обойти), asyncio, event loop (epoll/kqueue/IOCP), Task/Future/Coroutine, gather/TaskGroup, async Queue, примитивы синхронизации
│   ├── 03-standard-library-deep.md                   collections, itertools, typing, dataclasses, enum, pathlib, functools — «какую проблему решает» + примеры из реального бэкенда
│   └── 04-pitfalls-and-tasks.md                      Топ-10 подводных камней + 5 задач с level-C разбором
│
├── 02-fastapi/                                       FastAPI + pytest — требование №2
│   ├── 01-routing-and-dependency-injection.md        Эндпоинты, Depends (3 способа), middleware, exception handlers, lifespan — с диаграммой ASGI-конвейера
│   ├── 02-sqlalchemy-and-database.md                 Async engine/session, Repository pattern, N+1 (4 способа решения), транзакции, connection pool, Alembic
│   ├── 03-auth-and-security.md                       JWT (структура, проверка, refresh), OAuth2, RBAC, CORS, rate limiting, security headers
│   └── 04-testing-and-deployment.md                  pytest + pytest-asyncio, TestClient, dependency override, factories, contract tests, Docker multi-stage, CI/CD
│
├── 03-postgresql-mysql/                              Базы данных — требование №3
│   ├── 01-postgresql-deep.md                         MVCC (xmin/xmax, dead tuples, VACUUM), индексы (B-tree/GIN/BRIN/HASH/partial/covering), EXPLAIN с примерами вывода, CTE, оконные функции, партиционирование, уровни изоляции
│   ├── 02-mysql-deep-and-diff.md                     MySQL vs PostgreSQL: полная сравнительная таблица (20+ пунктов), когда мигрировать, подводные камни
│   └── 03-sql-tasks-with-solutions.md                12 SQL-задач с полным разбором + альтернативные решения + «почему это решение лучше»
│
├── 04-rabbitmq/                                      RabbitMQ — требование №4
│   ├── 01-amqp-concepts.md                           AMQP-модель (глубоко), exchange-типы с диаграммами, queues, bindings, vhosts, channels, DLX
│   ├── 02-reliability-and-patterns.md                Publisher confirms, consumer ack/nack/reject, outbox pattern (полная реализация), retry + backoff, TTL, quorum queues
│   └── 03-python-integration-tasks.md                aio-pika (producer/consumer/robust), FastAPI + RabbitMQ (lifespan), 3 задачи с решениями
│
├── 05-kafka/                                         Apache Kafka — требование №5
│   ├── 01-kafka-concepts.md                          Topic/Partition/Consumer Group/Offset, producer (key-hash, round-robin), Kafka vs RabbitMQ (детальная таблица)
│   ├── 02-reliability-and-consumption.md             At-least-once / exactly-once, idempotent producer, offset management, rebalancing, compaction
│   └── 03-python-integration-tasks.md                aiokafka/confluent-kafka, FastAPI + Kafka, 3 задачи с решениями
│
├── 06-redis/                                         Redis — требование №6
│   ├── 01-data-structures-and-caching.md             Все 8 структур данных + когда какую, кэш-стратегии (Cache Aside / Write Through / Write Behind / Write Around), TTL, инвалидация
│   ├── 02-persistence-and-advanced.md                RDB vs AOF (детально), Sentinel vs Cluster, Lua-скриптинг, distributed locks (Redlock), rate limiting (sliding window / token bucket)
│   └── 03-python-integration-tasks.md                redis-py / redis.asyncio, FastAPI + Redis (lifespan, cache middleware), 4 задачи с решениями
│
├── mock-interview/                                   Мок-интервью: вопросы + полные ответы
│   ├── 01-hr-screening.md                            HR/скрининг — рассказ о себе, зарплата, STAR-история
│   ├── 02-python-deep.md                             Python: 30 вопросов с развёрнутыми ответами (по 5–10 строк каждый)
│   ├── 03-fastapi.md                                 FastAPI: 20 вопросов с развёрнутыми ответами
│   ├── 04-postgresql-mysql.md                        БД: 20 вопросов с развёрнутыми ответами
│   ├── 05-rabbitmq-kafka.md                          Очереди: 20 вопросов с развёрнутыми ответами
│   ├── 06-redis.md                                   Redis: 15 вопросов с развёрнутыми ответами
│   ├── 07-live-coding.md                             Live-coding: 5 задач с полным решением + советы
│   └── 08-behavioral-star.md                         Поведенческие: 5 STAR-заготовок с разбором
│
├── tasks/                                            Практические задачи (с полным разбором)
│   ├── 01-python-tasks.md                            Python: 5 задач (async, декораторы, генераторы, asyncio.Queue)
│   ├── 02-fastapi-tasks.md                           FastAPI: 5 задач (CRUD, JWT, DI, contract test, rate limiter)
│   └── 03-queue-tasks.md                             RabbitMQ/Kafka: 5 задач (outbox, retry, DLQ, проектирование)
│
└── cheatsheets/                                      Повторение за день до собеса
    ├── python-cheatsheet.md                          Python / async на один экран (2 стр.)
    ├── fastapi-cheatsheet.md                         FastAPI + SQLAlchemy + pytest команды
    ├── postgres-mysql-cheatsheet.md                  SQL-диагностика и синтаксис
    ├── rabbitmq-cheatsheet.md                        RabbitMQ: exchanges, CLI, ack/nack/reject
    ├── kafka-cheatsheet.md                           Kafka: topics, consumer groups, CLI
    └── redis-cheatsheet.md                           Redis: типы, команды, TTL, кэш
```

---

## Как пользоваться (зависит от времени)

### Если 14+ дней — полный план

| День | Тема | Материалы | Время |
|------|------|-----------|-------|
| 0 | Понять, что хотят | `00-vacancy/*` | 1 ч |
| 1–2 | Python core (типы, ООП, MRO, декораторы) | `01-python-core/01` + `04` | 2 × 2 ч |
| 3 | Python core (async, GIL, asyncio) | `01-python-core/02` + `03` | 2 ч |
| 4–5 | FastAPI (routing, DI, middleware, lifespan) | `02-fastapi/01` + `03` | 2 × 2 ч |
| 6 | FastAPI (SQLAlchemy, N+1, транзакции) | `02-fastapi/02` | 2 ч |
| 7 | FastAPI (pytest, TestClient, CI/CD) | `02-fastapi/04` | 2 ч |
| 8–9 | PostgreSQL + MySQL | `03-postgresql-mysql/*` | 2 × 2 ч |
| 10 | RabbitMQ | `04-rabbitmq/*` | 2 ч |
| 11 | Apache Kafka | `05-kafka/*` | 2 ч |
| 12 | Redis | `06-redis/*` | 2 ч |
| 13 | Мок-интервью **вслух** | `mock-interview/*` | 3 ч |
| 14 | Повторение | `cheatsheets/*` + live-coding | 2 ч |

### Если 7 дней — интенсив

| День | Утро (2 ч) | Вечер (2 ч) |
|------|-----------|-------------|
| 1 | Python: типы, ООП, декораторы, async | Python: задачи (`tasks/01`) |
| 2 | FastAPI: DI, middleware, lifespan | FastAPI: SQLAlchemy, JWT, тесты |
| 3 | PostgreSQL: MVCC, индексы, EXPLAIN | MySQL diff + SQL-задачи |
| 4 | RabbitMQ: AMQP, DLX, outbox | Kafka: partitions, offset, exactly-once |
| 5 | Redis: структуры, кэш, rate limiting | Mock-interview: Python + FastAPI вслух |
| 6 | Mock-interview: БД + очереди вслух | Задачи: `tasks/02` + `tasks/03` |
| 7 | Cheatsheets | Live-coding + STAR |

### Если 3 дня — SOS

| День | Что делаем | Как |
|------|-----------|-----|
| 1 | Python core (типы, MRO, декораторы, async) + FastAPI core (DI, middleware, lifespan) | Читать + отвечать вслух. Писать код. |
| 2 | PostgreSQL (MVCC, индексы, EXPLAIN) + RabbitMQ/Kafka (AMQP, DLX, partitions) + Redis (структуры, кэш) | Только ключевое, без деталей |
| 3 | Mock-interview ВСЛУХ + cheatsheets + live-coding | Проговаривать, не учить новое |

### День перед собесом — чеклист

```
□ 08:00 — Пройти cheatsheets (30 мин)
□ 09:00 — Live-coding: 2 задачи «на доске» (30 мин)
□ 10:00 — STAR-истории вслух (30 мин)
□ Весь день — лёгкое повторение, НЕ учить новое
□ 20:00 — Спать (сон важнее ещё одного файла)
□ Батарейка ноутбука 100%
□ Ссылка на созвон открыта и проверена
□ Наушники/микрофон работают
□ Стакан воды на столе
□ Ручка + бумага для live-coding
```

---

## Три вещи, которые решают исход собеса

### 1. Async + FastAPI → покажи глубину
Ожидается, что ты не просто «написал API», а понимаешь:
- Как работает event loop (epoll/kqueue/IOCP)
- Почему `time.sleep()` в `async def` — катастрофа
- Как dependency injection упрощает тестирование (dependency_overrides)
- Как устроен ASGI-конвейер (Uvicorn → Starlette → FastAPI)
- Как работает lifespan и почему он заменил `@app.on_event`

### 2. Базы данных + очереди → покажи trade-off
- PostgreSQL vs MySQL — не синтаксис, а **архитектура** (MVCC, VACUUM vs undo log, DDL-транзакции)
- RabbitMQ vs Kafka — не «оба брокеры», а **разные модели** (smart broker vs smart consumer)
- Индексы — не «создаю и забываю», а понимание B-tree vs GIN vs BRIN vs partial

### 3. Системный дизайн → покажи комбинацию
«Как спроектировать нотификации?» — здесь нужно:
- FastAPI принимает запрос → пайплайн
- Outbox pattern (БД + очередь атомарно)
- RabbitMQ/Kafka для доставки → выбор брокера с обоснованием
- Redis для rate limiting и кэширования
- Consumer с retry + DLQ/DLX

---

## Как читать записи в файлах

Каждый файл использует единый формат для быстрого усвоения:

> **Простыми словами** — объяснение на бытовой аналогии (как если бы объяснял джуниору).
>
> **Технически** — точная формулировка с деталями уровня CPython / базы данных.
>
> **На собесе** — готовая фраза для ответа + что спросят следующим вопросом.
>
> **⚠️ Осторожно** — типичная ошибка, которую делают даже опытные.
>
> **✅ Правильно** — как делать.

Формат ответа: **✅ хорошо** / **⚠️ осторожно** / **❌ так не говори**.

---

## Ключевые правила подготовки

1. **Не читай молча — проговаривай вслух.** На собесе будет именно так. Если не можешь объяснить своими словами — ты не знаешь тему.
2. **Пиши код.** Каждую задачу реши на бумаге или в редакторе. Не просто прочитай решение.
3. **Задавай вопросы себе.** После каждого раздела спроси: «А что будет, если...?» — и попробуй ответить.
4. **Не учи новое в последний день.** Только повторение и проговаривание.
5. **Спи.** Уставший мозг на собесе работает на 50% мощности.

---

## Обновления в этой версии

- **pytest / pytest-asyncio** — полноценный материал по тестированию
- **Больше примеров** — 2–3 примера на каждую концепцию вместо одного
- **Больше диаграмм** — ASCII-схемы для всех ключевых архитектур
- **Больше задач** — 8 → 12 SQL-задач, 3 → 5 Python-задач, и т.д.
- **Глубже «почему»** — level-C объяснения внутреннего устройства
- **Mock-interview** — каждый ответ расширен до 4–8 строк с примерами кода