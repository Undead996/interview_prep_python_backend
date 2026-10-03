# SQL-задачи с полным разбором — 12 штук

> **Цель:** написать решение без гугла за 5 минут. Понимать, **почему** это решение, а не альтернативное.

---

## Задача 1. Вторая по величине зарплата

```sql
-- employee(id, name, salary)
-- Если есть дубликаты — вернуть вторую уникальную

-- Решение 1: подзапрос (работает с дубликатами)
SELECT MAX(salary) AS second_highest
FROM employee
WHERE salary < (SELECT MAX(salary) FROM employee);

-- Решение 2: OFFSET + DISTINCT
SELECT DISTINCT salary FROM employee
ORDER BY salary DESC
OFFSET 1 LIMIT 1;

-- Решение 3: оконная функция DENSE_RANK
SELECT salary FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employee
) ranked
WHERE rnk = 2
LIMIT 1;
-- DENSE_RANK: 100=1, 90=2, 90=2, 80=3 (не пропускает ранги при одинаковых)
```

**На собесе:** спросят «а что если есть несколько сотрудников со второй зарплатой?» — ответ: DENSE_RANK или DISTINCT.

---

## Задача 2. Пользователи без заказов

```sql
-- Решение 1: LEFT JOIN + NULL-check
SELECT u.id, u.name
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.id IS NULL;

-- Решение 2: NOT EXISTS (часто быстрее на больших таблицах)
SELECT u.id, u.name
FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);

-- Решение 3: NOT IN (осторожно с NULL!)
SELECT id, name FROM users
WHERE id NOT IN (SELECT DISTINCT user_id FROM orders);
-- ⚠️ Если в подзапросе есть NULL — NOT IN вернёт ПУСТОЙ результат!
```

**Когда EXISTS быстрее:** когда orders огромна, а users без заказов — мало. EXISTS останавливается на первом совпадении.

---

## Задача 3. Топ-3 продукта за месяц

```sql
SELECT p.name, SUM(oi.quantity * oi.price) AS revenue
FROM products p
JOIN order_items oi ON oi.product_id = p.id
JOIN orders o ON o.id = oi.order_id
WHERE o.created_at >= '2024-01-01'
  AND o.created_at <  '2024-02-01'
GROUP BY p.id, p.name
ORDER BY revenue DESC
LIMIT 3;
```

**Почему `GROUP BY p.id, p.name`?** В строгом PG — все неагрегированные поля должны быть в GROUP BY. `p.id` уникален, но `p.name` может повторяться.

---

## Задача 4. Дубликаты email

```sql
-- Только email и количество:
SELECT email, COUNT(*) AS cnt
FROM users
GROUP BY email
HAVING COUNT(*) > 1;

-- Полные строки дубликатов:
SELECT u.*
FROM users u
JOIN (
    SELECT email FROM users
    GROUP BY email
    HAVING COUNT(*) > 1
) dup ON dup.email = u.email
ORDER BY u.email, u.id;

-- Через оконную функцию:
SELECT * FROM (
    SELECT *, COUNT(*) OVER (PARTITION BY email) AS cnt
    FROM users
) ranked
WHERE cnt > 1;
```

---

## Задача 5. Средний чек по дням за последнюю неделю

```sql
SELECT
    created_at::date AS day,
    COUNT(*) AS order_count,
    ROUND(AVG(total), 2) AS avg_check,
    SUM(total) AS total_revenue
FROM orders
WHERE created_at >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY created_at::date
ORDER BY day;
```

---

## Задача 6. Накопительный итог (running total)

```sql
SELECT
    sale_date,
    amount,
    SUM(amount) OVER (ORDER BY sale_date ROWS UNBOUNDED PRECEDING) AS running_total,
    SUM(amount) OVER (PARTITION BY EXTRACT(YEAR FROM sale_date) ORDER BY sale_date) AS yearly_running_total
FROM daily_sales
ORDER BY sale_date;
```

**`ROWS UNBOUNDED PRECEDING`** — явно указываем окно. По умолчанию `RANGE UNBOUNDED PRECEDING` — может вести себя иначе при дубликатах.

---

## Задача 7. Департамент с максимальной средней зарплатой

