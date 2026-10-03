# Kafka: Надёжность и потребление (расширенно)

> **Цель:** понять гарантии доставки, offset management, rebalancing.

---

## 1. Гарантии доставки — как настроить

| Гарантия | acks | enable_idempotence | Риск |
|----------|------|-------------------|------|
| **At-most-once** | `acks=0` | ❌ | Потеря сообщения |
| **At-least-once** | `acks=1` (default) | ❌ | Дубликаты при сбое |
| **At-least-once (all)** | `acks=all` | ❌ | Дубликаты при сбое |
| **Exactly-once** | `acks=all` | ✅ | Лучшая гарантия |

```python
# Exactly-once producer
producer = AIOKafkaProducer(
    enable_idempotence=True,  # Kafka дедуплицирует
    acks="all",              # ждать все реплики
    max_in_flight_requests=1, # не более 1 не-confirmed
)
```

**Как работает idempotent producer:** Kafka присваивает каждому producer'у уникальный ID. Каждое сообщение нумеруется (sequence number). Брокер проверяет: если сообщение с таким sequence уже записано — не дублирует.

---

## 2. Offset — что это и как управлять

**Offset = позиция consumer в partition.**

```python
consumer = AIOKafkaConsumer(
    "orders",
    group_id="order-processor",
    auto_offset_reset="earliest",
    enable_auto_commit=False,  # manual commit!
)
```

**Auto-commit:**
- Kafka автоматически коммитит offset каждые N секунд
- Риск: consumer получил сообщение, обработал, но не успел закоммитить → после рестарта прочитает заново (at-least-once)
- Риск 2: consumer получил сообщение, упал до commit → повторит

**Manual commit (рекомендуется):**
```python
async for msg in consumer:
    await process(msg)     # сначала обработать
    await consumer.commit() # потом закоммитить
```

**auto.offset.reset:**
- `earliest` — читать с первого сообщения (новый consumer)
- `latest` — читать только новые (пропустит существующие)

---

> **Технически:** At-least-once = acks=all + manual commit. Exactly-once = idempotent producer + manual commit. Idempotent producer использует producer ID + sequence number для дедупликации на стороне брокера.