# RabbitMQ Cheatsheet

## CLI

```bash
rabbitmqctl list_queues name messages messages_ready messages_unacknowledged
rabbitmqctl list_exchanges name type
rabbitmqctl list_bindings
rabbitmq-diagnostics status
rabbitmq-diagnostics check_alarms
```

## AMQP-модель

```
Producer → Exchange → (binding=routing_key) → Queue → Consumer (ack)
Exchange: direct | fanout | topic | headers
```

## aio-pika

```python
conn = await aio_pika.connect_robust("amqp://guest:guest@localhost/")
channel = await conn.channel()
# Producer:
await channel.default_exchange.publish(
    aio_pika.Message(body=b"data", delivery_mode=aio_pika.DeliveryMode.PERSISTENT),
    routing_key="queue.name")
# Consumer:
queue = await channel.declare_queue("queue.name", durable=True)
async with queue.iterator() as qiter:
    async for msg in qiter:
        async with msg.process():  # auto ack/reject
            await process(msg.body)
```

## Ack / Nack / Reject

```python
msg.ack()                           # успех → удалить
msg.nack(requeue=True)             # временная ошибка → вернуть
msg.reject(requeue=False)          # фатальная → DLX
```

## Гарантии

```
Publisher confirm → producer-уровень
Consumer ack     → consumer-уровень
DLX              → dead letter при reject/expiry
Outbox pattern   → атомарная БД + очередь
Quorum queue     → RAFT-консенсус
```

## Настройка DLX

```python
channel.queue_declare("orders", arguments={
    "x-dead-letter-exchange": "orders.dlx",
    "x-message-ttl": 86400000,  # 24h
})
```

## FastAPI lifespan

```python
async def lifespan(app):
    app.state.rmq = await aio_pika.connect_robust(URL)
    app.state.channel = await app.state.rmq.channel()
    yield
    await app.state.rmq.close()
```