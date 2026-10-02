# Мок-интервью: Очереди (RabbitMQ + Kafka, 20 вопросов)

---

## Q1. RabbitMQ vs Kafka — что выберете и почему?

```
✅ Ответ: «Зависит от сценария.
RabbitMQ — когда нужно: 1) отправить сообщение и забыть, 2) RPC (reply-to),
3) подтверждение доставки (ack/nack), 4) простые очереди.
Kafka — когда нужно: 1) хранить историю событий, 2) несколько consumer-
групп читают одно и то же, 3) high throughput, 4) event sourcing.
Типичный выбор: нотификации → RabbitMQ, аудит → Kafka.»
```

## Q2. AMQP — exchange, queue, binding?

```
✅ Ответ: «Producer → Exchange (direct/fanout/topic/headers) → (binding) →
Queue → Consumer. Exchange по routing key решает, в какую очередь отдать.
Queue — FIFO. Binding — правило маршрутизации.»
```

## Q3. Как гарантировать доставку в RabbitMQ?

```
✅ Ответ: «Producer-side: publisher confirms (channel.confirm_delivery()).
Queue: durable=True, persistent messages (delivery_mode=2).
Consumer-side: ack вручную после обработки (basic_ack). DLX для
отклонённых. Outbox pattern для гарантии записи + отправки.»
```

## Q4. Kafka — что такое partition?

```
✅ Ответ: «Физический сегмент лога — append-only последовательность
сообщений. Все сообщения в одной partition — строго упорядочены.
Partition — единица параллелизма: один consumer читает одну partition.
Выбор: round-robin или key-hash (гарантирует порядок для ключа).»
```

## Q5. Kafka — offset management?

```
✅ Ответ: «Offset — позиция consumer в partition. auto.commit — риск
повтора при падении. manual.commit — at-least-once. Для exactly-once —
idempotent producer + manual commit или Kafka Transaction API.
auto.offset.reset — earliest/latest.»
```

## Q6. DLX в RabbitMQ?

```
✅ Ответ: «Dead Letter Exchange — exchange для сообщений, которые:
1) consumer reject с requeue=false, 2) истек TTL, 3) достигнут лимит
попыток. Решает: ретраи, мониторинг ошибок. Без DLX — сообщение
теряется навсегда.»
```