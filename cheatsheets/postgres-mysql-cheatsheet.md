# Шпаргалка: PostgreSQL + MySQL

---

## EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ... ;
-- Seq Scan — плохо
-- Index Scan — хорошо
-- Index Only Scan — отлично
```

## Индексы

```sql
CREATE INDEX ON users(email);                    -- B-tree
CREATE INDEX ON users USING GIN (data jsonb_path_ops);
CREATE INDEX ON orders USING BRIN (created_at);
CREATE INDEX ON users(email) WHERE status = 'active';  -- partial
```

## JOIN

```sql
SELECT u.name, o.total
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.id IS NULL;
```

## Оконные функции

```sql
ROW_NUMBER() OVER (PARTITION BY col ORDER BY col)
SUM(col) OVER (ORDER BY col ROWS UNBOUNDED PRECEDING)
LAG(col) OVER (ORDER BY col)
```

## CTE

```sql
WITH cte AS (SELECT ...) SELECT * FROM cte;
```

## Различия PG vs MySQL

| PG | MySQL |
|---|---|
| DDL в транзакции | DDL = неявный COMMIT |
| JSONB + GIN | JSON (текст) |
| VACUUM (MVCC) | undo log |
| GROUP BY — строгий | GROUP BY — нестрогий |
| FULL OUTER JOIN = ✅ | FULL OUTER JOIN = ❌