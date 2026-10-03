# RabbitMQ: AMQP-концепции — глубоко

> **Цель:** понять AMQP-модель "снизу": exchange, queue, binding, DLX, vhosts.

---

## 1. AMQP-модель — что куда идёт

```
Producer → Exchange → (bindings) → Queue → Consumer
```

**Producer** — приложение, которое шлёт сообщение. Он не знает, кто его прочитает.

**Exchange** — "почтальон", решает, в какую очередь положить сообщение.

**Queue** — буфер, где сообщения ждут, пока consumer их заберёт.

**Consumer** — приложение, которое читает (может подтвердить = ack).

**Binding** — правило: "сообщения с routing_key = X клади в очередь Y".

---

## 2. Типы Exchange — как они работают

| Тип | Маршрутизация | Характеристики |
|-----|--------------|----------------|
| **Direct** | routing_key = queue name | 1:1, точное совпадение |
| **Fanout** | всем подписанным очередям | 1:N, broadcast |
| **Topic** | routing_key по шаблону (topic.#) | Гибкая маршрутизация |
| **Headers** | по заголовкам (key-value) | Максимальная гибкость |

### Direct — когда нужно отправить "точно в одну очередь"

```
Producer → Exchange(direct) → binding("order.created") → Queue "orders"
```

### Fanout — когда нужно разослать всем

```
Producer → Exchange(fanout) → Queue "email" (всем)
                             → Queue "sms" (всем)
                             → Queue "push" (всем)
```

### Topic — когда нужно фильтровать по routing key

```
Producer отправляет с routing_key = "user.created.europe"
Exchange(topic) → binding("user.#") → Queue "user-events"
                → binding("*.created.*") → Queue "creation-events"
```

---

## 3. Dead Letter Exchange (DLX)

**Проблема:** что происходит с сообщением, которое consumer не смог обработать?

**Без DLX:** reject → сообщение **теряется навсегда**.

**С DLX:** reject → сообщение уходит на DLX → его может прочитать специальный consumer (мониторинг ошибок, повторная попытка).

```python
# Настройка DLX для очереди
channel.queue_declare(
    queue="orders",
    arguments={
        "x-dead-letter-exchange": "orders.dlx",  # куда уходят упавшие
        "x-message-ttl": 86400000,  # 24 часа в ms
    }
)
```

**Когда сообщение попадает в DLX:**
1. `basic.reject` с `requeue=false`
2. Истечение TTL (x-message-ttl)
3. Превышение лимита попыток (через custom header x-retry)

---

> **На собесе:** «Как работает RabbitMQ?» —  
> «Producer отправляет сообщение в exchange. Exchange по routing key
> и bindings кладёт в очередь. Consumer забирает (pull) или получает
> (push). После обработки — ack. Если ack нет — сообщение возвращается
> в очередь (после timeout). DLX для упавших сообщений.»