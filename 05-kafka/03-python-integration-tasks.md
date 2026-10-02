# Kafka + Python: интеграция и задачи

---

## aiokafka (async/await)

```python
import asyncio
from aiokafka import AIOKafkaProducer, AIOKafkaConsumer

# Producer
async def produce():
    producer = AIOKafkaProducer(
        bootstrap_servers="localhost:9092",
        enable_idempotence=True,
    )
    await producer.start()
    try:
        # send (одно сообщение)
        await producer.send("orders", b"order data")
        # send with key (partition by key)
        await producer.send("orders", key=b"user_42", value=b"order data")
        # batch send
        await producer.send_batch("orders", [b"msg1", b"msg2"])
    finally:
        await producer.stop()


# Consumer
async def consume():
    consumer = AIOKafkaConsumer(
        "orders",
        bootstrap_servers="localhost:9092",
        group_id="order-processor",
        enable_auto_commit=False,
    )
    await consumer.start()
    try:
        async for msg in consumer:
            print(f"Offset: {msg.offset}, Key: {msg.key}, Value: {msg.value}")
            await consumer.commit()  # ручной commit
    finally:
        await consumer.stop()
```

---

## confluent-kafka (синхронный, C-драйвер)

```python
from confluent_kafka import Producer, Consumer, KafkaError

# Producer
producer = Producer({"bootstrap.servers": "localhost:9092"})

def delivery_report(err, msg):
    if err is not None:
        print(f"Delivery failed: {err}")
    else:
        print(f"Delivered to {msg.topic()} [{msg.partition()}]")

producer.produce("orders", b"data", callback=delivery_report)
producer.flush()  # ждать отправки


# Consumer
consumer = Consumer({
    "bootstrap.servers": "localhost:9092",
    "group.id": "order-processor",
    "auto.offset.reset": "earliest",
})

consumer.subscribe(["orders"])
while True:
    msg = consumer.poll(timeout=1.0)
    if msg is None:
        continue
    if msg.error():
        print(f"Error: {msg.error()}")
        continue
    print(f"Received: {msg.value()}")
    consumer.commit(msg)
```

---

## FastAPI + Kafka (aiokafka)

```python
from fastapi import FastAPI
from contextlib import asynccontextmanager
from aiokafka import AIOKafkaProducer

@asynccontextmanager
async def lifespan(app):
    app.state.kafka = AIOKafkaProducer(bootstrap_servers="localhost:9092")
    await app.state.kafka.start()
    yield
    await app.state.kafka.stop()

app = FastAPI(lifespan=lifespan)

@app.post("/api/orders")
async def create_order(payload: OrderCreate):
    order = await create_order_in_db(payload)

    await app.state.kafka.send(
        "orders",
        key=str(order.user_id).encode(),
        value=order.model_dump_json().encode(),
    )
    return order
```

---

## Задача 1. Consumer retry с отложенной retry-очередью

**Условие:** Если обработка упала, сообщение должно попасть в retry-топик.
Попробовать 3 раза, потом — в dead-letter.

<details>
<summary>Решение</summary>

```python
MAX_RETRIES = 3

async def process_message(msg):
    try:
        await handle(msg.value)
        await consumer.commit()
    except Exception:
        retry_count = msg.headers.get("retry_count", 0)
        if retry_count < MAX_RETRIES:
            # Отправить в retry-топик
            await producer.send(
                "orders-retry",
                value=msg.value,
                headers={"retry_count": retry_count + 1, "original_offset": msg.offset},
            )
            await consumer.commit()  # сдвинуть offset
        else:
            # Dead letter
            await producer.send("orders-dlq", value=msg.value)
            await consumer.commit()
```
</details>

---

## Задача 2. Определите partition по user_id

**Условие:** Как сделать, чтобы все заказы пользователя были в одной partition?

<details>
<summary>Решение</summary>

```python
key = str(user_id).encode()
# Kafka использует hash(key) для выбора partition
# Один user_id → одна partition → порядок операций!

await producer.send("orders", key=key, value=order_data)

# Consumer — достаточно указать group.id, Kafka распределит partitions
# Если нужно читать partition напрямую:
consumer.assign([PartitionInfo(topic="orders", partition=0, offset=0)])
```
</details>

---

> **На собесе:** «Kafka vs RabbitMQ — что выберете для нотификаций?» —
> «Если нужно, чтобы нотификация была доставлена один раз и сразу — RabbitMQ.
> Если нужно, чтобы её можно было прочитать исторически, перечитать, и могло
> быть много consumer-групп — Kafka. Для email-нотификаций — RabbitMQ.
> Для event sourcing — Kafka.»