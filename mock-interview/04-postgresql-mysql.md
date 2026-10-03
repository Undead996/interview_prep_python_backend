# Мок-интервью: Базы данных (20 вопросов) — расширенные ответы

---

## Q1. MVCC — что это?

Multiversion Concurrency Control. Каждая транзакция видит снэпшот данных на момент начала. UPDATE не изменяет строку, а создаёт новую версию (xmin/xmax в PG, undo log в MySQL). VACUUM чистит dead tuples в PG. Читатели не блокируют писателей.

---

## Q2. Индексы — B-tree vs GIN vs BRIN?

**B-tree:** `=`, `<`, `>`, `BETWEEN`, `LIKE 'abc%'`. 95% случаев.
**GIN:** JSONB, массивы, full-text search. Инвертированный индекс.
**BRIN:** большие упорядоченные таблицы (логи). 100-1000x меньше B-tree, но менее точный.
**Partial:** `WHERE status='active'` — только часть строк.
**Covering:** `INCLUDE(email)` — данные прямо в индексе (Index Only Scan).

---

## Q3. PostgreSQL vs MySQL — главные отличия?

1. **MVCC:** PG — xmin/xmax + VACUUM, MySQL — undo log
2. **JSON:** PG — JSONB (бинарный + GIN), MySQL — TEXT
3. **DDL:** PG — в транзакции (ROLLBACK ALTER TABLE), MySQL — неявный COMMIT
4. **GROUP BY:** PG строгий, MySQL нестрогий
5. **FULL OUTER JOIN:** PG ✅, MySQL ❌ (до 8.0)
6. **Расширения:** PostGIS, TimescaleDB, pg_cron vs ограниченная экосистема MySQL

---

## Q4. SQL Injection — как предотвратить?

**Никогда не конкатенировать SQL.** Параметризованные запросы: `cursor.execute("... WHERE id = %s", (id,))`. ORM санирует автоматически — `select(User).where(User.id == id)`. Не использовать f-строки в SQL.

---

## Q5. Уровни изоляции транзакций?

**READ UNCOMMITTED** — грязное чтение. **READ COMMITTED** (default) — только committed. **REPEATABLE READ** — снэпшот на момент первого запроса (PG: snapshot isolation, защита от фантомов). **SERIALIZABLE** — полная изоляция. Чем строже — тем больше блокировок.

---

## Q6. VACUUM — зачем?

MVCC оставляет dead tuples. VACUUM помечает их как свободное место. Без VACUUM — bloat (таблица раздувается), запросы сканируют мёртвые строки. VACUUM FULL — перестраивает таблицу (блокировка!). Autovacuum — включён по умолчанию.

---

## Q7. Почему Index Only Scan не всегда работает?

Даже если все поля в индексе (covering), PostgreSQL должен проверить visibility (xmin/xmax) каждой строки в таблице. Без недавнего VACUUM — Heap Fetches > 0, и Scan не является чисто Index Only.

---

## Q8. EXPLAIN — как читать cost?

`(cost=0.29..8.31 rows=1)` — 0.29 — startup cost (до первой строки), 8.31 — total cost (все строки). Абстрактные единицы. `actual time` — реальное в ms. Расхождение `rows` между оценкой и actual → устаревшая статистика → ANALYZE.

---

## Q9. JOIN vs EXISTS vs IN?

**EXISTS** — быстрее для проверки наличия (останавливается на первом совпадении). **IN** — для маленьких подзапросов. **JOIN** — когда нужны данные из связанной таблицы. IN с NULL в подзапросе → пустой результат (опасно!).

---

## Q10. Что такое CTE?

Common Table Expression — временный именованный результат. `WITH name AS (SELECT ...)`. С PostgreSQL 12 — `MATERIALIZED` (вычислить 1 раз) или `NOT MATERIALIZED` (встроить как подзапрос). Рекурсивные CTE — для деревьев/иерархий.

---

## Q11. Как работает `INSERT ... ON CONFLICT`?

UPSERT в PostgreSQL. `ON CONFLICT (unique_column) DO UPDATE SET ...` При конфликте уникального ключа — обновляет существующую запись. В MySQL — `ON DUPLICATE KEY UPDATE`.

---

## Q12. Что такое `RETURNING`?

Возвращает вставленные/обновлённые строки: `INSERT INTO users (...) VALUES (...) RETURNING id, name`. В MySQL — только `LAST_INSERT_ID()`. Очень удобно для получения сгенерированных ID без отдельного SELECT.

---

## Q13. Партиционирование — когда нужно?

Когда таблица > 10-100 GB. Разделяет на физически отдельные куски. Типы: RANGE (по дате), LIST (по стране), HASH (равномерно). Ускоряет запросы (partition pruning) и упрощает удаление старых данных (DROP PARTITION).

---

## Q14. `ROW_NUMBER` vs `RANK` vs `DENSE_RANK`?

`ROW_NUMBER`: 1,2,3 (всегда уникален), `RANK`: 1,1,3 (пропускает при равенстве), `DENSE_RANK`: 1,1,2 (не пропускает).

---

## Q15. Что такое `FOR UPDATE`?

Блокировка строки при SELECT: `SELECT * FROM users WHERE id=1 FOR UPDATE`. Другие транзакции не могут изменить/заблокировать эту строку до COMMIT/ROLLBACK. Для пессимистичной блокировки: «забронировать, потом изменить».

---

## Q16. Репликация в PostgreSQL?

**Streaming (физическая):** WAL (Write-Ahead Log) непрерывно передаётся на реплики. Быстро, но реплика — точная копия (та же версия PG). **Logical (с PG 10):** изменения на уровне строк (INSERT/UPDATE/DELETE). Можно реплицировать отдельные таблицы, между версиями PG.

---

## Q17. `pg_stat_activity` — что показывает?

Текущие соединения и запросы: pid, state (active/idle/idle in transaction), query, wait_event. Используется для поиска долгих запросов, блокировок, висящих транзакций.

---

## Q18. Как найти медленные запросы?

`pg_stat_statements` (расширение): агрегация по запросам — total_time, calls, mean_time. `log_min_duration_statement = 1000` — логировать запросы дольше 1s. `auto_explain` — автоматический EXPLAIN для медленных запросов.

---

## Q19. Deadlock — как возникает и что делать?

Транзакция A заблокировала строку 1, ждёт строку 2. Транзакция B заблокировала строку 2, ждёт строку 1. PostgreSQL автоматически определяет deadlock и отменяет одну из транзакций. **Профилактика:** всегда блокировать в одном порядке, короткие транзакции.

---

## Q20. MySQL InnoDB vs MyISAM?

**InnoDB:** транзакции (ACID), row-level locking, MVCC, внешние ключи. Используется всегда. **MyISAM:** нет транзакций, table-level locking, быстрее чтение. Только для read-only логов/архивов.