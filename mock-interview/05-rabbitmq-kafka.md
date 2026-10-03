# Мок-интервью: Очереди (RabbitMQ + Kafka, 20 вопросов)

---

## Q1. RabbitMQ vs Kafka — что и когда?

**RabbitMQ:** задачи, уведомления (доставить 1 раз), RPC (reply-to), сложная маршрутизация (topic exchange). **Kafka:** event sourcing, стриминг, множество consumer-групп, replay истории, high throughput (>100K msg/s). RabbitMQ = smart broker, Kafka = smart consumer.

---

## Q2. AMQP — exchange, queue, binding?

Producer → Exchange → (binding = routing_key → queue) → Queue → Consumer. Exchange типы: direct (точное совпадение), fanout (broadcast), topic (шаблоны), headers (по заголовкам). Queue — FIFO-буфер. Consumer — читает + ack.

---

## Q3. Как гарантировать доставку в RabbitMQ?

Три уровня: 1) Publisher confirms (producer → брокер). 2) Consumer ack (consumer → брокер). 3) Outbox pattern (БД + очередь атомарно). Durable queues + persistent messages. DLX для упавших. Quorum queues для критичных данных.

---

## Q4. Kafka — что такое partition и offset?

Partition — физический append-only лог. Сообщения в одной partition строго упорядочены. Partition = единица параллелизма (1 consumer на partition). Offset — позиция consumer в partition (сколько прочитано). Хранится в `__consumer_offsets` topic.

---

## Q5. Offset management — auto vs manual?

**Auto-commit:** коммитит раз в N сек. Риск: обработал → упал до commit → дубликат. **Manual:** коммитим явно после обработки. At-least-once (дубликаты возможны). Рекомендация: manual + idempotent consumer.

---

## Q6. DLX в RabbitMQ?

Dead Letter Exchange — exchange для упавших сообщений. Сообщение попадает туда при: reject с requeue=false, истечении TTL, превышении попыток. Без DLX — потеря сообщения.

---

## Q7. Что такое consumer group в Kafka?

Логическая группа consumer-ов. Partitions делятся между членами группы. Разные группы читают одни и те же partitions независимо (свои offset). Одна partition → один consumer в группе.

---

## Q8. Rebalancing в Kafka — что происходит?

Consumer упал/добавился/partition добавилась → перераспределение partitions между consumer-ами. Во время ребаланса потребление ОСТАНОВЛЕНО. Минимизировать: стабильный group.id, CooperativeStickyAssignor, session.timeout.ms не слишком маленький.

---

## Q9. Exactly-once в Kafka — как?

Idempotent producer (ProducerID + sequence number) + `acks=all` + consumer manual commit. Для транзакционной записи: Kafka Transactions (producer.init_transactions() + commit_transaction()).

---

## Q10. Outbox pattern — зачем и как?

Проблема: записали в БД, но не отправили в очередь (сервер упал). Решение: в одной транзакции пишем в бизнес-таблицу + outbox-таблицу. Фоновый poller читает outbox и отправляет в очередь. После confirms — помечает sent.

---

## Q11. ack vs nack vs reject в RabbitMQ?

`basic.ack` — обработано, удалить. `basic.nack(requeue=True)` — временная ошибка, вернуть в очередь. `basic.reject(requeue=False)` — фатальная ошибка, в DLX. nack может отклонить несколько (multiple=True).

---

## Q12. Как сделать retry в Kafka?

Ошибка → отправляем в retry-topic с x-retries header. После N попыток → DLQ. Offset коммитим СРАЗУ (не блокируем partition). Идемпотентность через Redis или БД.

---

## Q13. Что такое `acks` в Kafka?

0 — producer не ждёт подтверждения (at-most-once, потеря возможна). 1 — ждёт от лидера partition (потеря при сбое лидера). all — ждёт от всех in-sync реплик (at-least-once, максимальная надёжность).

---

## Q14. Quorum queues vs Classic mirrored?

Quorum queues (RMQ 3.8+) используют RAFT-консенсус (не master-slave). Автоматическое восстановление, гарантированная доставка. Заменяют классические mirrored-очереди для критичных данных.

---

## Q15. Что такое идемпотентный producer в Kafka?

Producer получает уникальный ProducerID. Каждое сообщение нумеруется sequence_number. Брокер дедуплицирует: если PID:seq уже записан — игнорирует. Настройка: `enable_idempotence=True`.

---

## Q16. Kafka compaction — что это?

Key compaction: храним только ПОСЛЕДНЕЕ сообщение для каждого ключа. Старые удаляются. Позволяет хранить снэпшот состояния в Kafka. Используется для CDC (Change Data Capture), KTables.

---

## Q17. aio-pika vs pika?

**aio-pika** — async/await клиент для RabbitMQ, основан на asyncio. **pika** — синхронный/блокирующий. В FastAPI-проектах — aio-pika.

---

## Q18. prefetch_count — зачем?

Ограничение количества unacked-сообщений на consumer. `channel.basic_qos(prefetch_count=10)` — не более 10 сообщений без ack. Защита от перегрузки медленного consumer.

---

## Q19. Когда использовать RabbitMQ, а когда Redis Streams?

**RabbitMQ:** полноценный брокер (ack, DLX, routing, vhosts), production-grade. **Redis Streams:** простая очередь без отдельной инфраструктуры. Для лёгких проектов — Redis, для серьёзных — RabbitMQ.

---

## Q20. Kafka vs RabbitMQ — latency?

RabbitMQ: ~1ms (сообщение сразу consumer). Kafka: ~5ms (батчинг). Для low-latency RPC — RabbitMQ. Для throughput — Kafka.