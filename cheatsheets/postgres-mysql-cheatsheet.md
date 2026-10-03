# Шпаргалка: PostgreSQL + MySQL — последний день

---

## EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ... ;
-- Seq Scan — плохо (полное сканирование)
-- Index Scan — хорошо (индекс + чтение таблицы)
-- Index Only Scan — отлично (только индекс)
```

## Индексы

```sql
CREATE INDEX ON users(email);                    -- B-tree (=, <, >, BETWEEN, IN)
CREATE INDEX ON users USING GIN (data jsonb_path_ops);  -- JSONB
CREATE INDEX ON orders USING BRIN (created_at);  -- упорядоченные большие таблицы
CREATE INDEX ON users(email) WHERE status = 'active';  -- partial
CREATE INDEX ON users(name) INCLUDE (email);     -- covering (данные в индексе)
```

## JOIN

```sql
SELECT u.name, o.total
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.id IS NULL;  -- пользователи без заказов
```

## Оконные функции

```sql
ROW_NUMBER() OVER (PARTITION BY col ORDER BY col)     -- ранг
SUM(col) OVER (ORDER BY col ROWS UNBOUNDED PRECEDING) -- running total
LAG(col) OVER (ORDER BY col)                           -- предыдущее значение
RANK() OVER (ORDER BY col)                            -- ранг с пропусками
```

## CTE

```sql
WITH cte AS (SELECT ...) SELECT * FROM cte;
-- RECURSIVE для деревьев
```

## Различия PG vs MySQL

| PG | MySQL |
|----|-------|
| DDL в транзакции | DDL = неявный COMMIT |
| JSONB + GIN | JSON (текст) |
| VACUUM (MVCC) | undo log |
| GROUP BY — строгий | GROUP BY — нестрогий |
| FULL OUTER JOIN = ✅ | FULL OUTER JOIN = ❌ (до 8.0) |