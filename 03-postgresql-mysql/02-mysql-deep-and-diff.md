# MySQL и отличия от PostgreSQL — расширенный разбор

> **Цель:** понять принципиальные различия на уровне архитектуры, а не синтаксиса.
> Это **самый частый вопрос** на собесе.

---

## 1. MVCC — PG vs MySQL

| | PostgreSQL | MySQL (InnoDB) |
|--|-----------|----------------|
| **Как хранит версии** | xmin/xmax в строке | Undo log (отдельный сегмент) |
| **Cleanup** | VACUUM (периодически) | Автоматически (undo log очищается) |
| **Dead tuples** | Явно видно в pg_stat_* | Прозрачно |

**Практическое значение:**
- В PG нужно следить за autovacuum. Если он не успевает — bloat, падение производительности.
- В MySQL это прозрачно, но undo log может расти неконтролируемо при долгих транзакциях.

---

## 2. JSON — PG vs MySQL

| | PostgreSQL | MySQL |
|--|-----------|-------|
| **Тип** | JSONB (бинарный, с индексами GIN) | JSON (текстовый, TEXT под капотом) |
| **Индексы** | ✅ GIN (jsonb_path_ops) | ❌ Нет индексов по ключам |
| **Скорость** | Быстрее (бинарный) | Медленнее (парсинг текста) |
| **Операции** | `->>`, `@>`, `?` | `->>`, `->`, `JSON_EXTRACT` |

**Вывод:** если вы работаете с JSON — PostgreSQL намного удобнее и быстрее.

---

## 3. DDL в транзакции

```sql
-- PostgreSQL: можно ROLLBACK ALTER TABLE!
BEGIN;
  ALTER TABLE users ADD COLUMN phone TEXT;
  UPDATE users SET phone = '000' WHERE phone IS NULL;
ROLLBACK;  -- всё отменено, таблица без изменений

-- MySQL: ALTER = неявный COMMIT!
ALTER TABLE users ADD COLUMN phone TEXT;  -- ← неявный COMMIT!
UPDATE users SET phone = '000' WHERE phone IS NULL;  -- другой транзакции
```

---

## 4. GROUP BY — строгость

```sql
-- PostgreSQL: ❌ Ошибка — поле вне GROUP BY не агрегировано
SELECT name, department_id, AVG(salary)
FROM employee
GROUP BY department_id;
-- "column employee.name must appear in GROUP BY"

-- MySQL: ✅ Работает (возвращает первый name в группе)
```

**Последствия:** на PG нельзя "срезать углы" — SQL должен быть строгим. Код, написанный под MySQL, часто не работает на PG без доработок.

---

> **На собесе:** «PostgreSQL vs MySQL — главные отличия?» —  
> «1) MVCC: в PG — xmin/xmax + VACUUM, в MySQL — undo log (прозрачно).  
> 2) JSONB: в PG — бинарный с GIN-индексами, в MySQL — TEXT.  
> 3) DDL: в PG — в транзакции, в MySQL — неявный COMMIT.  
> 4) GROUP BY: PG — строгий, MySQL — нестрогий.  
> 5) FULL OUTER JOIN: в PG есть, в MySQL нет (до 8.0).»