# PostgreSQL — глубокое погружение (MVCC, индексы, EXPLAIN, CTE)

> **Цель:** понять, как PostgreSQL работает под капотом, чтобы читать EXPLAIN ANALYZE и выбирать правильные индексы.

---

## 1. MVCC — как PostgreSQL хранит версии строк

### Что такое MVCC?

**Простыми словами:** когда транзакция A читает таблицу, а транзакция B одновременно что-то в ней меняет — A видит данные такими, какими они были **до** изменений B. PostgreSQL не блокирует читателей для писателей (почти никогда).

**Технически:** каждая строка имеет скрытые системные поля:

```sql
-- xmin  — ID транзакции, создавшей эту версию строки
-- xmax  — ID транзакции, удалившей эту версию (0 = живая строка)
-- ctid  — физическое расположение (page, offset)
-- cmin/cmax — command ID внутри транзакции
```

### Как работает UPDATE

```
1. НЕ модифицирует существующую строку
2. Помечает старую строку: xmax = ID текущей транзакции (она "мёртвая")
3. Вставляет НОВУЮ строку с новыми данными: xmin = ID текущей транзакции
4. После COMMIT: старая версия = dead tuple
5. VACUUM позже удаляет dead tuples
```

### Как работает SELECT

```sql
-- Для каждой строки-кандидата PostgreSQL проверяет:
--   1. xmin должен быть committed (или это моя транзакция)
--   2. xmax должен быть 0 ИЛИ xmax-транзакция ещё не committed
--   3. Если xmax committed → строка удалена, skip

-- Упрощённо:
--   (xmin committed AND (xmax = 0 OR xmax not committed))
```

### Почему VACUUM нужен

После 1000 UPDATE одной строки у вас 1000 dead tuples и только 1 живая. Таблица раздувается («bloat»), запросы сканируют мёртвые строки, производительность падает.

| Операция | Блокирует таблицу? | Что делает |
|----------|-------------------|------------|
| **VACUUM** | ❌ Нет (non-blocking) | Помечает dead tuples как свободное место для reuse |
| **VACUUM FULL** | ✅ Да (ExclusiveLock) | Пересоздаёт таблицу, возвращает место ОС |
| **Autovacuum** | Автоматически | Фоновый VACUUM + ANALYZE |

```sql
-- Диагностика:
SELECT schemaname, relname, n_live_tup, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;

-- Принудительный VACUUM:
VACUUM ANALYZE users;
-- VACUUM FULL (осторожно — блокировка!):
VACUUM FULL users;
```

---

## 2. Индексы — полный разбор

### B-tree (по умолчанию)

```sql
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_name_email ON users(name, email);  -- составной
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);
```

**Покрывает:** `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `IN`, `IS NULL`, `LIKE 'abc%'`

**Не покрывает:** `LIKE '%abc'`, `~` (regex), `->>` (JSONB-оператор)

### GIN — Generalized Inverted Index

```sql
-- JSONB:
CREATE INDEX idx_products_data ON products USING GIN (data jsonb_path_ops);

-- Полнотекстовый поиск:
CREATE INDEX idx_articles_body ON articles USING GIN (to_tsvector('english', body));

-- Массивы:
CREATE INDEX idx_users_tags ON users USING GIN (tags);
```

**Когда:** JSONB, полнотекстовый поиск, массивы, `@>` (contains), `?` (exists key)

### BRIN — Block Range Index

```sql
CREATE INDEX idx_orders_created ON orders USING BRIN (created_at)
    WITH (pages_per_range = 32);
