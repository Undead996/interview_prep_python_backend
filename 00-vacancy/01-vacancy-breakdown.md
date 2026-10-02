# Разбор вакансии Python Backend Developer

## Типичная вакансия

```
Компания N ищет backend-разработчика (Python).
Стек: Python 3.12+, FastAPI, SQLAlchemy 2.0, PostgreSQL,
RabbitMQ / Kafka, Redis.
Плюсом: async/await, Docker, Alembic.

Ожидания: Middle / Senior.
```

## Что реально проверяют

| Что написано | Что имеют в виду | Как готовиться |
|---|---|---|
| **Python 3.12+** | Типы, ООП, async/await, GIL, декораторы, менеджеры контекста | `01-python-core/*` |
| **FastAPI** | Dependency Injection, Pydantic, middleware, аутентификация, lifespan | `02-fastapi/*` |
| **SQLAlchemy 2.0** | Async sessions, N+1, транзакции, connection pool | `02-fastapi/02-sqlalchemy-and-database.md` |
| **PostgreSQL / MySQL** | Различия, MVCC, индексы, EXPLAIN, SQL-задачи | `03-postgresql-mysql/*` |
| **RabbitMQ** | AMQP, exchanges, confirms, DLX, retry, интеграция | `04-rabbitmq/*` |
| **Kafka** | Topics, partitions, consumer groups, at-least-once, exactly-once | `05-kafka/*` |
| **Redis** | Структуры, кэш-стратегии, TTL, pub/sub, rate limiting | `06-redis/*` |
| **Docker** | Compose для тестового окружения, multi-stage | бонус |

## Типичные этапы собеса

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

## Три favourite-вопроса

1. **PostgreSQL vs MySQL** — «У нас MySQL, переезжаем на PostgreSQL. Что изменится?»
2. **RabbitMQ vs Kafka** — «Когда что выбирать?»
3. **«Напиши SQL-запрос с тремя JOIN и оконной функцией»** — классика.