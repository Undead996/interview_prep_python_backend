# Apache Kafka — концепции: почему быстрый и когда выбирать

> **Цель:** понять внутреннее устройство Kafka, чем отличается от RabbitMQ, когда выбирать каждую.

---

## 1. Что такое Kafka на уровне данных

**Простыми словами:** Kafka — это распределённый бесконечный журнал (commit log). В отличие от RabbitMQ, где сообщение исчезает после ack, в Kafka оно остаётся на N дней — его могут перечитать другие consumer-ы или даже те же самые.

```
Producer → Topic "orders" (3 partitions)
              ├── Partition 0: [msg0, msg1, msg2, ...]  ← append-only log
              ├── Partition 1: [msg0, msg1, ...]
              └── Partition 2: [msg0, msg1, msg2, msg3, ...]

Consumer Group "processors" (2 consumers)
              Consumer A: partitions [0, 2]
              Consumer B: partitions [1]
```

**Ключевые идеи:**
- Partition — физический append-only лог. Сообщения упорядочены **внутри** partition.
- Consumer Group — логическая группа consumer-ов, которые делят partitions между собой.
- Offset — позиция consumer в partition (сколько сообщений уже прочитано).
- Сообщение не удаляется — хранится N дней (retention) или пока не превышен размер.

---

## 2. Partition — единица параллелизма

```python
# Producer выбирает partition:
# 1. Без ключа — round-robin:
await producer.send("orders", value=b"data")

# 2. С ключом — hash(key) % num_partitions:
await producer.send("orders", key=b"user_42", value=b"data")
# ВСЕ сообщения user_42 → в одну partition → СТРОГИЙ порядок!
```

**Следствие:** если нужно 10 параллельных consumer'ов — нужно минимум 10 partitions. Один consumer может читать несколько partitions, но одна partition — только один consumer в группе.

---

## 3. Consumer Group — как consumers делят partitions

```
Topic "orders" (6 partitions)
Consumer Group "processors":
  Consumer A: partitions [0, 1]
  Consumer B: partitions [2, 3]
  Consumer C: partitions [4, 5]

Consumer B упал → Rebalance:
  Consumer A: [0, 1, 2, 3]
  Consumer C: [4, 5]
```

**Rebalance — дорогая операция:** во время ребаланса потребление **останавливается**. Минимизируйте:

1. Стабильный `group.id`
2. `session.timeout.ms` — не слишком маленький (10-30s)
3. `CooperativeStickyAssignor` (Kafka 2.4+) — пошаговый ребаланс (не снимает все partitions сразу)

```python
consumer = AIOKafkaConsumer(
    "orders",
    group_id="processor",
    session_timeout_ms=30000,
    heartbeat_interval_ms=10000,
    partition_assignment_strategy=[CooperativeStickyAssignor],
)
```

---

## 4. Kafka vs RabbitMQ — детальная таблица

| Характеристика | RabbitMQ | Kafka |
|---------------|---------|-------|
| **Модель** | Smart broker, dumb consumer | Dumb broker, smart consumer |
| **Хранение** | После ack — удалено | Хранится N дней (retention) |
| **Порядок сообщений** | В одной очереди | В одной partition |
| **Чтение** | Push (доставка consumer) / Pull | Pull (consumer запрашивает) |
| **Пропускная способность** | ~50K msg/s | ~1M+ msg/s (батчинг + zero-copy) |
| **Ретраи** | Через DLX (встроено) | Ручные (retry topic + DLQ) |
| **RPC** | ✅ (reply-to) | ❌ (не предназначен) |
| **Множество consumer-групп** | ❌ (нужны разные очереди) | ✅ (читают независимо, свои offset) |
| **Replay** | ❌ (сообщение уже удалено) | ✅ (offset можно сбросить) |
| **Latency** | Низкая (~1ms) | Низкая (~5ms) |
| **Сложность развёртывания** | Простая (1 узел) | Средняя (ZooKeeper/RAFT) |
| **Zero-copy** | ❌ | ✅ (sendfile syscall) |
| **Compaction** | ❌ | ✅ (key compaction: хранить только последнее значение) |
| **Типичное применение** | Задачи, уведомления, RPC | Event sourcing, стриминг, аналитика |

### Алгоритм выбора

```
Задача → один consumer, ack, без истории?         → RabbitMQ
Задача → много consumer-групп, replay, историю?   → Kafka
Нужен RPC (request-reply)?                         → RabbitMQ
Нужен high throughput (> 100K msg/s)?              → Kafka
Нужно хранить события годами?                      → Kafka
Простая интеграция, минимум инфраструктуры?        → RabbitMQ
```

---

## 5. Почему Kafka быстрый

1. **Append-only log** — запись всегда в конец файла (sequential write, ~600 MB/s на HDD)
2. **Zero-copy** — данные с диска → сетевой буфер без копирования в userspace (sendfile syscall)
3. **Батчинг** — producer собирает сообщения в батчи, отправляет одним запросом
4. **Сжатие** — gzip/snappy/lz4/zstd на уровне топика, распаковывается consumer
5. **Слабая зависимость от размера сообщения** — throughput определяется не размером, а количеством батчей

---

> **На собесе:** «Kafka vs RabbitMQ — что для нотификаций?» —
> «Если нужно доставить email/sms/push один раз и забыть — RabbitMQ (проще, ack, DLX). Если нужен аудит всех нотификаций, перечитывание, несколько consumer-групп — Kafka. Для email-нотификаций обычно RabbitMQ. Для event sourcing (перестройка состояния из истории) — только Kafka.»