```

**Для чего:** очень большие таблицы с коррелированными данными (логи, временные ряды). В 100–1000 раз меньше B-tree по размеру. Но менее точный — PostgreSQL всё равно придётся проверить строки в блоке.

### HASH — только для `=`

```sql
CREATE INDEX idx_users_email_hash ON users USING HASH (email);
```
Редко используется. B-tree покрывает `=` тоже. HASH — только если ключ очень длинный и не помещается в B-tree.

### Partial index

```sql
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;
CREATE INDEX idx_pending_orders ON orders(created_at) WHERE status = 'pending';
```

Меньше размер, быстрее вставка (индекс обновляется только для matching строк).

### Covering index (INCLUDE)

```sql
CREATE INDEX idx_users_name_cover ON users(name) INCLUDE (email, created_at);
```

Данные из INCLUDE хранятся прямо в индексе → Index Only Scan возможен без чтения таблицы.

### Сводная таблица

| Тип | Размер | Скорость поиска | Скорость вставки | Когда |
|-----|--------|----------------|-----------------|-------|
| B-tree | Средний | Быстрый | Быстрый | 95% случаев |
| GIN | Большой | Быстрый для JSONB/FT | Медленный | JSONB, массив, full-text |
| BRIN | Очень маленький | Медленнее | Быстрый | Логи, временные ряды |
| HASH | Маленький | Быстрый для `=` | Быстрый | Длинные ключи |

---

## 3. EXPLAIN ANALYZE — как читать

```sql
EXPLAIN (ANALYZE, BUFFERS, COSTS, TIMING) SELECT * FROM users WHERE email = 'alice@test.com';
```

### Пример вывода и расшифровка

```
Index Scan using idx_users_email on users  (cost=0.29..8.31 rows=1 width=120)
             (actual time=0.015..0.016 rows=1 loops=1)
  Index Cond: (email = 'alice@test.com'::text)
  Buffers: shared hit=3
Planning Time: 0.082 ms
Execution Time: 0.032 ms
```

| Строка | Значение |
|--------|---------|
| `Index Scan using idx_users_email` | Используется индекс (хорошо) |
| `cost=0.29..8.31` | Start-up cost .. Total cost (абстрактные единицы) |
| `actual time=0.015..0.016` | Реальное время: первый ряд .. все ряды (ms) |
| `rows=1` | Оценено: 1 строка (если сильно расходится с actual — устаревшая статистика) |
| `Buffers: shared hit=3` | 3 страницы прочитано из кэша (hit) или с диска (read) |
| `loops=1` | Сколько раз узел плана был выполнен (для вложенных циклов > 1) |

### Типы сканирования (от худшего к лучшему)

| Тип | Когда | Что значит |
|-----|-------|-----------|
| **Seq Scan** | Нет индекса / таблица маленькая | Чтение всей таблицы |
| **Bitmap Heap Scan** | Индекс находит много строк | Индекс + Heap в порядке физического расположения |
| **Index Scan** | Индекс + чтение строки | Хорошо: индекс → таблица |
| **Index Only Scan** | Все поля в индексе (covering) | Идеально: таблица не читается! |

```sql
-- Seq Scan → Index Scan:
EXPLAIN ANALYZE SELECT * FROM orders WHERE id = 42;
-- Без индекса: Seq Scan (плохо для больших таблиц)
-- С индексом: Index Scan (быстро)

-- Убедиться, что Index Only Scan работает:
EXPLAIN (ANALYZE, BUFFERS) SELECT email FROM users WHERE email LIKE 'alice%';
-- "Index Only Scan using idx_users_email on users"
-- "Heap Fetches: 0"  ← таблица не читалась!
```

---

## 4. CTE (Common Table Expressions)

```sql
-- Обычный CTE (читаемее подзапроса):
WITH recent_orders AS (
    SELECT user_id, COUNT(*) AS order_count
    FROM orders
    WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
    GROUP BY user_id
)
SELECT u.name, r.order_count
FROM users u
JOIN recent_orders r ON r.user_id = u.id
WHERE r.order_count >= 3
ORDER BY r.order_count DESC;

-- Рекурсивный CTE (деревья, иерархии):
WITH RECURSIVE subordinates AS (
    -- Базовый случай
    SELECT id, name, manager_id, 1 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Рекурсивный случай
    SELECT e.id, e.name, e.manager_id, s.level + 1
    FROM employees e
    JOIN subordinates s ON e.manager_id = s.id
)
SELECT * FROM subordinates ORDER BY level, name;

-- MATERIALIZED (по умолчанию в PG < 12) vs NOT MATERIALIZED (PG 12+):
WITH cte AS MATERIALIZED (SELECT ...)  -- вычислить 1 раз, использовать как таблицу
WITH cte AS NOT MATERIALIZED (...)     -- встроить как подзапрос (может быть быстрее)
```

---

## 5. Оконные функции — полный набор

```sql
-- ROW_NUMBER() — уникальный номер внутри группы:
SELECT name, department, salary,
       ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rank
FROM employee;

-- RANK() — с пропусками (одинаковые значения → одинаковый rank):
SELECT name, salary,
       RANK() OVER (ORDER BY salary DESC) AS rank
FROM employee;

