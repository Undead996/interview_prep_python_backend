# Kafka: Надёжность и потребление — guarantees, offset, rebalancing

> **Цель:** понимать уровни гарантий, управлять offset, минимизировать ребалансы.

---

## 1. Гарантии доставки — как настроить

| Гарантия | acks | enable_idempotence | max.in.flight | Риск |
|----------|------|-------------------|---------------|------|
| **At-most-once** | `0` | ❌ | любое | Потеря сообщений |
| **At-least-once** | `1` | ❌ | любое | Дубликаты |
| **At-least-once (all)** | `all` | ❌ | ≤ 5 | Дубликаты при сбое |
| **Exactly-once** | `all` | ✅ | ≤ 5 | — (лучшая гарантия) |

### Exactly-once producer: как работает

```python
producer = AIOKafkaProducer(
    bootstrap_servers="localhost:9092",
    enable_idempotence=True,    # дедупликация по ProducerID + sequence_number
    acks="all",                 # ждать подтверждения от всех in-sync реплик
    max_in_flight_requests_per_connection=5,  # не более 5 сообщений без confirms
)
```

**Как работает idempotent producer:**
1. Kafka присваивает каждому producer уникальный ProducerID (PID)
2. Каждое сообщение нумеруется sequence_number (0, 1, 2...)
3. Брокер проверяет: если сообщение с таким PID:seq уже записано — игнорирует дубликат
4. При перезапуске producer получает новый PID — дубликаты возможны на границе перезапуска

**Exactly-once семантика:** idempotent producer + транзакции Kafka + consumer с `isolation.level=read_committed`.

---

## 2. Offset Management

```python
consumer = AIOKafkaConsumer(
    "orders",
    bootstrap_servers="localhost:9092",
    group_id="order-processor",
    enable_auto_commit=False,        # manual commit! (рекомендуется)
    auto_offset_reset="earliest",    # earliest | latest | none
)
```

### Auto-commit vs Manual

| Режим | Поведение | Риск |
|-------|----------|------|
| **Auto-commit** (`enable_auto_commit=True`) | Offset коммитится каждые N секунд (`auto_commit_interval_ms`) | Сообщение обработано, но offset не успел закоммититься → после перезапуска прочитается заново (дубликат). Или offset закоммитился, но сообщение не обработано → потеря. |
| **Manual commit** | Commit вызывается явно после обработки | At-least-once: обработали → commit. Если упали до commit → повторим (идемпотентность consumer обязательна). |

```python
async for msg in consumer:
    try:
        await process(msg.value)      # сначала обработать
        await consumer.commit()       # потом закоммитить
    except Exception:
        # отправить в retry topic (не блокировать основную очередь!)
        await retry_producer.send("orders.retry", value=msg.value)
        await consumer.commit()       # коммитим даже при ошибке!
```

**Почему commit даже при ошибке?** Чтобы не блокировать partition. Упавшее сообщение уходит в retry-topic и обрабатывается асинхронно.

### auto.offset.reset

```
earliest — новый consumer читает с ПЕРВОГО сообщения в partition (история)
latest   — новый consumer читает только НОВЫЕ сообщения
none     — если offset не найден — ошибка
```

---

## 3. Rebalancing — что происходит и как минимизировать

```
Триггеры ребаланса:
1. Consumer покинул группу (упал, остановлен)
2. Новый consumer присоединился
3. Добавлена новая partition в topic
4. Истек session.timeout.ms (consumer не отправил heartbeat)

Что происходит:
1. Все consumer-ы получают Revoke событие → прекращают чтение
2. Перераспределение partitions между оставшимися consumer-ами
3. Consumer-ы получают Assign событие → начинают чтение с последнего committed offset

Во время ребаланса: ПОТРЕБЛЕНИЕ ОСТАНОВЛЕНО
```

**Минимизация ребалансов:**
```python
consumer = AIOKafkaConsumer(
    "orders",
    group_id="processor",
    session_timeout_ms=30000,        # не слишком маленький
    heartbeat_interval_ms=10000,     # heartbeat = session_timeout / 3
    max_poll_interval_ms=300000,     # макс. время между poll (для долгой обработки)
    partition_assignment_strategy=[  # Cooperative (Kafka 2.4+)
        CooperativeStickyAssignor,
    ],
)
```

**CooperativeStickyAssignor:** при ребалансе снимает **только те partitions, которые нужно переместить**, остальные продолжают потребляться. В отличие от Range/RoundRobin, где снимаются ВСЕ partitions.

---

## 4. Конфигурация producer/consumer

```python
# Producer — оптимальные настройки:
producer = AIOKafkaProducer(
    bootstrap_servers="localhost:9092",
    acks="all",
    enable_idempotence=True,
    compression_type="snappy",       # lz4 — быстрее, snappy — баланс, gzip — компактнее
    linger_ms=5,                     # ждать 5ms чтобы набрать батч
    batch_size=16384,                # макс. размер батча в байтах
    max_in_flight_requests_per_connection=5,
    request_timeout_ms=30000,
)

# Consumer — оптимальные настройки:
consumer = AIOKafkaConsumer(
    "orders",
    bootstrap_servers="localhost:9092",
    group_id="processor",
    enable_auto_commit=False,
    auto_offset_reset="earliest",
    max_poll_records=500,            # до 500 записей за poll
    fetch_min_bytes=1048576,         # ждать >= 1MB перед fetch
    fetch_max_wait_ms=500,           # но не дольше 500ms
    session_timeout_ms=30000,
    heartbeat_interval_ms=10000,
    max_poll_interval_ms=300000,
)
```

---

## 5. Мониторинг (CLI)

```bash
# Список топиков
kafka-topics --bootstrap-server localhost:9092 --list

# Детали топика
kafka-topics --bootstrap-server localhost:9092 --describe --topic orders

# Consumer groups
kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group processor
# LAG — сколько сообщений ещё не прочитано

# Офсеты
kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group processor --offsets

# Чтение (для отладки)
kafka-console-consumer --bootstrap-server localhost:9092 --topic orders --from-beginning
```

---

> **Технически:** Exactly-once = idempotent producer (ProducerID + sequence number) + `acks=all` + consumer `isolation.level=read_committed`. At-least-once = `acks=1`/`all` + manual commit. At-most-once = `acks=0` или auto-commit до обработки.