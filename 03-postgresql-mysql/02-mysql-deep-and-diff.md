# MySQL и отличия от PostgreSQL — полный разбор

> **Цель:** понимать архитектурные различия, а не синтаксис. Это **самый частый вопрос** на собеседовании.

---

## 1. MVCC — PostgreSQL vs MySQL (InnoDB)

| Характеристика | PostgreSQL | MySQL (InnoDB) |
|---------------|-----------|----------------|
| **Хранение версий** | xmin/xmax прямо в строке таблицы | Undo log (отдельный tablespace) |
| **Cleanup мёртвых версий** | VACUUM (периодический, autovacuum) | Автоматически по мере необходимости |
| **Dead tuples** | Видны в `pg_stat_user_tables` (n_dead_tup) | Прозрачны для пользователя |
| **Последствия** | Bloat если autovacuum не успевает | Undo log растёт при долгих транзакциях |
| **Настройка** | autovacuum_vacuum_scale_factor и пр. | innodb_undo_log_truncate=ON |
| **UNDO retention** | Нет (через VACUUM) | innodb_undo_log_truncate + innodb_purge_rseg_truncate_frequency |

**Практический вывод:**
- В PG нужно следить за autovacuum (метрики: n_dead_tup, last_autovacuum)
- В MySQL нужно следить за размером undo-сегментов и не держать долгие незакрытые транзакции

---

## 2. Полная сравнительная таблица (20+ пунктов)

| # | Характеристика | PostgreSQL | MySQL |
|---|---------------|-----------|-------|
| 1 | **MVCC** | xmin/xmax + VACUUM | Undo log (InnoDB) |
| 2 | **DDL в транзакции** | ✅ `ROLLBACK ALTER TABLE` | ❌ Неявный COMMIT |
| 3 | **JSON** | JSONB (бинарный, GIN-индексы) | JSON (текстовый, без индексов) |
| 4 | **GROUP BY** | Строгий (все поля должны быть в GROUP BY или агрегированы) | Нестрогий (`ANY_VALUE()`) |
| 5 | **FULL OUTER JOIN** | ✅ | ❌ (до 8.0; в 8.0+ через UNION) |
| 6 | **Оконные функции** | Полная поддержка (9.0+) | С 8.0, хуже производительность |
| 7 | **CTE** | ✅ + RECURSIVE + MATERIALIZED | ✅ + RECURSIVE (8.0+) |
| 8 | **Regex** | `~`, `~*`, `!~` операторы + `regexp_replace()` | `REGEXP`, `REGEXP_REPLACE` |
| 9 | **Типы данных** | `BOOLEAN`, `ARRAY`, `UUID`, `JSONB`, `INET`, `CIDR`, `HSTORE` | `TINYINT`, `ENUM`, `SET` |
| 10 | **Агрегаты** | `STRING_AGG`, `ARRAY_AGG`, `JSON_AGG` | `GROUP_CONCAT`, `JSON_ARRAYAGG` |
| 11 | **Индексы** | B-tree, GIN, BRIN, HASH, GiST, SP-GiST, partial, covering | B-tree, FULLTEXT, SPATIAL (Geographic) |
| 12 | **Partial indexes** | ✅ (`WHERE condition`) | ❌ |
| 13 | **Covering indexes** | ✅ (`INCLUDE`) | ✅ (вторичные индексы InnoDB включают PK) |
| 14 | **EXPLAIN** | `EXPLAIN (ANALYZE, BUFFERS)` — подробный | `EXPLAIN ANALYZE` (8.0.18+) + `FORMAT=JSON` |
| 15 | **Партиционирование** | RANGE / LIST / HASH + subpartitioning | RANGE / LIST / HASH / KEY |
| 16 | **Репликация** | Streaming (физическая) + Logical (с 10.0) | Binary log (statement/row/mixed) |
| 17 | **Движки хранения** | Только один (heap) | InnoDB, MyISAM, MEMORY, ARCHIVE |
| 18 | **Транзакции в DDL** | Полная поддержка | Только InnoDB (не DDL!) |
| 19 | **UPSERT** | `INSERT ... ON CONFLICT DO UPDATE` | `INSERT ... ON DUPLICATE KEY UPDATE` или `REPLACE INTO` |
| 20 | **GENERATED columns** | ✅ (STORED) | ✅ (VIRTUAL + STORED) |
| 21 | **CHECK constraints** | ✅ (полноценная поддержка) | ✅ (8.0.16+, ранее игнорировались!) |
| 22 | **RETURNING** | ✅ (`INSERT ... RETURNING *`) | ❌ (только через `LAST_INSERT_ID()`) |

---

## 3. Движки MySQL — InnoDB vs MyISAM

