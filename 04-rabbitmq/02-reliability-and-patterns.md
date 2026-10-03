# RabbitMQ: Надёжность и паттерны — глубоко

> **Цель:** понимать, как гарантировать, что сообщение **точно дойдёт** до consumer, и как организовать retry.

---

## 1. Три уровня гарантий

```
Уровень 1 (Producer → Broker): Publisher Confirms
Уровень 2 (Broker → Consumer): Consumer Ack
Уровень 3 (Бизнес-уровень):     Outbox Pattern
```

### Publisher Confirms

```python
# Включаем confirms:
await channel.confirm_delivery()

# Публикуем:
try:
    await channel.default_exchange.publish(
        aio_pika.Message(
            body=payload,
            delivery_mode=aio_pika.DeliveryMode.PERSISTENT,
        ),
        routing_key="order.created",
        mandatory=True,  # если нет очереди — вернуть PublishError
    )
except aio_pika.exceptions.DeliveryError:
    logger.error("Message NOT delivered to any queue!")
```

**Без confirms:** отправили и забыли. Если брокер упал до записи — сообщение потеряно.

**С confirms:** producer ждёт, что брокер сохранил сообщение (и реплицировал на кворум).

### Consumer Ack

```
Consumer → получил → обработал → basic.ack      → сообщение удаляется
                   → ошибка    → basic.nack(requeue=True)  → возвращается в очередь
                   → фатальная → basic.reject(requeue=False) → DLX
```

```python
async def callback(message: aio_pika.IncomingMessage):
    async with message.process():  # автоматический ack/reject
        try:
            await process_order(message.body)
            # process() делает ack автоматически
        except TemporaryError:
            raise  # process() отправит nack (requeue)
        except PermanentError:
            await message.reject(requeue=False)  # → DLX
```

**Без ack (auto_ack=True):** consumer получил → сообщение удаляется. Если consumer упал во время обработки — сообщение потеряно.

**С ack:** consumer подтверждает **после** обработки. При падении consumer (TCP-разрыв) — сообщение возвращается в очередь.

### Outbox Pattern

**Проблема:**
```
1. INSERT INTO orders (...)    ✅
2. [CRASH! сервер упал!]       ❌
3. publish("order.created")   — не выполнилось!
```

**Решение:**
```
1. BEGIN;
     INSERT INTO orders (...);
     INSERT INTO outbox (topic, payload, status='pending');
   COMMIT;                                    ← атомарно!
2. Фоновый poller: читает outbox, публикует в RabbitMQ
3. После publisher-confirm: outbox.status = 'sent'
4. При старте — повтор pending
```

```sql
CREATE TABLE outbox (
    id BIGSERIAL PRIMARY KEY,
    topic TEXT NOT NULL,
    routing_key TEXT NOT NULL,
    payload JSONB NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | sent | failed
    attempts INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    sent_at TIMESTAMPTZ
);

CREATE INDEX idx_outbox_pending ON outbox(status, created_at)
    WHERE status = 'pending';
```

```python
async def outbox_poller(pool, rmq_channel, batch_size=100, interval=1.0):
    """Читает outbox, публикует, помечает sent."""
    while True:
        async with pool.acquire() as conn:
            events = await conn.fetch(
                """SELECT id, topic, routing_key, payload
                   FROM outbox
                   WHERE status = 'pending'
                   ORDER BY created_at
                   LIMIT $1
                   FOR UPDATE SKIP LOCKED""",
                batch_size,
            )
            for event in events:
                try:
                    exchange = await rmq_channel.get_exchange(event["topic"])
                    await exchange.publish(
                        aio_pika.Message(
                            body=event["payload"].encode(),
                            delivery_mode=aio_pika.DeliveryMode.PERSISTENT,
                            message_id=str(event["id"]),
                        ),
                        routing_key=event["routing_key"],
                    )
                    await conn.execute(
                        "UPDATE outbox SET status='sent', sent_at=NOW() WHERE id=$1",
                        event["id"],
                    )
                except Exception:
                    await conn.execute(
                        "UPDATE outbox SET attempts=attempts+1 WHERE id=$1",
                        event["id"],
                    )
        await asyncio.sleep(interval)
```

---

## 2. Retry с Exponential Backoff

```
Сообщение → orders.work (TTL=1s) → если ошибка → reject → DLX
                                                                     ↓
                                                          orders.retry (TTL=2s, 4s, 8s...)
                                                                     ↓
                                                          orders.dead (после N попыток)
```

```python
async def publish_with_retry_headers(channel, event):
    retry_count = int(event.headers.get("x-retry-count", 0))
    if retry_count >= 3:
        # DLQ
        await channel.default_exchange.publish(
            aio_pika.Message(body=event.body, headers={"x-error": str(last_error)}),
            routing_key="orders.dead",
        )
    else:
        delay = 2 ** retry_count  # 1, 2, 4
        delay_ms = delay * 1000
        await channel.default_exchange.publish(
            aio_pika.Message(
                body=event.body,
                headers={"x-retry-count": retry_count + 1},
                expiration=str(delay_ms),  # TTL сообщения
            ),
            routing_key="orders.retry",
        )
```

---

## 3. Connection resilience

```python
# connect_robust — reconnect автоматически:
connection = await aio_pika.connect_robust(
    "amqp://user:pass@localhost/vhost",
    reconnect_interval=1.0,   # начальный интервал
    reconnect_max_interval=30.0,  # максимальный интервал (backoff)
)

# Health check:
async def health_check(connection) -> bool:
    try:
        channel = await connection.channel()
        await channel.close()
        return True
    except Exception:
        return False
```

---

## 4. Мониторинг

```bash
# Размер очередей
rabbitmqctl list_queues name messages messages_ready messages_unacknowledged

# Статистика exchange
rabbitmqctl list_exchanges name type

# Connections
rabbitmqctl list_connections name state peer_host

# В целом
rabbitmq-diagnostics status
rabbitmq-diagnostics check_alarms
```

---

> **На собесе:** «Как гарантировать доставку в RabbitMQ?» —
> «Три уровня:
> 1) Publisher confirms — producer подтверждает, что брокер получил.
> 2) Consumer ack — consumer подтверждает, что обработал.
> 3) Outbox pattern — атомарная запись в БД + очередь (решает проблему «записал в БД, но не отправил»).
> DMQ для упавших с retry + exponential backoff. Quorum queues для критичных данных (RAFT-консенсус).»