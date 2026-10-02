# RabbitMQ: Надёжность и паттерны

---

## Publisher Confirms

```python
import pika

# Включить подтверждения публикации
channel.confirm_delivery()

def on_publish_confirm(frame):
    if isinstance(frame, pika.frame.ConfirmFrame):
        print("Message delivered to broker")
    else:
        print("Message lost!")

channel.basic_publish(
    exchange="orders",
    routing_key="order.created",
    body=msg,
    mandatory=True,  # требовать очередь
)
```

---

## Consumer Ack

```python
def callback(ch, method, properties, body):
    try:
        process_order(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except TemporaryError:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)
    except PermanentError:
        ch.basic_reject(delivery_tag=method.delivery_tag, requeue=False)  # → DLX
```

| Действие | Поведение |
|---|---|
| `ack` | Сообщение удалено из очереди |
| `nack + requeue=True` | Сообщение возвращается в очередь |
| `reject + requeue=False` | Сообщение → DLX |
| TCP упал без ack | Сообщение возвращается в очередь (после timeout) |

---

## Ретраи (Outbox Pattern)

```
                    ┌──────────────┐
  POST /api/orders  │    Outbox    │
       ───────────→ │ (таблица БД) │──→ Poller ───→ RabbitMQ
                    └──────────────┘
```

```python
# Outbox pattern — atomic write to DB + send
async def create_order(payload):
    async with db.begin():
        order = await repo.create(payload)
        # Сохраняем в outbox (таблица БД)
        await outbox_repo.add(order.id, "order.created")

# Poller читает outbox и шлёт в RabbitMQ
async def outbox_poller():
    while True:
        events = await outbox_repo.get_pending()
        for event in events:
            try:
                await publish(event.topic, event.payload)
                await outbox_repo.mark_done(event.id)
            except Exception:
                pass  # retry next poll
        await asyncio.sleep(1)
```

---

## TTL (Time-To-Live)

```python
# TTL сообщения (24 часа)
channel.basic_publish(
    exchange="orders",
    routing_key="order.expired",
    body=msg,
    properties=pika.BasicProperties(
        expiration="86400000",  # 24h в ms
    ),
)

# TTL очереди
channel.queue_declare(
    queue="temporary",
    arguments={"x-message-ttl": 3600000}  # 1 час
)
```

---

## Quorum Queues (кластер)

```python
# Гарантированная доставка в кластере
channel.queue_declare(
    queue="important",
    arguments={
        "x-queue-type": "quorum",
        "x-quorum-initial-cluster-size": 3,
    }
)
```

---

## RabbitMQ vs Kafka (коротко)

| RabbitMQ | Kafka |
|---|---|
| Умный брокер, тупой consumer | Тупой брокер, умный consumer |
| Сообщение — 1 раз (ack — удалено) | Хранится, пока не expire |
| Порядок в одной очереди | Порядок в одной партиции |
| Лучше: синхронные RPC, подтверждения | Лучше: стримы, реплаи, Big Data |

---

## Подводные камни

| ❌ Ошибка | ✅ Правильно |
|---|---|
| `nack` без задержки — бесконечный цикл | `basic_reject` → DLX |
| Не включить publisher confirms | Потеря сообщения на стороне брокера |
| Одна очередь для всего | Разделить по типу: orders, notifications |
| Нет мониторинга | `rabbitmqctl list_queues` — размер, consumer count |

---

> **Технически:** Publisher confirms + consumer ack = at-least-once доставка.
> Outbox pattern гарантирует, что сообщение записано в БД до отправки.
> Quorum queues обеспечивают согласованность в кластере.