# Kafka: Надёжность и потребление

---

## Гарантии доставки

| Гарантия | Описание | Как включить |
|---|---|---|
| **At-most-once** | Сообщение может быть потеряно | `acks=0` |
| **At-least-once** | Сообщение дойдёт, может дублироваться | `acks=1` (default) |
| **Exactly-once** | Сообщение ровно один раз | `acks=all` + idempotent producer |

```python
# Python — idempotent producer
producer = AIOKafkaProducer(
    bootstrap_servers="localhost:9092",
    enable_idempotence=True,   # гарантия exactly-once
    acks="all",                # ждать все реплики
    max_in_flight_requests=1,  # не более одного не-confirmed
)
```

---

## Consumer и Offset

```python
consumer = AIOKafkaConsumer(
    "orders",
    bootstrap_servers="localhost:9092",
    group_id="order-processor",
    auto_offset_reset="earliest",  # или "latest"
    enable_auto_commit=False,      # ручной commit
)

async for msg in consumer:
    await process(msg)
    await consumer.commit()  # сохранить offset
```

### Offset commit

```
auto.commit = True (default)
  → commit каждые N секунд (auto.commit.interval.ms)
  → риск: consumer упал между commit → повторное чтение
  → last processed offset = commit после process

manual commit (рекомендуется)
  → consumer.commit() после успешной обработки
  → риск: consumer упал до commit → read again (at-least-once)
```

---

## Rebalancing

```
Consumer Group "worker" (3 consumers, 6 partitions)

  ┌─ Consumer A (p0, p1) ──→ rebalance ──→ Consumer A (p0, p3)
  ├─ Consumer B (p2, p3)          │        Consumer B (p1, p5)
  └─ Consumer C (p4, p5)          │        Consumer C (p2, p4)
                                   ↓
                          Consumer D (новый)
                          → rebalance всех!

Во время rebalance — сообщения НЕ обрабатываются!
```

**Как уменьшить rebalancing:**
- `group.id` стабильный
- `session.timeout.ms` — не слишком маленький
- `partition.assignment.strategy` — `CooperativeStickyAssignor` (Kafka 2.4+)

```python
consumer = AIOKafkaConsumer(
    "orders",
    bootstrap_servers="localhost:9092",
    group_id="order-processor",
    session_timeout_ms=30000,       # 30 сек без heartbeat → dead
    heartbeat_interval_ms=3000,     # heartbeat каждые 3 сек
)
```

---

## Exactly-once + Idempotency

```python
# Idempotent producer — Kafka дедуплицирует
producer = AIOKafkaProducer(
    bootstrap_servers="localhost:9092",
    enable_idempotence=True,
    acks="all",
)

# Consumer-side dedup (если нужно)
processed_ids = set()  # или Redis Set
async for msg in consumer:
    event_id = msg.headers.get("event_id", msg.offset)
    if event_id not in processed_ids:
        await process(msg)
        processed_ids.add(event_id)
    await consumer.commit()
```

---

## Подводные камни

| ❌ Ошибка | ✅ Правильно |
|---|---|
| `auto.offset.reset=latest` | Пропустишь все сообщения до старта consumer |
| Один consumer на partition | Много consumers = rebalance каждый раз |
| Не мониторить lag | `kafka-consumer-groups --bootstrap-server --group --describe` |
| `acks=0` — потеря сообщения | Никогда в проде! |
| Enable auto commit на прод | Лучше manual commit |

---

> **Технически:** At-least-once = acks=all + manual commit. Exactly-once =
> idempotent producer (enable_idempotence=True) + manual commit или
> Kafka Transaction API (Kafka 0.11+). Idempotent producer использует
> producer ID + sequence number для дедупликации на стороне брокера.