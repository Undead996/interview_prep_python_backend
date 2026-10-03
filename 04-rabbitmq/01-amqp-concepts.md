# RabbitMQ: AMQP-модель и концепции — глубоко

> **Цель:** понять AMQP-модель снизу: exchange, queue, binding, DLX, vhosts — и когда что применять.

---

## 1. AMQP-модель — что куда идёт

```
                            ┌──────────┐
                   ┌───────►│ Queue A  │───► Consumer A1 (ack)
                   │        └──────────┘
Producer ──► Exchange ──┤
   (publish)    │        ┌──────────┐
                   └───────►│ Queue B  │───► Consumer B1 (ack)
                            └──────────┘
```

**Producer** — приложение, которое публикует сообщение. Оно **не знает**, кто его прочитает. Знает только exchange и routing_key.

**Exchange** — «почтальон»: по routing_key и binding-правилам решает, в какую очередь положить.

**Queue** — FIFO-буфер, где сообщения ждут consumer.

**Binding** — правило: «сообщения с routing_key=X → очередь Y».

**Consumer** — читает очередь и подтверждает обработку (ack) или отклоняет (nack/reject).

**VHost** — виртуальный хост (изоляция: права, exchanges, очереди). По умолчанию — `/`.

**Channel** — легковесное TCP-соединение внутри одного connection. Позволяет параллельно работать с RabbitMQ без открытия новых TCP-сокетов (1 connection = до 65K channels).

---

## 2. Типы Exchange — детально с примерами

### Direct (точное совпадение routing_key)

```
Producer: routing_key="order.created"
Exchange(direct):
  binding("order.created") → Queue "orders"
  binding("user.created")  → Queue "users"
```

**Когда:** 1:1 доставка, RPC, конкретная задача в конкретную очередь.

### Fanout (broadcast всем подписанным очередям)

```
Producer: (routing_key игнорируется)
Exchange(fanout):
  → Queue "email"  (send email)
  → Queue "sms"    (send sms)
  → Queue "push"   (send push)
```

**Когда:** одно событие → несколько независимых обработчиков. Например: «заказ создан» → отправить email, обновить аналитику, очистить кэш.

### Topic (маршрутизация по шаблону routing_key)

```
Producer: routing_key="order.created.europe"
Exchange(topic):
  binding("order.#")        → Queue "all-order-events"
  binding("*.created.*")    → Queue "all-creation-events"
  binding("#.europe")       → Queue "europe-events"
```

**Символы:**
- `*` — ровно одно слово (разделённое точкой)
- `#` — ноль или более слов

**Когда:** сложная гибкая маршрутизация. Например: события по регионам (`order.*.europe`), по типам (`*.created.*`).

### Headers (маршрутизация по заголовкам сообщения)

```
Producer: headers={"format": "json", "type": "report"}
Exchange(headers):
  binding(x-match=all, format=json, type=report) → Queue "json-reports"
  binding(x-match=any, format=xml, format=csv)   → Queue "structured-reports"
```

**Когда:** если routing_key неудобен — например, множество опциональных параметров.

---

## 3. Свойства очередей

| Свойство | Что делает | Когда use |
|----------|-----------|----------|
| `durable=True` | Очередь переживает рестарт брокера | Всегда для продакшена |
| `auto_delete=True` | Удаляется, когда отключается последний consumer | Временные RPC-очереди |
| `exclusive=True` | Только один consumer, удаляется при его отключении | RPC, временные очереди |
| `x-message-ttl` | TTL сообщения в ms (если не обработано — в DLX) | Retry-очереди |
| `x-dead-letter-exchange` | Куда уходят упавшие сообщения | Всегда для продакшена |
| `x-max-length` | Максимальная длина очереди | Защита от переполнения |
| `x-queue-mode=lazy` | Lazy queue — сообщения на диске, не в RAM | Большие очереди (>1M сообщений) |

---

## 4. Dead Letter Exchange (DLX)

**Проблема:** consumer не может обработать сообщение. Что дальше?

**Без DLX:** `basic.reject(requeue=False)` → сообщение **теряется навсегда**.

**С DLX:** reject → сообщение уходит на DLX → специальный consumer мониторит ошибки.

```python
# Настройка:
channel.queue_declare(
    queue="orders",
    arguments={
        "x-dead-letter-exchange": "orders.dlx",
        "x-message-ttl": 86_400_000,  # 24 часа
        "x-dead-letter-routing-key": "orders.dead",
    }
)

# Очередь для dead-сообщений:
channel.queue_declare("orders.dead", durable=True)
channel.queue_bind("orders.dead", "orders.dlx", "orders.dead")
```

### Retry-паттерн через DLX

```
Queue "orders" (TTL=1s) → DLX "orders.retry" → Queue "orders.work" → Consumer
Если ошибка → reject → "orders.retry" → "orders.work" (через 1s) → retry

Реализация:
1. orders.work — основная рабочая очередь
2. orders.retry — очередь с TTL=delay (exponential backoff)
3. orders.dead — DLQ после N попыток (через x-retries header)
```

---

## 5. Quorum Queues (RabbitMQ 3.8+)

```python
channel.queue_declare(
    queue="critical-orders",
    arguments={"x-queue-type": "quorum"},
    durable=True,
)
```

**Преимущества над классическими mirrored-очередями:**
- RAFT-консенсус (не master-slave)
- Автоматическое восстановление после split-brain
- Гарантированная доставка (не теряются сообщения при сбое узла)

**Минусы:** чуть выше latency, нельзя `auto_delete` и `exclusive`.

---

## 6. VHosts и права

```
VHost "/"         — дефолтный
VHost "staging"   — тестовое окружение
VHost "customers" — отдельный домен для customer-сервиса

rabbitmqctl add_vhost staging
rabbitmqctl set_permissions -p staging user ".*" ".*" ".*"
```

**Когда:** изоляция окружений в одном кластере, продуктовые домены, multi-tenancy.

---

> **На собесе:** «Как работает RabbitMQ?» —
> «Producer шлёт сообщение в exchange. Exchange по routing key и bindings определяет очередь. Consumer читает и подтверждает (basic_ack). Если не подтвердил — после таймаута сообщение возвращается. DLX — для упавших сообщений. Ключевые гарантии: publisher confirms (producer-уровень), consumer ack, persistent messages + durable queues, outbox pattern.»