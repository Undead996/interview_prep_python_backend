# Подготовка к собеседованию — Python Backend Developer

**Стек:** Python 3.12+ · FastAPI · PostgreSQL · MySQL · RabbitMQ · Apache Kafka · Redis  
**Бонусом:** SQLAlchemy 2.0 · Alembic · Docker · SQL-задачи

Папка собрана под позицию Python-бэкенд-разработчика (middle/senior). Внутри — не «шпаргалка на 5 минут», а полноценный материал:
теория **техническим языком с объяснением "почему"**, примеры кода и команд, ASCII/Mermaid-диаграммы,
мок-интервью с вопросами и ответами, задачи с разбором и шпаргалки на последний день.

---

## Структура папки

```
interview_prep_python_backend/
├── README.md                                     ← навигация + план подготовки
│
├── 00-vacancy/                                   Разбор вакансии и стратегия
│   ├── 01-vacancy-breakdown.md                   Что проверяют за строками вакансии + 3 любимых вопроса
│   ├── 02-self-assessment.md                     Требование → твоя позиция → шкала 1–5 с критериями
│   └── 03-prep-plan.md                           План на 14/7/3 дня + день перед собесом (пошаговый)
│
├── 01-python-core/                               Python (требование №1)
│   ├── 01-language-deep.md                       Типы, ООП, MRO, декораторы, контекстные менеджеры, data classes, аннотации типов — с level-C объяснением
│   ├── 02-async-and-concurrency.md               Async/await, asyncio, GIL, threading vs multiprocessing — с псевдокодом event loop
│   ├── 03-standard-library-deep.md               collections, itertools, typing, dataclasses, enum, pathlib — "какую проблему решает"
│   └── 04-pitfalls-and-tasks.md                  Подводные камни + задачи с разбором (3 задачи)
│
├── 02-fastapi/                                   FastAPI (требование №2)
│   ├── 01-routing-and-dependency-injection.md    Эндпоинты, Depends, middleware, exception handlers, lifespan — с объяснением ASGI-конвейера
│   ├── 02-sqlalchemy-and-database.md             SQLAlchemy 2.0, Alembic, transactions, N+1, connection pool — с Repository Pattern
│   ├── 03-auth-and-security.md                   JWT, OAuth2, HTTPBearer, CORS, hashing, rate limiting — с разбором структуры JWT
│   └── 04-testing-and-deployment.md              TestClient, pytest, Docker, lifespan, async tests — с dependency override
│
├── 03-postgresql-mysql/                          Базы данных (требование №3)
│   ├── 01-postgresql-deep.md                     Типы, MVCC (xmin/xmax), индексы (B-tree/GIN/BRIN), EXPLAIN, VACUUM, партиционирование
│   ├── 02-mysql-deep-and-diff.md                 MySQL vs PostgreSQL: транзакции, движки, locks, типы — разбор архитектурных отличий
│   └── 03-sql-tasks-with-solutions.md            SQL-задачи (8 шт. с полным разбором + "почему это решение")
│
├── 04-rabbitmq/                                  RabbitMQ (требование №4)
│   ├── 01-amqp-concepts.md                       Exchanges, queues, bindings, vhosts, dead letter exchange — AMQP-модель
│   ├── 02-reliability-and-patterns.md            Publisher confirms, consumer ack, retry, outbox, TTL — гарантии доставки
│   └── 03-python-integration-tasks.md            aio-pika/pika, интеграция с FastAPI + задачи
│
├── 05-kafka/                                     Apache Kafka (требование №5)
│   ├── 01-kafka-concepts.md                      Topics, partitions, producers, consumers, offsets, Kafka vs RabbitMQ — "почему Kafka быстрый"
│   ├── 02-reliability-and-consumption.md         At-least-once, exactly-once, idempotency, rebalancing — offset management
│   └── 03-python-integration-tasks.md            aiokafka/confluent-kafka, интеграция + задачи
│
├── 06-redis/                                     Redis (требование №6)
│   ├── 01-data-structures-and-caching.md         Strings, lists, sets, hashes, streams, cache strategies, TTL — когда что выбирать
│   ├── 02-persistence-and-advanced.md            RDB/AOF, Sentinel, Cluster, rate limiting, locks, pub/sub — с Lua-script
│   └── 03-python-integration-tasks.md            redis-py/aioredis, интеграция + задачи
│
├── mock-interview/                               Мок-интервью: вопросы + ответы
│   ├── 01-hr-screening.md                        HR/скрининг — STAR-история, ожидания, рассказ о себе
│   ├── 02-python-deep.md                         Python (~30 вопросов с расширенными ответами)
│   ├── 03-fastapi.md                             FastAPI (~20 вопросов с расширенными ответами)
│   ├── 04-postgresql-mysql.md                    БД (~20 вопросов с расширенными ответами)
│   ├── 05-rabbitmq-kafka.md                      Очереди (~20 вопросов с расширенными ответами)
│   ├── 06-redis.md                               Redis (~15 вопросов с расширенными ответами)
│   ├── 07-live-coding.md                         Live-coding: 3 задачи с решением и советами
│   └── 08-behavioral-star.md                     Поведенческие вопросы + 3 STAR-заготовки с разбором
│
├── tasks/                                        Практические задачи (с разбором)
│   ├── 01-python-tasks.md                        Python: функции, async, декораторы (3 задачи с разбором)
│   ├── 02-fastapi-tasks.md                       FastAPI: DI, SQLAlchemy, валидация (3 задачи с разбором)
│   └── 03-queue-tasks.md                         RabbitMQ/Kafka: дизайн и интеграция (3 задачи с разбором)
│
└── cheatsheets/                                  Повторение за день до собеса
    ├── python-cheatsheet.md                      Python / async на один экран
    ├── fastapi-cheatsheet.md                     FastAPI + SQLAlchemy команды
    ├── postgres-mysql-cheatsheet.md              SQL-диагностика и синтаксис
    ├── rabbitmq-cheatsheet.md                    RabbitMQ: exchanges, CLI, ack/requeue
    ├── kafka-cheatsheet.md                       Kafka: topics, consumer groups, CLI
    └── redis-cheatsheet.md                       Redis: типы, команды, TTL, кэш
```

