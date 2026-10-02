# MySQL и отличия от PostgreSQL

---

## Сравнение

| Параметр | PostgreSQL | MySQL |
|---|---|---|
| ACID | Полная | Зависит от движка (InnoDB — ACID, MyISAM — нет) |
| MVCC | Да (всегда) | Да (InnoDB) |
| JSON | JSONB (бинарный, с индексами) | JSON (текстовый, без GIN) |
| Индексы | B-tree, GIN, BRIN, GiST, SP-GiST | B-tree, Hash, Full-text, Spatial |
| CTE / RECURSIVE | ✅ (WITH RECURSIVE) | ✅ (8.0+) |
| Оконные функции | ✅ (2009+) | ✅ (8.0+) |
| Партиционирование | Декларативное (10+) | Раньше, но ограничения |
| Тип ENUM | CREATE TYPE | ENUM — нативный тип |
| FULL OUTER JOIN | ✅ | ❌ (до 8.0 — UNION) |
| Асинхронность | asyncpg | Нет нативного async |
| Репликация | Streaming, Logical | Master-slave, Group Replication |
| VACUUM | Нужен (MVCC) | Не нужен (InnoDB — undo log) |

---

## Когда мигрируют с MySQL на PG

```
MySQL (MyISAM) → PostgreSQL:
  ❌ Нет транзакций
  ❌ Нет FOREIGN KEY
  ❌ Нет оконных функций / CTE
  ❌ Блокировка таблицы при ALTER

MySQL (InnoDB) → PostgreSQL:
  ✅ Транзакции есть
  ✅ MVCC есть
  ⚠️ Различия в синтаксисе и поведении
```

---

## Различия в синтаксисе

```sql
-- PostgreSQL
SELECT NOW();
SELECT CURRENT_DATE;
SELECT CURRENT_TIMESTAMP;
SELECT ARRAY[1,2,3];
SELECT 'text' LIKE '%test%';

-- MySQL
SELECT NOW();
SELECT CURDATE();
SELECT CURRENT_TIMESTAMP();
SELECT JSON_ARRAY(1,2,3);
SELECT 'text' LIKE '%test%';
SELECT 1 + 1;  -- без FROM (в PG — SELECT 1+1 тоже работает)
```

### Различия в транзакциях и блокировках

```sql
-- PostgreSQL: DDL в транзакции
BEGIN;
  ALTER TABLE users ADD COLUMN phone TEXT;
  UPDATE users SET phone = '000' WHERE phone IS NULL;
COMMIT;  -- ✅

-- MySQL: DDL — неявная фиксация
-- ALTER TABLE users ADD COLUMN phone TEXT;  -- неявный COMMIT!
```

---

## Движки MySQL

| Движок | Транзакции | Foreign Keys | Full-text |
|---|---|---|---|
| **InnoDB** (default) | ✅ | ✅ | ✅ |
| **MyISAM** | ❌ | ❌ | ✅ |
| **Memory** | ❌ | ❌ | ❌ |
| **CSV** | ❌ | ❌ | ❌ |

---

## mysql vs postgres: index

```sql
-- PostgreSQL — много типов
CREATE INDEX ON users USING GIN (data jsonb_path_ops);
CREATE INDEX ON orders USING BRIN (created_at);
CREATE INDEX ON users USING GiST (geo_data);

-- MySQL — в основном B-tree (+ FULLTEXT, SPATIAL)
CREATE INDEX idx_users_email ON users(email);
CREATE FULLTEXT INDEX idx_articles_body ON articles(body);
```

---

## Подводные камни

| ❌ Ошибка | ✅ Правильно |
|---|---|
| MySQL по умолчанию — `READ COMMITTED`? | Repeatable Read (InnoDB) |
| `GROUP BY` без строгих правил | MySQL допускает столбцы вне GROUP BY — PG — нет |
| `HAVING` может использовать алиасы | PG — нет, MySQL — да |
| `LIMIT` без ORDER BY | Разный порядок в PG и MySQL — недетерминирован |
| MyISAM — «всё хорошо» | Нет транзакций! Никогда не использовать |

---

> **На собесе:** «Чем PostgreSQL отличается от MySQL?» —
> «1) PostgreSQL — полный ACID, MySQL — только InnoDB.
> 2) PostgreSQL — JSONB с GIN-индексами, MySQL — JSON только для хранения.
> 3) PostgreSQL — оконные функции и CTE с RECURSIVE исторически раньше.
> 4) MVCC в PG — через версионирование (xmin/xmax, VACUUM),
> в MySQL — через undo log.
> 5) DDL в PG — в транзакции, в MySQL — неявный COMMIT.»