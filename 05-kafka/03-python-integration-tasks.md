# Kafka + Python: интеграция и задачи

> **Цель:** написать producer/consumer на aiokafka, FastAPI-интеграцию, 3 задачи.

---

## 1. aiokafka — async producer/consumer

```python
from aiokafka import AIOKafkaProducer, AIOKafkaConsumer

# === Producer ===
async def produce():
    producer = AIOKafkaProducer(
        bootstrap_servers="localhost:9092",
        enable_idempotence=True,
        acks="all",
        compression_type="snappy",
    )
    await producer.start()
    try:
        # Одно сообщение:
        await producer.send("orders", value=b"data")
        # С ключом (гарантия порядка для user_42):
        await producer.send("orders", key=b"user_42", value=b"order_data")
        # Батч:
        batch = producer.create_batch()
        batch.append(key=b"k1", value=b"v1", timestamp=None)
        batch.append(key=b"k2", value=b"v2", timestamp=None)
        await producer.send_batch(batch, "orders")
    finally:
        await producer.stop()

# === Consumer ===
async def consume():
    consumer = AIOKafkaConsumer(
        "orders",
        bootstrap_servers="localhost:9092",
        group_id="processor",
        enable_auto_commit=False,
        auto_offset_reset="earliest",
    )
    await consumer.start()
    try:
        async for msg in consumer:
            print(f"Partition={msg.partition}, Offset={msg.offset}, Key={msg.key}")
            await process(msg.value)
            await consumer.commit()
    finally:
        await consumer.stop()
```

---

## 2. FastAPI + Kafka

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    app.state.kafka_producer = AIOKafkaProducer(
        bootstrap_servers=KAFKA_BOOTSTRAP,
        enable_idempotence=True,
        acks="all",
    )
    await app.state.kafka_producer.start()
    logger.info("Kafka producer started")
    yield
    # Shutdown
    await app.state.kafka_producer.stop()
    logger.info("Kafka producer stopped")

app = FastAPI(lifespan=lifespan)

@app.post("/api/orders")
async def create_order(payload: OrderCreate, db: AsyncSession = Depends(get_db)):
    order = await create_order_in_db(db, payload)

    producer: AIOKafkaProducer = request.app.state.kafka_producer
    await producer.send(
        "orders",
        key=str(order.user_id).encode(),
        value=order.model_dump_json().encode(),
    )
    return order
```

---

## Задача 1. Consumer с retry + DLQ

```python
MAX_RETRIES = 3
RETRY_TOPIC = "orders-retry"
DLQ_TOPIC = "orders-dlq"

async def process_with_retry():
    consumer = AIOKafkaConsumer("orders", ...)
    producer = AIOKafkaProducer(...)
    await consumer.start(); await producer.start()

    try:
        async for msg in consumer:
            retries = int(msg.headers.get("x-retries", "0"))
            try:
                await handle(msg.value)
                await consumer.commit()
            except TemporaryError:
                if retries < MAX_RETRIES:
                    await producer.send(
                        RETRY_TOPIC,
                        value=msg.value,
                        headers=[("x-retries", str(retries + 1).encode())],
                    )
                else:
                    await producer.send(DLQ_TOPIC, value=msg.value)
                await consumer.commit()  # коммитим даже при ошибке!
            except PermanentError:
                await producer.send(DLQ_TOPIC, value=msg.value)
                await consumer.commit()
    finally:
        await consumer.stop(); await producer.stop()
```

---

## Задача 2. Idempotent consumer (дедупликация через Redis)

```python
async def process_idempotent(msg, redis: Redis):
    # Используем topic + partition + offset как уникальный ключ
    dedup_key = f"dedup:{msg.topic}:{msg.partition}:{msg.offset}"
    if await redis.exists(dedup_key):
        return  # уже обработано
    await process(msg.value)
    await redis.setex(dedup_key, 86400, "1")  # храним 24 часа
    await consumer.commit()
```

---

## Задача 3. Гарантированный порядок обработки

```python
# Ключ = user_id → все сообщения одного пользователя в одной partition
await producer.send("orders", key=str(user_id).encode(), value=payload)

# Consumer обрабатывает строго последовательно (одна partition — один consumer)
async for msg in consumer:
    await process_sequentially(msg.value)  # порядок гарантирован
    await consumer.commit()
```

---

> **На собесе:** «Как сделать retry в Kafka?» —
> «При ошибке отправляем в retry-топик с x-retries header. После N попыток — DLQ. Offset коммитим всегда — не блокируем partition. Идемпотентность через Redis (topic:partition:offset как ключ). Для exactly-once: idempotent producer + consumer с manual commit + транзакции при записи в БД.»