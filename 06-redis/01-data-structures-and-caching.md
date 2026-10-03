# Redis: структуры данных и кэширование (расширенно)

> **Цель:** понять, какие структуры когда выбирать, как работает кэш-стратегия.

---

## 1. Почему Redis быстрый

Redis — **in-memory** (данные в RAM) и **однопоточный** (одна команда за раз). Это значит:
- Все операции **атомарны** — нет гонок между командами
- Нет оверхеда на блокировки
- Запрос к ключу — O(1) или O(log N)

**Цена:** если Redis упадёт без persistence — данные потеряны (если не настроен AOF/RDB).

---

## 2. Когда какую структуру выбирать

| Задача | Структура | Пример |
|--------|-----------|--------|
| Кэш сериализованного объекта | String (JSON) | `setex user:42 "{...}" 3600` |
| Счётчик | String (INCR) | `incr "page_views"` |
| Очередь задач | List | `lpush tasks job1; rpop tasks` |
| Уникальные значения | Set | `sadd "emails" "alice@ex.com"` |
| Объект с полями | Hash | `hset user:42 name "Alice" age 30` |
| Рейтинг | Sorted Set | `zadd leaderboard 100 "Alice"` |
| Надёжная очередь | Stream | `xadd mystream "order_id" 42` |

---

## 3. Стратегии кэширования — когда что

### Cache Aside (самая популярная)
```
1. Читаем из Redis
2. Если нет → читаем из БД → пишем в Redis (с TTL)
3. Отдаём
```

**Плюсы:** просто, гибко, TTL решает инвалидацию.
**Минусы:** первый запрос — медленный (промах).

### Write Through
```
Запись: БД + Redis (атомарно)
Чтение: из Redis (всегда есть)
```

**Плюсы:** консистентность.
**Минусы:** запись медленнее (две операции).

### Write Behind
```
Запись: только в Redis → асинхронно в БД
```

**Плюсы:** запись очень быстрая.
**Минусы:** риск потери данных (Redis упал → данные в очереди потеряны).

---

## 4. Streams — надёжная очередь (вместо List)

Redis List — простая очередь. Если consumer упал и не rpop'нул — сообщение потеряно.

Redis Stream — с consumer groups, ack, блокирующим чтением:

```python
# Producer
await r.xadd("mystream", {"event": "order.created", "order_id": "42"})

# Consumer group
await r.xgroup_create("mystream", "processors", id="0")
messages = await r.xreadgroup("processors", "worker1", {"mystream": ">"})
for msg_id, fields in messages:
    await process(fields)
    await r.xack("mystream", "processors", msg_id)
```

**Когда Stream вместо List:** когда нужна гарантия доставки и мониторинг.

---

> **На собесе:** «Какие структуры Redis используете?» —  
> «Strings для кэша (JSON-сериализованный объект), Hashes для полей объекта,
> Sorted Sets для рейтингов и rate limiting, Streams для надёжных очередей.
> List — только для временных очередей без гарантий.»