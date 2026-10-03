# Задачи RabbitMQ / Kafka — 5 задач с разбором

---

## Задача 1. Outbox pattern (полная реализация)

**Проблема:** сервер записал в БД, но упал до отправки в очередь → сообщение потеряно.

**Решение:**
```sql
-- В ОДНОЙ транзакции:
BEGIN;
  INSERT INTO orders (...) VALUES (...);
  INSERT INTO outbox (topic, routing_key, payload, status) VALUES ('orders', 'order.created', '{}', 'pending');
COMMIT;
```

```python
async def outbox_poller(pool, rmq_exchange, batch_size=100):
    while True:
        async with pool.acquire() as conn:
            events = await conn.fetch(
                """SELECT id, routing_key, payload FROM outbox
                   WHERE status='pending' ORDER BY created_at LIMIT $1
                   FOR UPDATE SKIP LOCKED""", batch_size)
            for e in events:
                await rmq_exchange.publish(
                    aio_pika.Message(body=e["payload"].encode(), delivery_mode=PERSISTENT),
                    routing_key=e["routing_key"])
                await conn.execute("UPDATE outbox SET status='sent', sent_at=NOW() WHERE id=$1", e["id"])
        await asyncio.sleep(1)
```

**Гарантии:** outbox + publisher confirms = сообщение точно дойдёт.

---

## Задача 2. Consumer retry с DLQ (Kafka)

```python
async def process_with_retry():
    consumer = AIOKafkaConsumer("orders", group_id="processor", enable_auto_commit=False)
    producer = AIOKafkaProducer()
    await consumer.start(); await producer.start()
    try:
        async for msg in consumer:
            retries = int(msg.headers.get("x-retries", "0"))
            try:
                await handle(msg.value)
                await consumer.commit()
            except TemporaryError:
                if retries < 3:
                    await producer.send("orders-retry", value=msg.value,
                        headers=[("x-retries", str(retries+1).encode())])
                else:
                    await producer.send("orders-dlq", value=msg.value)
                await consumer.commit()  # ВСЕГДА коммитим!
            except PermanentError:
                await producer.send("orders-dlq", value=msg.value)
                await consumer.commit()
    finally:
        await consumer.stop(); await producer.stop()
```

**Ключевое:** коммитим offset всегда, чтобы не блокировать partition.

---

## Задача 3. RabbitMQ vs Kafka — проектирование нотификаций

**Требование:** действие → email + push-уведомление.

**Выбор: RabbitMQ.**
- Сообщение доставить 2 обработчикам: email и push (Fanout exchange)
- Ack: email-сервис подтверждает отправку
- DLX: для ошибок (повторная попытка)
- Не нужна история — после отправки сообщение не нужно

**Архитектура:**
```
FastAPI → Exchange(notifications, fanout) → Queue "email" → Worker Email
                                           → Queue "push"  → Worker Push
```

**Когда выбрали бы Kafka:** если нужен аудит нотификаций (кто, когда получил) или несколько consumer-групп читают одни и те же события.

---

## Задача 4. Idempotent consumer (дедупликация)

```python
async def idempotent_handle(msg, redis: Redis):
    # topic + partition + offset уникален для каждого сообщения в Kafka
    dedup_key = f"dedup:{msg.topic}:{msg.partition}:{msg.offset}"
    if await redis.exists(dedup_key):
        logger.info(f"Skipping duplicate: {dedup_key}")
        return
    await process(msg.value)
    await redis.setex(dedup_key, 86400, "1")  # храним 24 часа
```

---

## Задача 5. Graceful shutdown consumer

```python
async def consume_with_graceful_shutdown():
    consumer = AIOKafkaConsumer("orders", ...)
    await consumer.start()
    shutdown = asyncio.Event()

    async def handle_shutdown():
        loop = asyncio.get_running_loop()
        for sig in (signal.SIGINT, signal.SIGTERM):
            loop.add_signal_handler(sig, lambda: asyncio.create_task(set_shutdown()))
        async def set_shutdown():
            logger.info("Shutdown signal received")
            shutdown.set()
        await handle_shutdown()

    try:
        while not shutdown.is_set():
            # poll с коротким таймаутом чтобы проверять shutdown
            data = await consumer.getmany(timeout_ms=1000)
            for tp, messages in data.items():
                for msg in messages:
                    await process(msg.value)
                    await consumer.commit()
    finally:
        logger.info("Stopping consumer...")
        await consumer.stop()
        logger.info("Consumer stopped")
```