# Шпаргалка: Apache Kafka

---

## CLI

```bash
kafka-topics --bootstrap-server localhost:9092 --list
kafka-topics --bootstrap-server localhost:9092 --describe --topic orders
kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group processor
kafka-console-producer --broker-list localhost:9092 --topic orders
kafka-console-consumer --bootstrap-server localhost:9092 --topic orders --from-beginning
```

## Концепции

```
Topic → Partition (append-only log) → Consumer Group (offset)

Producer → key hash → partition (ordered for same key)
Consumer → group.id → partitions распределяются между consumers

auto.offset.reset = earliest | latest | none
enable_auto_commit = true | false  (лучше false)
```

## aiokafka

```python
# Producer
producer = AIOKafkaProducer(bootstrap_servers="localhost:9092", enable_idempotence=True)
await producer.start()
await producer.send("topic", key=b"k", value=b"v")
await producer.stop()

# Consumer
consumer = AIOKafkaConsumer("topic", bootstrap_servers="localhost:9092", group_id="g")
await consumer.start()
async for msg in consumer:
    await consumer.commit()
await consumer.stop()
```

## Гарантии

```
acks=0            — at-most-once
acks=1 (default)   — at-least-once (broker)
acks=all           — at-least-once (all replicas)
enable_idempotence — exactly-once
```

## Kafka vs RabbitMQ

```
Kafka:   event sourcing, replay, high throughput, multiple consumers
RabbitMQ: RPC, ack tasks, low latency, one consumer
```