-- DENSE_RANK() — без пропусков:
SELECT name, salary,
       DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank
FROM employee;

-- SUM() OVER — накопительная сумма:
SELECT date, amount,
       SUM(amount) OVER (ORDER BY date) AS running_total
FROM daily_sales;

-- LAG/LEAD — предыдущее/следующее значение:
SELECT date, amount,
       LAG(amount) OVER (ORDER BY date) AS prev_amount,
       amount - LAG(amount) OVER (ORDER BY date) AS diff
FROM daily_sales;

-- PARTITION BY + ORDER BY:
SELECT department, name, salary,
       AVG(salary) OVER (PARTITION BY department) AS dept_avg,
       salary - AVG(salary) OVER (PARTITION BY department) AS diff_from_avg
FROM employee;
```

---

## 6. Уровни изоляции

| Уровень | Dirty Read | Non-repeatable Read | Phantom Read | Производительность |
|---------|-----------|--------------------|-------------|-------------------|
| **READ UNCOMMITTED** | ✅ возможен | ✅ | ✅ | Максимальная |
| **READ COMMITTED** (default) | ❌ | ✅ возможен | ✅ | Хорошая |
| **REPEATABLE READ** | ❌ | ❌ | ✅ в PG — ❌* | Хорошая |
| **SERIALIZABLE** | ❌ | ❌ | ❌ | Низкая |

*В PostgreSQL REPEATABLE READ защищает от фантомов благодаря snapshot isolation.

```sql
-- Установить уровень:
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM users WHERE id = 1;  -- снэпшот на момент начала транзакции
-- ...
COMMIT;

-- Проверить текущий уровень:
SHOW default_transaction_isolation;
```

---

## 7. Партиционирование

```sql
-- Range partitioning:
CREATE TABLE orders (
    id BIGINT,
    created_at TIMESTAMPTZ NOT NULL,
    total NUMERIC
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2024_q1 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
CREATE TABLE orders_2024_q2 PARTITION OF orders
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

-- List partitioning:
CREATE TABLE users PARTITION BY LIST (country);
CREATE TABLE users_ru PARTITION OF users FOR VALUES IN ('RU', 'BY', 'KZ');
CREATE TABLE users_eu PARTITION OF users FOR VALUES IN ('DE', 'FR', 'IT');

-- Hash partitioning:
CREATE TABLE events PARTITION BY HASH (user_id);
CREATE TABLE events_0 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE events_1 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 1);
```

---

## 8. Диагностика

```sql
-- Активные запросы:
SELECT pid, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE state != 'idle' AND pid != pg_backend_pid();

-- Блокировки:
SELECT blocked.pid AS blocked_pid,
       blocked.query AS blocked_query,
       blocking.pid AS blocking_pid,
       blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks bl ON bl.pid = blocked.pid
JOIN pg_locks bkl ON bkl.locktype = bl.locktype
    AND bkl.database IS NOT DISTINCT FROM bl.database
    AND bkl.relation IS NOT DISTINCT FROM bl.relation
    AND bkl.page IS NOT DISTINCT FROM bl.page
    AND bkl.tuple IS NOT DISTINCT FROM bl.tuple
    AND bkl.virtualxid IS NOT DISTINCT FROM bl.virtualxid
    AND bkl.transactionid IS NOT DISTINCT FROM bl.transactionid
    AND bkl.classid IS NOT DISTINCT FROM bl.classid
    AND bkl.objid IS NOT DISTINCT FROM bl.objid
    AND bkl.objsubid IS NOT DISTINCT FROM bl.objsubid
    AND bkl.pid != bl.pid
JOIN pg_stat_activity blocking ON bkl.pid = blocking.pid
WHERE NOT bl.granted;

-- Размеры:
SELECT relname,
       pg_size_pretty(pg_total_relation_size(relid)) AS total,
       pg_size_pretty(pg_relation_size(relid)) AS table,
       pg_size_pretty(pg_indexes_size(relid)) AS indexes
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 10;

-- Индексы (неиспользуемые):
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

---

> **На собесе:** «Что такое MVCC?» —
> «Механизм изоляции через версионирование строк. Каждая строка имеет xmin (кто создал) и xmax (кто удалил). UPDATE создаёт новую версию, старая становится dead tuple. VACUUM чистит dead tuples. Читатели не блокируют писателей. Это основа ACID в PostgreSQL — snapshot isolation без блокировок чтения.»