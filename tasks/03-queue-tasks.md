# Задачи RabbitMQ / Kafka

---

## Задача 1. Outbox pattern

**Условие:** Опишите реализацию Outbox pattern для гарантии,
что сообщение не потеряется между записью в БД и отправкой в RabbitMQ/Kafka.

<details>
<summary>Решение</summary>

**Проблема:** Приложение записывает в БД, потом отправляет в очередь.
Если упадёт между ними — сообщение потеряно.

**Решение (Outbox pattern):**
1. В одной транзакции с бизнес-записью создаём запись в таблице `outbox`:
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
3. При старте приложения — повторить все pending.

**Гарантии:** Outbox + RabbitMQ publisher confirms = сообщение точно дойдёт.
</details>

---

## Задача 2. Consumer retry с DLQ

**Условие:** Напишите consumer для Kafka, который при ошибке пытается 3 раза
обработать сообщение, а потом отправляет в DLQ (dead letter queue).

<details>
<summary>Решение</summary>

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
                    # Отправляем в retry-топик
                    await producer.send("orders-retry", value=msg.value, headers={"x-retries": retries + 1})
                    await consumer.commit()
                else:
                    # В DLQ
                    await producer.send("orders-dlq", value=msg.value, headers={"error": str(e)})
                    await consumer.commit()
    finally:
        await consumer.stop()
        await producer.stop()
```
</details>

---

## Задача 3. RabbitMQ vs Kafka — проектирование нотификаций

**Условие:** Спроектируйте сервис нотификаций. Пользователь совершает действие →
нужно отправить email и push-уведомление. Что выберете — RabbitMQ или Kafka?

<details>
<summary>Решение</summary>

```
Выбор: RabbitMQ (сообщение нужно доставить 1 раз — email + push)

Архитектура:

  FastAPI → Exchange "notifications" → Queue "email" → Worker (send email)
                                     → Queue "push"  → Worker (send push)

Почему RabbitMQ, а не Kafka:
- Сообщение прочтётся 1 раз (email worker + push worker).
- Не нужно хранить историю нотификаций (Kafka оверхед).
- Нужно подтверждение (ack) от email-сервиса.
- Можно DLX для ошибок (повторная отправка).

Если бы нужен был аудит (кто когда получил нотификацию) — Kafka,
так как можно перечитать историю.
```
</details>

---

> **На собесе:** Задачи на очереди — это про trade-off. Интересует не столько
> код, сколько понимание гарантий доставки, outbox pattern, DLQ.