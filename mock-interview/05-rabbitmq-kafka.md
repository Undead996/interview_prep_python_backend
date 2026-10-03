# Мок-интервью: Очереди (RabbitMQ + Kafka, 20 вопросов) — расширенные ответы

---

## Q1. RabbitMQ vs Kafka — что выберете и почему?

```
✅ Ответ:
RabbitMQ — когда:
- Сообщение нужно доставить одному consumer
- Нужен RPC (reply-to)
- Нужна сложная маршрутизация (topic exchange)
- Низкая latency

Kafka — когда:
- Сообщение прочитают несколько consumer-групп
- Нужно перечитывать историю
- Высокий throughput (> 100K msg/s)
- Event sourcing / CQRS

Типичные сценарии: нотификации → RMQ, аудит → Kafka
```

---

## Q2. AMQP — exchange, queue, binding?

```
✅ Ответ:
Producer → Exchange (direct/fanout/topic/headers) → (binding) → Queue → Consumer.
Exchange — определяет, в какую очередь положить сообщение.
Binding — правило маршрутизации (routing_key → queue).
Queue — FIFO-буфер.
Consumer — читает и подтверждает (ack).
```

---

## Q3. Как гарантировать доставку в RabbitMQ?

```
✅ Ответ:
1. Publisher confirms — producer ждёт подтверждения от брокера
2. durable=True для очереди + persistent messages (delivery_mode=2)
3. Consumer ack после обработки (basic_ack)
4. DLX для упавших сообщений
5. Outbox pattern — атомарная запись в БД + очередь
```

---

## Q4. Kafka — что такое partition?

```
✅ Ответ:
Partition — физический сегмент лога (append-only). Все сообщения
в одной partition строго упорядочены. Partition — единица парал-
лелизма: один consumer читает одну partition.

Producer выбирает partition:
- Round-robin (равномерно)
- Key-hash (один key → одна partition → порядок)
```

---

## Q5. Kafka — offset management?

```
✅ Ответ:
Offset — позиция consumer в partition.
Auto-commit (раз в N сек) — риск повтора при сбое.
Manual commit — at-least-once (commit после обработки).
For exactly-once: idempotent producer + manual commit.
auto.offset.reset: earliest (с первого) / latest (только новые).
```

---

## Q6. DLX в RabbitMQ?

```
✅ Ответ:
Dead Letter Exchange — exchange для упавших сообщений.
Сообщение попадает туда если:
1. Consumer reject с requeue=false
2. Истек TTL сообщения
3. Достигнут лимит попыток

Без DLX сообщение теряется навсегда.
```