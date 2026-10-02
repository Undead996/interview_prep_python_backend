# SQL-задачи с разбором (8 штук)

---

## Задача 1. Вторая по величине зарплата

```sql
-- employee(id, name, salary)
SELECT MAX(salary) FROM employee
WHERE salary < (SELECT MAX(salary) FROM employee);

-- Или
SELECT salary FROM employee
ORDER BY salary DESC
OFFSET 1 LIMIT 1;
```

---

## Задача 2. Пользователи без заказов

```sql
-- users(id, name), orders(id, user_id, total)
SELECT u.id, u.name
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.id IS NULL;

-- EXISTS — быстрее на больших таблицах
SELECT u.id, u.name
FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);
```

---

## Задача 3. Топ-3 продукта за месяц

```sql
-- products(id, name), order_items(order_id, product_id, qty), orders(id, created_at)
SELECT p.name, SUM(oi.qty * oi.price) AS total
FROM products p
JOIN order_items oi ON oi.product_id = p.id
JOIN orders o ON o.id = oi.order_id
WHERE o.created_at >= '2024-01-01' AND o.created_at < '2024-02-01'
GROUP BY p.id, p.name
ORDER BY total DESC
LIMIT 3;
```

---

## Задача 4. Дубликаты email

```sql
-- users(id, email, name)
SELECT email, COUNT(*)
FROM users
GROUP BY email
HAVING COUNT(*) > 1;

-- Полная информация
SELECT u.*, dup.cnt
FROM users u
JOIN (
    SELECT email, COUNT(*) AS cnt
    FROM users
    GROUP BY email
    HAVING COUNT(*) > 1
) dup ON dup.email = u.email
ORDER BY u.email, u.id;
```

---

## Задача 5. Средний чек по дням (последняя неделя)

```sql
SELECT
    created_at::date AS day,
    COUNT(*) AS orders,
    AVG(total)::numeric(10,2) AS avg_check,
    SUM(total) AS revenue
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
    SUM(amount) OVER (ORDER BY sale_date) AS running_total
FROM daily_sales
ORDER BY sale_date;
```

---

## Задача 7. Департамент с макс. средней зарплатой

```sql
-- employee(id, name, salary, department_id)
-- department(id, name)
SELECT d.name, AVG(e.salary) AS avg_salary
FROM department d
JOIN employee e ON e.department_id = d.id
GROUP BY d.id, d.name
ORDER BY avg_salary DESC
LIMIT 1;
```

---

## Задача 8. Запрос с оконной функцией (категория + ранг)

```sql
-- sales(id, product_id, amount, sale_date)
-- products(id, name, category_id)
-- categories(id, name)
SELECT
    c.name AS category,
    p.name AS product,
    s.amount,
    ROW_NUMBER() OVER (PARTITION BY c.id ORDER BY s.amount DESC) AS rank_in_category
FROM sales s
JOIN products p ON p.id = s.product_id
JOIN categories c ON c.id = p.category_id
ORDER BY c.name, rank_in_category;
```

---

> **На собесе:** Начинай с JOIN и агрегации. Если задача на ранги — оконные функции.
> CTE — для читаемости. Всегда объясняй, почему выбрал этот способ.