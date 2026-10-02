# RabbitMQ: AMQP-концепции

---

## Что такое AMQP

> **Простыми словами:** RabbitMQ — это почтальон. Приложение шлёт сообщение в очередь,
> почтальон доставляет его подписчику. Если подписчик занят — сообщение ждёт.

```
               ┌──────────┐
  Producer ───→│ Exchange │───→ Queue ───→ Consumer
               │   (type) │
               └──────────┘
```

---

## Exchanges — типы

| Тип | Маршрутизация | Использование |
|---|---|---|
| **Direct** | exact routing key | RPC, конкретный адресат |
| **Fanout** | broadcast всем | Логи, уведомления |
| **Topic** | routing key по шаблону `topic.*` | События предметной области |
| **Headers** | по заголовкам | Гибкая маршрутизация |

```python
# Direct
channel.basic_publish(exchange="orders", routing_key="order.created", body=msg)

# Fanout
channel.basic_publish(exchange="logs", routing_key="", body=log_entry)
# Все подписчики получат копию

# Topic
channel.basic_publish(exchange="events", routing_key="user.created", body=msg)
# Consumer подписан на routing_key="user.#"
```

---

## Queues — свойства

| Аттрибут | Назначение | Пример |
|---|---|---|
| `durable=True` | Очередь живёт после рестарта RabbitMQ | Важные задачи |
| `auto_delete` | Удаляется, когда отключился последний consumer | RPC |
| `exclusive` | Только для одного соединения | Временные |
| `arguments` | `x-max-priority`, `x-message-ttl` | Специфические |

---

## Dead Letter Exchange (DLX)

```
                DLX (dead-letter exchange)
                    ↑
Producer → Exchange → [Queue] → Consumer (ack)
                         ↓ failed
                    [DL Queue] → DL Consumer
```

Сообщение попадает в DLX если:
- Consumer rejects (`basic.reject`) с `requeue=false`
- Истек TTL сообщения
- Достигнут лимит попыток

```python
# Настройка DLX при объявлении очереди
channel.queue_declare(
    queue="orders",
    arguments={
        "x-dead-letter-exchange": "orders.dlx",
        "x-message-ttl": 86400000,  # 24 часа в ms
    }
)
```

---

## VHosts (виртуальные хосты)

```
RabbitMQ
 ├── vhost "/"         (разработка)
 ├── vhost "staging"   (тестирование)
 └── vhost "prod"      (продакшен)
```

Изоляция: приложения не видят очереди друг друга. Пользователи + права — на vhost.

---

## Channels (каналы)

Одно TCP-соединение — много каналов. Канал — логический поток сообщений.
Используйте разные каналы для разных воркеров.

```python
connection = pika.BlockingConnection(params)
channel1 = connection.channel()  # канал для publish
channel2 = connection.channel()  # канал для consume
```

---

## Подводные камни

| ❌ Ошибка | ✅ Правильно |
|---|---|
| `queue_declare` каждый раз при старте | Только при создании — Idempotent (но лишнее) |
| Нет `durable` на очереди | Выживет только в памяти |
| Exchange fanout без consumer | Сообщение теряется — очереди нет |
| DLX не настроен | Сообщения пропадают при reject |

---

> **Технически:** AMQP — протокол прикладного уровня. Сообщение → Exchange →
> (bindings) → Queue → Consumer. Подтверждение — ack/nack.
>
> **На собесе:** «Как работает RabbitMQ?» — «Producer шлёт сообщение в exchange.
> Exchange по binding rules кладёт в очередь. Consumer берёт (pull) или получает
> (push). После обработки — ack. Если ack нет — сообщение вернётся в очередь
> (после timeout).»