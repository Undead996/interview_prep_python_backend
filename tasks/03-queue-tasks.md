# Задачи RabbitMQ / Kafka — расширенный разбор

---

## Задача 1. Outbox pattern

**Условие:** Опишите реализацию Outbox pattern для гарантии, что сообщение не потеряется между записью в БД и отправкой в RabbitMQ/Kafka.

**Проблема:** Приложение записывает в БД, потом отправляет в очередь. Если упадёт между ними — сообщение потеряно.

**Решение (Outbox pattern):**
1. В **одной транзакции** с бизнес-записью создаём запись в таблице `outbox`:

```sql
BEGIN;
  INSERT INTO orders (...);
  INSERT INTO outbox (topic, payload, status) VALUES ('order.created', '{...}', 'pending');
COMMIT;
```

2. Фоновый poller читает `outbox WHERE status = 'pending'`:

```python
async def outbox_poller(rabbit):
    while True:
        events = await outbox_repo.get_pending(limit=100)
        for event in events:
            try:
                await rabbit.publish(event.topic, event.payload)
                await outbox_repo.mark_done(event.id)
            except Exception:
                logger.warning(f"Outbox event {event.id} failed, will retry")
        await asyncio.sleep(1)
```

3. **При старте приложения** — повторить все pending.

**Гарантии:** Outbox + RabbitMQ publisher confirms = сообщение точно дойдёт.

---

## Задача 2. Consumer retry с DLQ

**Условие:** Consumer для Kafka, который при ошибке пытается 3 раза обработать сообщение, а потом отправляет в DLQ.

```python
async def process_with_retry():
    consumer = AIOKafkaConsumer("orders", bootstrap_servers="localhost:9092", group_id="processor")
    producer = AIOKafkaProducer(bootstrap_servers="localhost:9092")
    await consumer.start()
    await producer.start()

    try:
        async for msg in consumer:
            retries = msg.headers.get("x-retries", 0)
            try:
                await handle(msg.value)
                await consumer.commit()
            except Exception as e:
                if retries < 3:
                    await producer.send("orders-retry", value=msg.value, headers={"x-retries": retries + 1})
                    await consumer.commit()
                else:
                    await producer.send("orders-dlq", value=msg.value, headers={"error": str(e)})
                    await consumer.commit()
    finally:
        await consumer.stop()
        await producer.stop()
```

**Ключевое:** мы коммитим offset даже при ошибке — чтобы не блокировать чтение новых сообщений. Упавшее уходит в retry-топик.

---

## Задача 3. RabbitMQ vs Kafka — проектирование нотификаций

**Условие:** Спроектируйте сервис нотификаций. Пользователь совершает действие → нужно отправить email и push-уведомление.

**Выбор:** RabbitMQ (сообщение нужно доставить 1 раз).

**Архитектура:**

```
FastAPI → Exchange "notifications" → Queue "email" → Worker (send email)
                                   → Queue "push"  → Worker (send push)
```

**Почему RabbitMQ, а не Kafka:**
- Сообщение прочтётся 1 раз (email worker + push worker) — не нужно хранить историю
- Нужно подтверждение (ack) от email-сервиса
- DLX для ошибок (повторная отправка)
- Простота: Fanout exchange решит задачу

**Когда выбрали бы Kafka:** если нужен аудит (кто когда получил нотификацию).