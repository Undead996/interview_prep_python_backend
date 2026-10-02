# Шпаргалка: RabbitMQ

---

## CLI

```bash
rabbitmqctl list_queues                   # размер, consumer count
rabbitmqctl list_exchanges                # все exchanges
rabbitmqctl list_bindings                 # привязки
rabbitmqctl status                        # здоровье
```

## Концепции

```
Producer → Exchange → (binding) → Queue → Consumer

Exchange: direct / fanout / topic / headers
Queue: durable, auto_delete, arguments (x-message-ttl, x-dead-letter-exchange)
```

## aio-pika

```python
connection = await aio_pika.connect_robust("amqp://guest:guest@localhost/")
channel = await connection.channel()
await channel.default_exchange.publish(
    aio_pika.Message(body=b"hello", delivery_mode=aio_pika.DeliveryMode.PERSISTENT),
    routing_key="queue.name",
)
```

## Гарантии

```
Publisher confirm  → потеря на стороне producer
Consumer ack       → потеря на стороне consumer
DLX                → dead letter при reject
Outbox pattern     → гарантия записи + отправки
```

## Ack / Nack / Reject

```python
ch.basic_ack(delivery_tag=method.delivery_tag)      # успех
ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)   # retry
ch.basic_reject(delivery_tag=method.delivery_tag, requeue=False) # → DLX
```