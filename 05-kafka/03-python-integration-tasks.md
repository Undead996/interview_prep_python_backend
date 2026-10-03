# Kafka + Python: интеграция и задачи (расширенно)

---

## aiokafka — async producer/consumer

```python
from aiokafka import AIOKafkaProducer, AIOKafkaConsumer

# Producer
async def produce():
    producer = AIOKafkaProducer(
        bootstrap_servers="localhost:9092",
        enable_idempotence=True,
    )
    await producer.start()
    try:
        # Одно сообщение
        await producer.send("orders", b"data")
        # С ключом (partition by user_id)
        await producer.send("orders", key=b"user_42", value=b"data")
        # Бачч
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
            await consumer.commit()
    finally:
        await consumer.stop()
```

---

## FastAPI + Kafka

```python
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

> **На собесе:** «Как сделать retry в Kafka?» —  
> «1) При ошибке отправляем в retry-топик с header retry_count.
> 2) После N попыток — в DLQ (dead letter queue).
> 3) Offset при этом коммитим — не блокируем чтение новых сообщений.
> В RabbitMQ — DLX, в Kafka — DLQ через отдельный topic.»