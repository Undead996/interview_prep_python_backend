# Apache Kafka Cheatsheet

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
Topic → Partition (append-only) → Consumer Group → Offset
Producer: key-hash → partition (порядок) | round-robin (равномерно)
Consumer: group.id → partitions делятся между consumers
Rebalance: один consumer = одна или несколько partitions
```

## aiokafka

```python
# Producer
producer = AIOKafkaProducer(
    bootstrap_servers="localhost:9092",
    enable_idempotence=True, acks="all",
    compression_type="snappy")
await producer.start()
await producer.send("orders", key=b"k", value=b"v")
await producer.stop()

# Consumer
consumer = AIOKafkaConsumer(
    "orders", bootstrap_servers="localhost:9092",
    group_id="g", enable_auto_commit=False,
    auto_offset_reset="earliest")
await consumer.start()
async for msg in consumer:
    await process(msg.value)
    await consumer.commit()
await consumer.stop()
```

## Гарантии

```
acks=0            — at-most-once (потеря)
acks=1 (default)   — at-least-once
acks=all           — at-least-once (все реплики)
enable_idempotence — exactly-once (PID + seq_number)
```

## Offset

```python
enable_auto_commit=False  # manual commit (рекомендуется)
auto_offset_reset="earliest"  # новый consumer → с начала
auto_offset_reset="latest"    # новый consumer → только новые
```

## Kafka vs RabbitMQ

```
Kafka:   event sourcing, replay, high throughput, multiple consumers
RMQ:     задачи, RPC, уведомления, ack, низкая latency
```