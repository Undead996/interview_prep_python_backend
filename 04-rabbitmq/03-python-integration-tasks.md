# RabbitMQ + Python: интеграция и задачи

> **Цель:** написать producer/consumer на aio-pika, интегрировать с FastAPI, решить 3 задачи.

---

## 1. aio-pika — async RabbitMQ клиент

```python
import aio_pika
import asyncio

# === Producer ===
async def publish_order(order_id: str, payload: bytes):
    connection = await aio_pika.connect_robust("amqp://guest:guest@localhost/")
    async with connection:
        channel = await connection.channel()
        # Объявить exchange и очередь:
        exchange = await channel.declare_exchange("orders", aio_pika.ExchangeType.TOPIC, durable=True)
        queue = await channel.declare_queue("orders.created", durable=True)
        await queue.bind(exchange, routing_key="order.created")

        await exchange.publish(
            aio_pika.Message(
                body=payload,
                delivery_mode=aio_pika.DeliveryMode.PERSISTENT,
                message_id=order_id,
                headers={"source": "fastapi"},
            ),
            routing_key="order.created",
        )

# === Consumer ===
async def consume_orders():
    connection = await aio_pika.connect_robust("amqp://guest:guest@localhost/")
    async with connection:
        channel = await connection.channel()
        await channel.set_qos(prefetch_count=10)  # не более 10 unacked

        queue = await channel.declare_queue("orders.created", durable=True)
        async with queue.iterator() as queue_iter:
            async for message in queue_iter:
                async with message.process():  # auto ack/nack
                    print(f"Received: {message.body.decode()}")
                    await asyncio.sleep(0.1)  # имитация обработки
```

---

## 2. FastAPI + RabbitMQ (lifespan)

```python
from contextlib import asynccontextmanager
import aio_pika

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    app.state.rmq_connection = await aio_pika.connect_robust(
        f"amqp://{RABBITMQ_USER}:{RABBITMQ_PASS}@{RABBITMQ_HOST}/"
    )
    app.state.rmq_channel = await app.state.rmq_connection.channel()
    app.state.rmq_exchange = await app.state.rmq_channel.declare_exchange(
        "notifications", aio_pika.ExchangeType.TOPIC, durable=True
    )
    logger.info("RabbitMQ connected")

    yield

    # Shutdown
    await app.state.rmq_connection.close()
    logger.info("RabbitMQ disconnected")

app = FastAPI(lifespan=lifespan)

@app.post("/api/orders")
async def create_order(
    payload: OrderCreate,
    db: AsyncSession = Depends(get_db),
):
    order = await create_order_in_db(db, payload)

    exchange: aio_pika.Exchange = request.app.state.rmq_exchange
    await exchange.publish(
        aio_pika.Message(
            body=order.model_dump_json().encode(),
            delivery_mode=aio_pika.DeliveryMode.PERSISTENT,
            message_id=str(order.id),
        ),
        routing_key="order.created",
    )
    return order
```

---

## Задача 1. Consumer с retry и DLX

```python
MAX_RETRIES = 3

async def callback(message: aio_pika.IncomingMessage):
    async with message.process():
        try:
            retry_count = int(message.headers.get("x-retry-count", 0))
            await process(message.body)
        except TemporaryError:
            if retry_count < MAX_RETRIES:
                # republic с увеличенным retries
                await message.channel.default_exchange.publish(
                    aio_pika.Message(
                        body=message.body,
                        headers={"x-retry-count": retry_count + 1},
                        expiration=str(2 ** retry_count * 1000),  # ms
                    ),
                    routing_key="orders.retry",
                )
                await message.ack()  # подтверждаем оригинал
            else:
                await message.reject(requeue=False)  # → DLX
        except PermanentError:
            await message.reject(requeue=False)  # → DLX сразу
```

---

## Задача 2. Outbox Poller (полная)

См. `02-reliability-and-patterns.md` — раздел Outbox Pattern с полным SQL + Python кодом.

---

## Задача 3. RPC через RabbitMQ (request-reply)

```python
import uuid

async def rpc_call(method: str, params: dict, timeout: float = 5.0) -> dict:
    connection = await aio_pika.connect_robust("amqp://guest:guest@localhost/")
    async with connection:
        channel = await connection.channel()
        # Временная очередь для ответа
        callback_queue = await channel.declare_queue(exclusive=True, auto_delete=True)

        correlation_id = str(uuid.uuid4())
        future: asyncio.Future = asyncio.get_running_loop().create_future()

        async def on_response(message: aio_pika.IncomingMessage):
            if message.correlation_id == correlation_id:
                future.set_result(message.body)

        await callback_queue.consume(on_response)

        await channel.default_exchange.publish(
            aio_pika.Message(
                body=json.dumps({"method": method, "params": params}).encode(),
                correlation_id=correlation_id,
                reply_to=callback_queue.name,
            ),
            routing_key="rpc_queue",
        )

        return json.loads(await asyncio.wait_for(future, timeout=timeout))
```

---

> **На собесе:** «RabbitMQ — как обеспечить надёжность?» —
> «Publisher confirms на producer, consumer ack, persistent messages + durable queues. Outbox pattern для атомарной записи. DLX + retry c exponential backoff. Quorum queues для критичных данных.»