# Мок-интервью: Redis (15 вопросов) — расширенные ответы

---

## Q1. Какие структуры данных в Redis?

8 структур: **String** (кэш, счётчики), **List** (простые очереди/стеки), **Set** (уникальные), **Hash** (объекты с атомарными полями), **Sorted Set** (рейтинги, rate limiting), **Stream** (надёжные очереди с ack), **Bitmap** (флаги, аналитика), **HyperLogLog** (приблизительный подсчёт уникальных, 12KB).

---

## Q2. TTL — что происходит при истечении?

**Пассивно:** при GET/любом обращении — проверка, истёк → удалён. **Активно:** каждые 100ms Redis сканирует случайную выборку (~20) expired-ключей. Если > 25% истекли — повторяет.

---

## Q3. Redis как очередь — List vs Streams?

**List (LPUSH/RPOP):** простая очередь, без гарантий — consumer упал → сообщение потеряно. **Streams (XADD/XREADGROUP):** consumer groups, ack (XACK), блокирующее чтение, мониторинг pending. Для продакшена — Streams.

---

## Q4. RDB vs AOF?

**RDB:** снапшот (dump.rdb), компактный, быстрый старт, потеря последних минут. **AOF:** лог запросов, потеря ~1 секунды (everysec), больше места. **Рекомендация:** RDB + AOF гибридный режим.

---

## Q5. Sentinel vs Cluster?

**Sentinel:** High Availability (failover) без шардирования: 3+ Sentinel-узлов мониторят мастер, при падении выбирают новый из реплик. **Cluster:** шардирование (16384 слотов) + HA. Для больших данных (не помещаются на одной ноде).

---

## Q6. Как инвалидировать кэш?

1. **TTL** (setex) — автоматическое истечение.
2. **DEL** при изменении данных (Cache Aside).
3. **Versioned keys** — `user:42:v2`.
4. **Write Through** — БД + Redis синхронно.
5. **Write Behind** — пишем в Redis, асинхронно в БД (риск потери).

---

## Q7. Distributed Lock — как правильно?

```python
# Acquire: SET lock NX PX ttl
lock_value = str(uuid.uuid4())
acquired = await redis.set(f"lock:{name}", lock_value, nx=True, ex=ttl)

# Release (Lua — атомарно!):
script = """
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
end
"""
```

**Почему Lua:** без Lua возможна гонка: между GET и DEL другой процесс может установить свой замок.

---

## Q8. Sliding window vs Token bucket?

**Sliding window:** `INCR ratelimit:{IP}:{minute}` + `EXPIRE`. Просто, эффективно, неточное на границе окна. **Token bucket:** пополняется с фиксированной скоростью. Точнее, но нужен Lua-скрипт. Для большинства случаев достаточно sliding window.

---

## Q9. Pub/Sub vs Streams?

**Pub/Sub (PUBLISH/SUBSCRIBE):** fire-and-forget — если subscriber не в сети → сообщение потеряно. **Streams:** persistent, consumer groups, ack. Для надёжной доставки — Streams.

---

## Q10. Pipeline vs Transaction?

**Pipeline:** группа команд одним round-trip, НЕ атомарно. Быстро. **Transaction (MULTI/EXEC):** атомарно. Медленнее (блокировка на время EXEC). Для инкрементов — pipeline, для консистентности — transaction.

---

## Q11. Почему Redis однопоточный?

Простота: нет блокировок, нет race condition. Все операции атомарны. Высокая скорость: данные в RAM, O(1) для большинства операций. Многопоточность появилась с Redis 6 (только для I/O, не для команд).

---

## Q12. Как посчитать уникальных посетителей?

**Точно:** `PFADD unique:visitors 192.168.1.1` (HyperLogLog). ~0.81% ошибка, фиксированные 12KB памяти. **В 100 раз меньше памяти, чем Set.**

---

## Q13. Cache stampede — что это и как избежать?

Множество одновременных запросов при cache miss → все идут в БД. **Решение:** locking (SETNX lock перед запросом к БД), probabilistic early recompute (пересоздаём кэш до истечения TTL).

---

## Q14. Что такое Redis `KEYS *` и почему нельзя?

Сканирует ВЕСЬ keyspace. На проде с миллионами ключей — блокирует Redis на секунды/минуты. **Заменить:** `SCAN 0 MATCH user:* COUNT 100` — итеративный, неблокирующий.

---

## Q15. `FLUSHALL` vs `FLUSHDB`?

`FLUSHDB` — очищает текущую DB (0). `FLUSHALL` — очищает ВСЕ DB (0-15). ASYNC опция (Redis 4+) — фоновое удаление без блокировки. На проде — только с `ASYNC` и в maintenance-окно.