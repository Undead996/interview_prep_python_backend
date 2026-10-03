# PostgreSQL — глубокое погружение (расширенно)

> **Цель:** понять, как работает MVCC под капотом, когда использовать каждый тип
> индекса, как читать EXPLAIN ANALYZE.

---

## 1. MVCC — как это внутри

Каждая строка в PostgreSQL имеет скрытые поля: `xmin` и `xmax`.

```sql
-- xmin: ID транзакции, которая создала эту версию строки
-- xmax: ID транзакции, которая удалила эту версию строки (0 = не удалена)
```

**Как работает UPDATE:**
1. Сервер **не модифицирует** существующую строку — он помечает её как удалённую (xmax = текущая транзакция)
2. Создаётся **новая** строка с новым значением (xmin = текущая транзакция)
3. После COMMIT старая версия становится "dead tuple"
4. VACUUM удаляет dead tuples

**Как работает SELECT:**
1. Для каждой строки проверяем: видна ли она текущей транзакции?
2. Если xmin committed и (xmax = 0 или xmax не committed) → строка видна
3. Если xmax committed → строка удалена (skip)

### Почему VACUUM нужен

После 1000 UPDATE одной строки у вас 1000 dead tuples и 1 живая. Таблица раздувается. VACUUM помечает dead tuples как "можно переиспользовать место".

**VACUUM vs VACUUM FULL:**
| Операция | Блокирует? | Что делает |
|----------|-----------|------------|
| VACUUM | ❌ (не блокирует запись) | Помечает место как свободное для reuse |
| VACUUM FULL | ✅ (блокирует таблицу) | Пересоздаёт таблицу, возвращает место ОС |

---

## 2. Индексы — когда какой

### B-tree (по умолчанию)
```sql
CREATE INDEX ON users(email);
```
**Покрывает:** `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `IN`, `LIKE 'abc%'` (без leading %), `IS NULL`

**Не покроет:** `LIKE '%abc'`, `jsonb->>'field'`, `array_contains`

### GIN (Generalized Inverted Index)
```sql
CREATE INDEX ON users USING GIN (metadata jsonb_path_ops);
CREATE INDEX ON articles USING GIN (to_tsvector('english', body));
```
**Для:** JSONB, массивы, полнотекстовый поиск. Инвертированный индекс — каждое значение ссылается на строку, а не строка на значение.

### BRIN (Block Range Index)
```sql
CREATE INDEX ON orders USING BRIN (created_at) WITH (pages_per_range = 32);
```
**Для:** очень большие, упорядоченные таблицы (логи, временные ряды). Хранит мета-информацию по блокам, а не по строкам. В 1000 раз меньше B-tree, но менее точный.

### Partial Index
```sql
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';
```
Индекс содержит только строки с `status = 'active'`. Меньше размер, быстрее вставка.

### Covering Index (INCLUDE)
```sql
CREATE INDEX ON users(name) INCLUDE (email, status);
```
Данные email и status хранятся прямо в индексе — не нужно идти в таблицу.

---

## 3. EXPLAIN ANALYZE — как читать

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE email = 'alice@example.com';
```

**Типы сканирования (от худшего к лучшему):**

| Тип | Ситуация | Что значит |
|-----|----------|-----------|
| **Seq Scan** | Нет подходящего индекса | Чтение всей таблицы — плохо для больших таблиц |
| **Bitmap Heap Scan** | Индекс находит много строк | Комбинация индекса + последовательного чтения |
| **Index Scan** | Индекс + чтение строки из таблицы | Хорошо — индекс + 1 round-trip к таблице |
| **Index Only Scan** | Все нужные поля в индексе | Отлично — таблица не читается вообще |

```sql
-- Пример: Index Only Scan
EXPLAIN (ANALYZE, BUFFERS) SELECT email FROM users WHERE email LIKE 'alice%';
-- "Index Only Scan using idx_users_email on users"
-- "Heap Fetches: 0" — таблица не читалась!
```

---

> **На собесе:** «Что такое MVCC?» —  
> «Механизм изоляции через версионирование строк. Каждая транзакция видит
> снимок данных на свой момент. UPDATE создаёт новую версию строки,
> старая становится dead tuple. VACUUM чистит dead tuples. Читатели
> не блокируют писателей — основа ACID в PostgreSQL.»