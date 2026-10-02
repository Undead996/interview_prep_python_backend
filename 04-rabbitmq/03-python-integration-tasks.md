# RabbitMQ + Python: интеграция и задачи

---

## aio-pika (async/await)

```python
import aio_pika
import asyncio

async def main():
    connection = await aio_pika.connect_robust(
        "amqp://guest:guest@localhost/"
    )

    async with connection:
        channel = await connection.channel()

        # Producer
        await channel.default_exchange.publish(
            aio_pika.Message(
                body=b"Hello",
                delivery_mode=aio_pika.DeliveryMode.PERSISTENT,
            ),
            routing_key="test.queue",
        )

        # Consumer
        queue = await channel.declare_queue("test.queue", durable=True)
        async with queue.iterator() as queue_iter:
            async for message in queue_iter:
                async with message.process():
                    print(message.body.decode())
                    # auto-ack + защита от потери

asyncio.run(main())
```

---

## pika (синхронный)

```python
import pika

# Producer
connection = pika.BlockingConnection(pika.URLParameters("amqp://guest:guest@localhost/"))
channel = connection.channel()
channel.basic_publish(
    exchange="",
    routing_key="task_queue",
    body=b"Task body",
    properties=pika.BasicProperties(
        delivery_mode=2,  # persistent
    ),
)

# Consumer
def callback(ch, method, properties, body):
    print(f"Received: {body}")
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(queue="task_queue", on_message_callback=callback)
channel.start_consuming()
```

---

## FastAPI + RabbitMQ

```python
from fastapi import FastAPI
from contextlib import asynccontextmanager
import aio_pika

@asynccontextmanager
async def lifespan(app):
    app.state.rmq = await aio_pika.connect_robust("amqp://guest:guest@localhost/")
    app.state.channel = await app.state.rmq.channel()
    yield
    await app.state.rmq.close()

app = FastAPI(lifespan=lifespan)

@app.post("/api/orders")
async def create_order(payload: OrderCreate):
    # Бизнес-логика
    order = await create_order_in_db(payload)

    # Публикация в RabbitMQ
    await app.state.channel.default_exchange.publish(
        aio_pika.Message(
            body=order.model_dump_json().encode(),
            delivery_mode=aio_pika.DeliveryMode.PERSISTENT,
        ),
        routing_key="order.created",
    )
    return order
```

---

## Задача 1. Подтверждение доставки

**Условие:** Напишите consumer с retry 3 раза, после — DLX.

<details>
<summary>Решение</summary>

```python
import aio_pika

MAX_RETRIES = 3

async def process_with_retry(message: aio_pika.IncomingMessage):
    try:
        # Обработка
        await process(message.body)
        await message.ack()
    except Exception:
        # Проверяем, сколько раз уже пытались
        retry_count = message.headers.get("x-retry", 0)
        if retry_count < MAX_RETRIES:
            headers = dict(message.headers or {})
            headers["x-retry"] = retry_count + 1
            # Re-queue с увеличенной задержкой
            await message.reject(requeue=False)
            await channel.default_exchange.publish(
                aio_pika.Message(
                    body=message.body,
                    headers=headers,
                    delivery_mode=aio_pika.DeliveryMode.PERSISTENT,
                ),
                routing_key="order.retry",
            )
        else:
            # Отправляем в DLX
            await message.reject(requeue=False)
```
</details>

---

## Задача 2. Outbox pattern

**Условие:** Опишите, как гарантировать, что сообщение не потеряется при падении
приложения между записью в БД и отправкой в RabbitMQ.

<details>
<summary>Решение</summary>

**Outbox pattern:**
1. В той же транзакции, что и запись в БД, пишем запись в таблицу `outbox`.
2. Фоновый poller читает `outbox`, отправляет в RabbitMQ.
3. После подтверждения от RabbitMQ — помечает `outbox.status = 'done'`.
4. Если упали — на старте читаем `outbox WHERE status = 'pending'`, повторяем.

```sql
CREATE TABLE outbox (
    id BIGSERIAL,
    topic TEXT NOT NULL,
    payload JSONB NOT NULL,
    status TEXT DEFAULT 'pending',
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```
</details>

---

> **На собесе:** «RabbitMQ — как гарантировать доставку?» —
> «Publisher confirms (на стороне producer), consumer ack (на стороне consumer),
> DLX для упавших сообщений, outbox pattern для гарантии записи + отправки.»