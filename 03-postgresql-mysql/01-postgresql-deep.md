# PostgreSQL — глубокое погружение

---

## Типы данных PostgreSQL

| Тип | PostgreSQL | Особенность |
|---|---|---|
| INTEGER / BIGINT | `int`, `int64` | Стандарт |
| VARCHAR(n) / TEXT | `str` | TEXT = VARCHAR без лимита |
| TIMESTAMP / TIMESTAMPTZ | `datetime` | Всегда хранить UTC |
| JSONB | `dict` | С индексами (GIN) |
| UUID | `uuid` | Первичные ключи |
| ARRAY | `list` | `TEXT[]`, `INT[]` |
| ENUM | Кастомный | CREATE TYPE |

---

## MVCC (Multiversion Concurrency Control)

> **Простыми словами:** Каждая транзакция видит «снимок» базы на момент своего начала.
> Читатели не блокируют писателей (и наоборот).

```sql
-- Каждая строка содержит:
-- xmin — id транзакции, создавшей строку
-- xmax — id транзакции, удалившей строку

-- Мёртвые строки (dead tuples):
SELECT relname, n_dead_tup, n_live_tup
FROM pg_stat_user_tables
WHERE relname = 'orders';

-- VACUUM — чистит мёртвые строки (не блокирует запись)
VACUUM orders;

-- VACUUM FULL — пересоздаёт таблицу (блокирует! только в окно)
VACUUM FULL orders;
```

---

## Индексы

```sql
-- B-tree (default) — =, <, >, BETWEEN, IN, LIKE (без leading %)
CREATE INDEX idx_users_email ON users(email);

-- GIN — JSONB, массивы, полнотекстовый поиск
CREATE INDEX idx_users_data ON users USING GIN (metadata jsonb_path_ops);

-- BRIN — большие упорядоченные таблицы (логи, временные ряды)
CREATE INDEX idx_orders_created ON orders USING BRIN (created_at)
WITH (pages_per_range = 32);

-- Partial index — только для нужных строк
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';

-- Covering index (include) — для covering queries
CREATE INDEX idx_users_name ON users(name) INCLUDE (email, status);
```

### EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE email = 'alice@example.com';

-- Seq Scan — плохо (полное сканирование таблицы)
-- Index Scan — хорошо
-- Index Only Scan — отлично (данные в индексе)
-- Bitmap Heap Scan — нормально (для многих строк)

-- Cost: первые числа = startup cost, вторые = total cost
-- Rows: оценка строк
-- Execution Time: реальное время (мс)
```

---

## Транзакции и изоляция

```sql
-- Уровни изоляции (по умолчанию READ COMMITTED)
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- SERIALIZABLE — для финансовых операций (защита от фантомов)

-- Advisory locks — прикладные блокировки (не строки!)
SELECT pg_advisory_lock(42);
SELECT pg_advisory_unlock(42);

-- Блокировки
SELECT pid, wait_event, query, state
FROM pg_stat_activity
WHERE state = 'active' AND wait_event IS NOT NULL;
```

---

## Партиционирование

```sql
-- По диапазону (pg 10+)
CREATE TABLE orders (
    id BIGSERIAL,
    created_at TIMESTAMPTZ NOT NULL,
    data JSONB
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2024_01
    PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02
    PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
```

---

## CTE и оконные функции

```sql
-- CTE (читаемее подзапроса)
WITH active_users AS (
    SELECT id, name FROM users WHERE status = 'active'
),
recent_orders AS (
    SELECT user_id, SUM(total) AS spent
    FROM orders
    WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
    GROUP BY user_id
)
SELECT u.name, COALESCE(o.spent, 0) AS recent_spent
FROM active_users u
LEFT JOIN recent_orders o ON o.user_id = u.id;

-- Оконные функции
SELECT
    product_id,
    sale_date,
    amount,
    SUM(amount) OVER (PARTITION BY product_id ORDER BY sale_date) AS running_total,
    ROW_NUMBER() OVER (ORDER BY amount DESC) AS rank,
    LAG(amount) OVER (PARTITION BY product_id ORDER BY sale_date) AS prev_amount
FROM sales;
```

---

## pg_stat_* — диагностика

```sql
-- Самые медленные запросы
SELECT query, calls, mean_time, rows
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

-- Размер таблицы и индекса
SELECT relname, pg_size_pretty(pg_total_relation_size(relid))
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```

---

## Подводные камни

| ❌ Ошибка | ✅ Правильно |
|---|---|
| `SELECT *` на продакшене | Явно перечислить поля |
| Нет `ORDER BY` с `LIMIT` | Результат недетерминирован |
| `VACUUM FULL` в рабочее время | Полная блокировка |
| Нет `pg_hba.conf` для localhost | Не подключиться из приложения |
| Индекс на `boolean` | Бесполезно |

---

> **Технически:** MVCC — изоляция через версионирование строк.
> GIN — инвертированный индекс для JSONB/массивов. BRIN — блочный индекс
> для упорядоченных данных.