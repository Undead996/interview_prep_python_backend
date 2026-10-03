# RabbitMQ: Надёжность и паттерны — глубоко

> **Цель:** понять, как гарантировать, что сообщение **точно дойдёт** до consumer,
> и как организовать retry + DLX.

---

## 1. Publisher Confirms — как producer узнаёт, что сообщение дошло

```python
channel.confirm_delivery()  # включает подтверждения

def on_confirm(frame):
    if isinstance(frame, pika.frame.ConfirmFrame):
        print("Сообщение получено брокером")
    else:
        print("Сообщение ПОТЕРЯНО!")

channel.basic_publish(
    exchange="orders",
    routing_key="order.created",
    body=msg,
    mandatory=True,  # сообщение ТРЕБУЕТ очередь
)
```

**Без publisher confirms:** producer отправляет и "забывает". Если брокер упал до записи — сообщение потеряно.

**С publisher confirms:** producer ждёт подтверждения от брокера, что сообщение сохранилось (и продублировано на реплики).

---

## 2. Consumer Ack — как consumer сообщает, что обработал

```
Consumer получил → обработал → basic_ack → сообщение удаляется
                            → basic_nack + requeue=True → возвращается в очередь
                            → basic_reject + requeue=False → в DLX
```

```python
def callback(ch, method, properties, body):
    try:
        process_order(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)  # ✅ успех — удалить
    except TemporaryError:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)  # повторить
    except PermanentError:
        ch.basic_reject(delivery_tag=method.delivery_tag, requeue=False)  # → DLX
```

**Важно:** если TCP-соединение упало **без ack**, после таймаута сообщение возвращается в очередь (другой consumer его получит). Это даёт **at-least-once** гарантию.

---

## 3. Outbox Pattern — гарантия "запись + отправка"

**Проблема:** приложение записывает в БД, потом шлёт в RabbitMQ. Если упадёт между ними — сообщение потеряно:

```
1. Запись в БД ✅
2. [CRASH!] → отправка не произошла → сообщение потеряно
```

**Решение (Outbox pattern):**
```
1. В той же транзакции: запись в бизнес-таблицу + запись в outbox-таблицу
2. Фоновый poller читает outbox, отправляет в RabbitMQ
3. После подтверждения от RMQ — помечает outbox.status = 'done'
4. При старте — повторяем все pending
```

```sql
CREATE TABLE outbox (
    id BIGSERIAL,
    topic TEXT NOT NULL,
    payload JSONB NOT NULL,
    status TEXT DEFAULT 'pending',
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

> **На собесе:** «Как гарантировать доставку в RabbitMQ?» —  
> «Три уровня: 1) Publisher confirms — producer ждёт, что брокер получил сообщение.
> 2) Consumer ack — consumer подтверждает обработку. 3) Outbox pattern —
> атомарная запись в БД + очередь через таблицу outbox. DLX для упавших.»