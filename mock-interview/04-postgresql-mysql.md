# Мок-интервью: Базы данных (20 вопросов) — расширенные ответы

---

## Q1. MVCC — что это?

```
✅ Ответ:
Multiversion Concurrency Control — механизм изоляции транзакций в PostgreSQL.
Каждая транзакция видит снэпшот данных на момент своего начала.
UPDATE не изменяет строку, а создаёт новую версию — старая становится
dead tuple. VACUUM чистит dead tuples. Читатели не блокируют писателей.
```

---

## Q2. Индексы — B-tree vs GIN vs BRIN?

```
✅ Ответ:
B-tree — универсальный (=, <, >, BETWEEN, LIKE без leading %, IN).
GIN — для JSONB, массивов, full-text (инвертированный индекс).
BRIN — для больших упорядоченных таблиц (логи, временные ряды).
Partial index — только для части строк (WHERE status = 'active').
Covering index — поля в INCLUDE, чтобы не ходить в таблицу.
```

---

## Q3. PostgreSQL vs MySQL — главные отличия?

```
✅ Ответ:
1. MVCC: PG — xmin/xmax + VACUUM, MySQL — undo log
2. JSON: PG — JSONB (бинарный + GIN), MySQL — TEXT
3. DDL: PG — в транзакции, MySQL — неявный COMMIT
4. GROUP BY: PG — строгий, MySQL — нестрогий
5. FULL OUTER JOIN: PG есть, MySQL нет (до 8.0)
```

---

## Q4. SQL Injection — как предотвратить?

```
✅ Ответ:
Никогда не конкатенировать SQL! Всегда параметризованные запросы:
cursor.execute("SELECT * FROM users WHERE id = %s", [user_id])
Для ORM (SQLAlchemy) — санирует автоматически.
Проверять, что нет f-строк в SQL.
```

---

## Q5. Транзакции — уровни изоляции?

```
✅ Ответ:
READ COMMITTED (default) — видим committed других транзакций.
REPEATABLE READ — снэпшот на момент первого запроса (PG).
SERIALIZABLE — полная изоляция (защита от фантомов).
Чем строже — тем больше блокировок и меньше производительность.
```

---

## Q6. VACUUM — зачем?

```
✅ Ответ:
MVCC оставляет dead tuples (мёртвые строки) при UPDATE/DELETE.
VACUUM удаляет их, освобождая место для reuse.
VACUUM FULL — перестраивает таблицу (блокировка!).
Autovacuum — включён по умолчанию.
Без VACUUM — bloat, падение производительности.
```