---

## Как пользоваться (по времени до собеса)

### Если 14+ дней
| День | Тема | Материалы |
|------|------|-----------|
| 0 | Понять, что хотят | `00-vacancy/*` |
| 1–2 | Python core | `01-python-core/*` + `tasks/01-python-tasks.md` |
| 3–4 | FastAPI | `02-fastapi/*` + `tasks/02-fastapi-tasks.md` |
| 5–6 | PostgreSQL + MySQL | `03-postgresql-mysql/*` |
| 7–8 | RabbitMQ | `04-rabbitmq/*` |
| 9–10 | Apache Kafka | `05-kafka/*` |
| 11 | Redis | `06-redis/*` |
| 12 | Повторение + очереди (задачи) | `tasks/03-queue-tasks.md` |
| 13 | Мок-интервью | `mock-interview/*` — **вслух** |
| 14 | Повторение | `cheatsheets/*` |

### Если 5–7 дней
1. `00-vacancy/01` + `00-vacancy/02` — 30 мин
2. `mock-interview/02-python-deep.md`, `03-fastapi.md`, `04-postgresql-mysql.md` — вслух
3. `tasks/01-python-tasks.md`, `02-fastapi-tasks.md` — решить на бумаге
4. `cheatsheets/*` — пролистать

### День перед собесом
**Не учить новое.** Прогнать `cheatsheets/*`, потом `mock-interview/07-live-coding.md` и
`mock-interview/08-behavioral-star.md` вслух.

---

## Три вещи, которые решают исход этого собеса

1. **Async + FastAPI.** Ожидается, что ты не просто «написал API на FastAPI», а понимаешь async/await, lifespan, dependency injection, SQLAlchemy 2.0 async sessions.
2. **Базы данных + очереди.** PostgreSQL vs MySQL — частый вопрос на миграции. RabbitMQ vs Kafka — когда что выбирать. Покажи понимание trade-off.
3. **Системный дизайн.** «Как спроектировать нотификации» — здесь надо показать, что умеешь комбинировать: FastAPI → RabbitMQ/Kafka → PostgreSQL → Redis cache.

---

## Как читать записи в файлах

> **Простыми словами** — объяснение на бытовой аналогии.  
> **Технически** — точная формулировка с деталями.  
> **На собесе** — как это прозвучит в ответе и что за этим последует.
>
> Формат: ✅ хорошо / ⚠️ осторожно / ❌ так не говори.

Каждый файл содержит не просто "что", а **"почему"** — объяснение внутреннего устройства, trade-off и когда что выбирать.