```sql
SELECT d.name, ROUND(AVG(e.salary), 2) AS avg_salary
FROM department d
JOIN employee e ON e.department_id = d.id
GROUP BY d.id, d.name
ORDER BY avg_salary DESC
LIMIT 1;
```

**Если несколько департаментов с одинаковой макс. средней — выдаст один. Исправить:**

```sql
WITH ranked AS (
    SELECT d.name, AVG(e.salary) AS avg_salary,
           RANK() OVER (ORDER BY AVG(e.salary) DESC) AS rnk
    FROM department d
    JOIN employee e ON e.department_id = d.id
    GROUP BY d.id, d.name
)
SELECT name, avg_salary FROM ranked WHERE rnk = 1;
```

---

## Задача 8. Категория + ранг продукта по продажам

```sql
SELECT
    c.name AS category,
    p.name AS product,
    SUM(s.amount) AS total_sales,
    ROW_NUMBER() OVER (PARTITION BY c.id ORDER BY SUM(s.amount) DESC) AS rank_in_category
FROM sales s
JOIN products p ON p.id = s.product_id
JOIN categories c ON c.id = p.category_id
GROUP BY c.id, c.name, p.id, p.name
ORDER BY c.name, rank_in_category;
```

**`ROW_NUMBER()` vs `RANK()` vs `DENSE_RANK()`:**
- `ROW_NUMBER()`: 1, 2, 3 (даже если равны — уникален)
- `RANK()`: 1, 1, 3 (пропускает ранг при равенстве)
- `DENSE_RANK()`: 1, 1, 2 (не пропускает)

---

## Задача 9. Удалить дубликаты, оставив последний по дате

```sql
DELETE FROM users a
USING users b
WHERE a.email = b.email
  AND a.created_at < b.created_at;
-- Оставляет запись с максимальным created_at для каждого email

-- Или через CTE:
WITH duplicates AS (
    SELECT id,
           ROW_NUMBER() OVER (PARTITION BY email ORDER BY created_at DESC) AS rn
    FROM users
)
DELETE FROM users
WHERE id IN (SELECT id FROM duplicates WHERE rn > 1);
```

---

## Задача 10. Найти пропуски в последовательности ID

```sql
-- Сравниваем текущий ID со следующим:
SELECT cur.id + 1 AS gap_start
FROM users cur
LEFT JOIN users nxt ON nxt.id = cur.id + 1
WHERE nxt.id IS NULL
  AND cur.id < (SELECT MAX(id) FROM users);
-- Получаем: 3 (если пропущены 2), 7 (если пропущены 6)
```

---

## Задача 11. Количество заказов на пользователя с процентом

```sql
SELECT
    u.name,
    COUNT(o.id) AS order_count,
    ROUND(COUNT(o.id) * 100.0 / SUM(COUNT(o.id)) OVER (), 2) AS pct_of_total
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name
ORDER BY order_count DESC;
```

---

## Задача 12. Клиенты, купившие товары из ВСЕХ категорий

```sql
-- division: найти user_id, для которых НЕ существует категории без их заказа
SELECT user_id
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
GROUP BY user_id
HAVING COUNT(DISTINCT p.category_id) = (SELECT COUNT(*) FROM categories);
```

---

## Сводка: когда что использовать

| Задача | Ключевая техника |
|--------|-----------------|
| Топ-N | `ORDER BY ... DESC LIMIT N` |
| Ранги внутри группы | `ROW_NUMBER() / RANK() / DENSE_RANK() OVER (PARTITION BY ...)` |
| Накопительная сумма | `SUM() OVER (ORDER BY ...)` |
| Пользователи без X | `LEFT JOIN ... WHERE ... IS NULL` или `NOT EXISTS` |
| Дубликаты | `GROUP BY ... HAVING COUNT(*) > 1` |
| Удаление дубликатов | `ROW_NUMBER() OVER (PARTITION BY ...)` + DELETE |
| Иерархии | `WITH RECURSIVE` |
| Division (все категории) | `HAVING COUNT(DISTINCT ...) = (SELECT COUNT(*) FROM ...)` |

---

> **На собесе:** всегда начинай с `JOIN` и `GROUP BY`. Если задача на ранги — оконные функции. `CTE` для читаемости. Всегда объясняй **почему** выбрал этот способ — интервьюер оценит мышление выше, чем синтаксис.