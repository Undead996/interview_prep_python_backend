# Apache Kafka — концепции

---

## Что такое Kafka

> **Простыми словами:** Kafka — это «склад» сообщений. Producer пишет в конец лога,
> consumer читает с того места, которое ему интересно. В отличие от RabbitMQ,
> сообщение не исчезает после прочтения — его могут прочитать несколько раз
> разные consumers.

```
 Producer ──→ ┌──────────────────────┐
               │   Partition 0        │
               │ ┌────┬────┬────┬────┐│ ← Topic
               │ │msg1│msg2│msg3│msg4││
               │ └────┴────┴────┴────┘│
               └──────────────────────┘
                          ↓
 Consumer Group (offset = 2)
```

---

## Основные концепции

| Понятие | Описание |
|---|---|
| **Topic** | Категория сообщений (как таблица) |
| **Partition** | Физический сегмент сообщений (append-only log) |
| **Producer** | Пишет сообщения в topic |
| **Consumer** | Читает сообщения из topic |
| **Consumer Group** | Группа consumers, делящая partitions |
| **Offset** | Позиция consumer в partition (номер сообщения) |
| **Broker** | Сервер Kafka |
| **Zookeeper / KRaft** | Координация кластера |

---

## Topics и Partitions

```
Topic "orders"
 ├── Partition 0 (leader на broker-1)
 │    msg[0], msg[1], msg[2], msg[3], ...
 ├── Partition 1 (leader на broker-2)
 │    msg[0], msg[1], msg[2], ...
 └── Partition 2 (leader на broker-3)
      msg[0], msg[1], ...

Producer выбирает partition по:
- round-robin (равномерно)
- key hash (user_id → partition 0 — порядок для пользователя)
```

---

## Kafka vs RabbitMQ

| Характеристика | RabbitMQ | Kafka |
|---|---|---|
| **Модель** | Smart broker, dumb consumer | Dumb broker, smart consumer |
| **Хранение** | После ack — удалено | Хранится N дней (configurable) |
| **Порядок** | В одной очереди | В одной partition |
| **Скорость** | ~50K msg/s | ~1M msg/s |
| **Ретраи** | Через DLX | Позиция offset — просто читать заново |
| **RPC** | ✅ (reply-to) | ❌ (не предназначен) |
| **Рефакторинг** | Легко (очереди) | Сложно (нужны новые topics) |
| **Типичное** | Задачи, RPC, синхронные потоки | Event sourcing, стримы, Big Data |

---

## Когда выбирать Kafka, когда RabbitMQ

```
Нужно:
- Сообщение достанется одному потребителю  → RabbitMQ
- Сообщение прочитают несколько раз        → Kafka
- Нужно перечитать историю                 → Kafka
- Низкая latency + RPC                     → RabbitMQ
- Высокий throughput (1M+/с)               → Kafka
- Event sourcing / CQRS                    → Kafka
```

---

## Подводные камни

| ❌ Ошибка | ✅ Правильно |
|---|---|
| `auto.offset.reset=latest` (default) | Новый consumer не читает существующие сообщения |
| Одна partition = один consumer | Параллелизм только через больше partitions |
| Нет мониторинга rebalancing | Consumer group ребалансится → остановка |
| Сообщение > 1MB | По умолчанию max 1MB — меняется в конфиге |

---

> **Технически:** Kafka — распределенный, аппендикс-лог. Сообщение хранится по
> умолчанию 7 дней или до 1GB на partition (настраивается). Partition — фундаментальная
> единица параллелизма. Consumer group — много consumers делят partitions.
> Offset — «закладка» consumer в логе.