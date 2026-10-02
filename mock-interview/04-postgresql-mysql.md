# Мок-интервью: Базы данных (20 вопросов)

---

## Q1. MVCC — что это?

```
✅ Ответ: «Multiversion Concurrency Control — каждая транзакция видит
snapshot базы на момент своего начала. Читатели не блокируют писателей.
Старые версии строк — dead tuples — чистит VACUUM (в PG).»
```

## Q2. Индексы — B-tree vs GIN vs BRIN?

```
✅ Ответ: «B-tree — =, <, >, BETWEEN, LIKE (без leading %). Универсальный.
GIN — JSONB, массивы, full-text. Составной индекс: (a, b) покрывает
WHERE a=... и WHERE a=... AND b=..., но НЕ WHERE b=... (leading column).»
```

## Q3. PostgreSQL vs MySQL — главные отличия?

```
✅ Ответ: «1) MVCC в PG — xmin/xmax + VACUUM, в MySQL — undo log.
2) JSON в PG — JSONB (бинарный + GIN), в MySQL — только хранение.
3) DDL в PG — в транзакции, в MySQL — неявный COMMIT.
4) PG — строже: GROUP BY без агрегации всех полей — ошибка.»
```

## Q4. SQL Injection — как предотвратить?

```
✅ Ответ: «Никогда не конкатенировать SQL. Всегда параметризованные запросы:
cursor.execute("SELECT * FROM users WHERE id = %s", [user_id]).
Для ORM (SQLAlchemy) — санирует. Для тестов — проверить, что нет
f-строк в SQL.»
```

## Q5. Транзакции — уровни изоляции?

```
✅ Ответ: «READ COMMITTED (default) — видим committed данные других
транзакций. REPEATABLE READ — snapshot на момент первого запроса.
SERIALIZABLE — полная изоляция (защита от фантомов). Чем строже —
тем больше блокировок.»
```

## Q6. VACUUM — зачем?

```
✅ Ответ: «MVCC оставляет мёртвые строки при UPDATE/DELETE. VACUUM
удаляет их, освобождая место для reuse. VACUUM FULL — перестраивает
таблицу (блокировка!). Autovacuum — включён по умолчанию. Если не
успевает — bloat, падение производительности. PostgreSQL.»
```