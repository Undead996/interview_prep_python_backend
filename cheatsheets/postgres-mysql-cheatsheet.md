# PostgreSQL + MySQL Cheatsheet

## EXPLAIN

```sql
EXPLAIN (ANALYZE, BUFFERS, COSTS, TIMING) SELECT ...;
-- Seq Scan → плохо (нет индекса или таблица маленькая)
-- Index Scan → хорошо
-- Index Only Scan → отлично (Heap Fetches: 0)
-- Bitmap Heap Scan → компромисс (много строк из индекса)
```

## Индексы

```sql
CREATE INDEX ON users(email);                                    -- B-tree
CREATE INDEX ON users USING GIN (data jsonb_path_ops);          -- JSONB
CREATE INDEX ON orders USING BRIN (created_at);                  -- логи
CREATE INDEX ON users(email) WHERE status = 'active';            -- partial
CREATE INDEX ON users(name) INCLUDE (email, created_at);        -- covering
```

## JOIN

```sql
LEFT JOIN ... WHERE ... IS NULL  -- без совпадений
NOT EXISTS (SELECT 1 FROM ...)   -- быстрее для анти-join
IN (SELECT ...)                  -- ⚠️ NULL в подзапросе → пусто!
```

## Оконные функции

```sql
ROW_NUMBER() OVER (PARTITION BY col ORDER BY col)     -- 1,2,3
RANK() OVER (ORDER BY col)                            -- 1,1,3
DENSE_RANK() OVER (ORDER BY col)                      -- 1,1,2
SUM(col) OVER (ORDER BY col)                          -- running total
LAG(col) OVER (ORDER BY col)                          -- предыдущее
```

## CTE

```sql
WITH cte AS (SELECT ...) SELECT * FROM cte;
WITH RECURSIVE tree AS (...) SELECT * FROM tree;      -- иерархии
```

## Различия PG vs MySQL

| PG | MySQL |
|----|-------|
| DDL в транзакции (ROLLBACK ALTER) | DDL = неявный COMMIT |
| JSONB + GIN | JSON (текст) |
| VACUUM (xmin/xmax) | undo log |
| GROUP BY — строгий | GROUP BY — нестрогий |
| FULL OUTER JOIN ✅ | FULL OUTER JOIN ❌ (до 8.0) |
| RETURNING ✅ | LAST_INSERT_ID() |
| ILIKE (регистронезависимый) | LIKE + collation |
| ON CONFLICT DO UPDATE | ON DUPLICATE KEY UPDATE |

## Диагностика PG

```sql
-- Активные запросы:
SELECT pid, state, wait_event, query FROM pg_stat_activity WHERE state != 'idle';

-- Размеры таблиц:
SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) FROM pg_stat_user_tables;

-- Dead tuples:
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum FROM pg_stat_user_tables;

-- Медленные запросы (через pg_stat_statements):
SELECT query, calls, mean_exec_time FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;
```

## Транзакции

```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT ... FOR UPDATE;  -- блокировка строки
SAVEPOINT sp1;
ROLLBACK TO sp1;
COMMIT;  -- или ROLLBACK
```