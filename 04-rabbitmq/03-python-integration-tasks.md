# RabbitMQ + Python: интеграция и задачи — расширенно

---

## aio-pika — асинхронная работа с RabbitMQ

```python
import aio_pika
import asyncio

async def main():
    connection = await aio_pika.connect_robust("amqp://guest:guest@localhost/")
    async with connection:
        channel = await connection.channel()
        
        # Producer
        await channel.default_exchange.publish(
            aio_pika.Message(
                body=b"Hello",
                delivery_mode=aio_pika.DeliveryMode.PERSISTENT,  # durable
            ),
            routing_key="test.queue",
        )
        
        # Consumer
        queue = await channel.declare_queue("test.queue", durable=True)
        async with queue.iterator() as queue_iter:
            async for message in queue_iter:
                async with message.process():
                    print(message.body.decode())
```

**`connect_robust`** — reconnect при потере соединения. Используйте в production.

---

## FastAPI + RabbitMQ

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app):
    # Старт
    app.state.rmq = await aio_pika.connect_robust("amqp://guest:guest@localhost/")
    app.state.channel = await app.state.rmq.channel()
    yield
    # Shutdown
    await app.state.rmq.close()

app = FastAPI(lifespan=lifespan)

@app.post("/api/orders")
async def create_order(payload: OrderCreate):
    order = await create_order_in_db(payload)
    await app.state.channel.default_exchange.publish(
        aio_pika.Message(body=order.model_dump_json().encode()),
        routing_key="order.created",
    )
    return order
```

**Зачем publish в эндпоинте, а не в background task?** Чтобы сразу вернуть ответ клиенту, а отправка в очередь — "fire and forget". Если нужна гарантия — outbox pattern.

---

> **На собесе:** «RabbitMQ — как гарантировать доставку?» —  
> «Publisher confirms (на стороне producer), consumer ack (на стороне consumer),
> DLX для упавших сообщений, outbox pattern для гарантии записи + отправки.»