| Характеристика | InnoDB (default) | MyISAM |
|---------------|-----------------|--------|
| **Транзакции (ACID)** | ✅ | ❌ |
| **MVCC** | ✅ (undo log) | ❌ |
| **Внешние ключи** | ✅ | ❌ |
| **Блокировки** | Row-level | Table-level |
| **Скорость чтения** | Средняя | Высокая (нет MVCC-оверхеда) |
| **Скорость записи** | Высокая | Низкая (table lock) |
| **Восстановление после сбоя** | Автоматическое (redo log) | Ручное (REPAIR TABLE) |
| **FULLTEXT-индекс** | ✅ (5.6+) | ✅ |
| **Сжатие данных** | ✅ (Barracuda) | ✅ (myisampack) |
| **Когда использовать** | Всегда (для продакшена) | Только для read-only логов/архивов |

---

## 4. Когда мигрировать с MySQL на PostgreSQL

### Причины для миграции:
1. **JSONB** — нужны GIN-индексы по JSON-ключам
2. **DDL в транзакциях** — миграции должны откатываться атомарно
3. **Строгий SQL** — GROUP BY без сюрпризов
4. **Оконные функции** — сложная аналитика быстрее
5. **Расширения** — PostGIS, pg_cron, pg_partman, TimescaleDB
6. **Типы данных** — ARRAY, UUID, INET, HSTORE, пользовательские типы

### Когда НЕ мигрировать:
1. **Только чтение, простые запросы** — MySQL проще
2. **Устоявшийся MySQL-кластер** — стоимость миграции > выгоды
3. **ORM абстрагирует различия** — если SQL не пишете руками
4. **Нужна репликация master-master** — MySQL (Galera) проще, чем PG (BDR)

---

## 5. Синтаксические различия (шпаргалка)

```sql
-- Лимит:
-- PG:  LIMIT 10 OFFSET 20
-- MySQL: LIMIT 20, 10  (offset, limit!)

-- Тип даты:
-- PG:  SELECT NOW()::date;
-- MySQL: SELECT CAST(NOW() AS DATE);

-- Автоинкремент:
-- PG:  id SERIAL PRIMARY KEY  или  id INT GENERATED ALWAYS AS IDENTITY
-- MySQL: id INT AUTO_INCREMENT PRIMARY KEY

-- Конкатенация:
-- PG:  SELECT 'Hello' || ' ' || 'World';
-- MySQL: SELECT CONCAT('Hello', ' ', 'World');

-- Булевы значения:
-- PG:  SELECT TRUE, FALSE;  (настоящий boolean тип)
-- MySQL: SELECT TRUE, FALSE;  (TINYINT(1) под капотом)

-- ILIKE (регистронезависимый LIKE):
-- PG:  SELECT * FROM users WHERE name ILIKE 'alice%';
-- MySQL: SELECT * FROM users WHERE name LIKE 'alice%';  -- зависит от collation!

-- UPSERT:
-- PG:  INSERT INTO t (id, val) VALUES (1, 'a') ON CONFLICT (id) DO UPDATE SET val = 'a';
-- MySQL: INSERT INTO t (id, val) VALUES (1, 'a') ON DUPLICATE KEY UPDATE val = 'a';

-- Вернуть вставленное:
-- PG:  INSERT INTO users (name) VALUES ('Alice') RETURNING id, name;
-- MySQL: SELECT LAST_INSERT_ID();

-- Обновление с JOIN:
-- PG:
UPDATE users u SET name = 'New' FROM orders o
WHERE u.id = o.user_id AND o.total > 100;

-- MySQL:
UPDATE users u JOIN orders o ON u.id = o.user_id
SET u.name = 'New' WHERE o.total > 100;

-- Создание таблицы как SELECT:
-- PG:  CREATE TABLE archive AS SELECT * FROM orders WHERE created_at < '2023-01-01';
-- MySQL: CREATE TABLE archive SELECT * FROM orders WHERE created_at < '2023-01-01';
```

---

## 6. SQL Injection — как предотвратить

```python
# ❌ НИКОГДА — конкатенация:
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")

# ✅ Параметризованный запрос:
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))     # PG
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))     # MySQL

# В SQLAlchemy — автоматически:
stmt = select(User).where(User.id == user_id)  # безопасно!
db.execute(text("SELECT * FROM users WHERE id = :id"), {"id": user_id})  # безопасно!
```

---

> **На собесе:** «PostgreSQL vs MySQL — главные отличия?» —
> «Не синтаксис, а архитектура:
> 1) MVCC: PG — xmin/xmax + VACUUM (нужен мониторинг bloat), MySQL — undo log (прозрачно, но при долгих транзакциях растёт).
> 2) JSONB: бинарный с GIN-индексами в PG vs TEXT в MySQL — если работаете с JSON, PG радикально быстрее.
> 3) DDL в транзакциях: PG позволяет ROLLBACK ALTER TABLE, MySQL делает неявный COMMIT — миграции в PG безопаснее.
> 4) GROUP BY: PG строгий (учит писать корректный SQL), MySQL позволяет ошибки.
> 5) Расширения: PostGIS, TimescaleDB, pg_cron — экосистема PG богаче для аналитики и геоданных.»