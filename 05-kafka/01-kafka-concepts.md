# Apache Kafka — концепции (расширенно)

> **Цель:** понять, как устроен Kafka "снизу": почему он быстрый, чем отличается
> от RabbitMQ, когда выбирать.

---

## 1. Что такое Kafka на уровне данных

Когда producer пишет в Kafka:

```
Producer → Topic "orders"
              └── Partition 0: [msg0, msg1, msg2, ...]  → append-only log
              └── Partition 1: [msg0, msg1, ...]         → append-only log
```

**Kafka — это распределённый commit log.** Каждое сообщение **не удаляется** после прочтения. Оно хранится N дней (настраивается).

**Отличие от RabbitMQ:** в RabbitMQ сообщение исчезает после ack. В Kafka оно остаётся — его могут прочитать другие consumers или перечитать те же.

---

## 2. Partition — единица параллелизма

```python
# Producer выбирает partition
# 1. Round-robin (равномерно)
await producer.send("orders", value=b"data")

# 2. Key-hash (один key → одна partition)
await producer.send("orders", key=b"user_42", value=b"data")
# Все заказы user_42 — в одной partition → строгий порядок!
```

**Следствие:** один consumer может читать одну partition. Если вам нужно 10 параллельных consumers — нужно 10 partition.

---

## 3. Consumer Group — как consumers делят partitions

```
Topic "orders" (6 partitions)
  Consumer Group "processors":
    Consumer A: partitions [0, 1]
    Consumer B: partitions [2, 3]
    Consumer C: partitions [4, 5]
```

Если Consumer B упал → ребаланс: A получает [0, 1, 2], C получает [3, 4, 5].

**Во время ребаланса сообщения НЕ обрабатываются.** Минимизируйте ребалансы:
- Стабильный `group.id`
- `session.timeout.ms` — не слишком маленький
- `CooperativeStickyAssignor` (Kafka 2.4+) — пошаговый ребаланс

---

## 4. Kafka vs RabbitMQ — детально

| Характеристика | RabbitMQ | Kafka | Почему |
|---------------|---------|-------|--------|
| Модель | Smart broker | Smart consumer | RMQ роутит, Kafka хранит |
| Хранение | После ack — удалено | Хранится N дней | Kafka — лог событий |
| Порядок | В одной очереди | В одной partition | Оба упорядочены |
| Throughput | ~50K msg/s | ~1M msg/s | Kafka батчит |
| Ретраи | Через DLX | Просто читать заново (offset) | Kafka удобнее для replay |
| RPC | ✅ (reply-to) | ❌ | Kafka не для синхронных вызовов |
| Типичное | Задачи, уведомления | Event sourcing, Big Data | — |

**Когда выбирать:**
```
Сообщение одному → RabbitMQ
Сообщение многим → Kafka
История не нужна → RabbitMQ
История нужна → Kafka
RPC нужен → RabbitMQ
High throughput → Kafka
```

---

> **На собесе:** «Kafka vs RabbitMQ — что выберете для нотификаций?» —  
> «Если нотификацию нужно доставить один раз и забыть — RabbitMQ.
> Если нужно хранить историю, перечитывать, несколько групп — Kafka.
> Для email-нотификаций обычно RabbitMQ. Для event sourcing — Kafka.»