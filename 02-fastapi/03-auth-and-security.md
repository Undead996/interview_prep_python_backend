# FastAPI: Auth, Security, Rate Limiting — глубоко

> **Цель:** понять, как работает JWT (не просто скопировать код), построить
> правильную аутентификацию и защиту от DDoS.

---

## 1. JWT — что внутри токена

JWT (JSON Web Token) — это **три base64-закодированные строки**, разделённые точкой:

```
header.payload.signature
```

### Header
```json
{"alg": "HS256", "typ": "JWT"}
```

### Payload (claims)
```json
{
  "sub": "42",           // subject — идентификатор пользователя
  "exp": 1712345678,    // expires — когда истекает (Unix time)
  "iat": 1712342078,    // issued at — когда создан
  "role": "admin"       // кастомные поля
}
```

### Signature
HMAC-SHA256(`base64(header) + "." + base64(payload)`, SECRET_KEY)

**Кто угодно может прочитать payload** (base64 — не шифрование, а кодирование). Signature нужна, чтобы убедиться, что токен **не подделан**.

**Как проверить:** сервер вычисляет HMAC-SHA256 от header+payload с своим SECRET_KEY и сравнивает с signature. Если совпадает — токен настоящий.

### Почему `exp` обязателен?

```python
# ❌ Без exp — токен живёт вечно!
token = jwt.encode({"sub": "1"}, SECRET_KEY, algorithm="HS256")
# Через год — всё ещё валиден!

# ✅ С exp — токен умирает
token = jwt.encode(
    {"sub": "1", "exp": datetime.now(timezone.utc) + timedelta(minutes=30)},
    SECRET_KEY,
    algorithm="HS256",
)
```

---

## 2. OAuth2 Password Flow — как это работает

```
POST /api/auth/login {username, password}
  ↓
1. Ищем пользователя по email/username
2. Сравниваем пароль (bcrypt)
3. Создаём JWT (sub=user.id, exp=30 min)
4. Возвращаем {"access_token": "...", "token_type": "bearer"}
  ↓
Клиент сохраняет токен (localStorage / cookie)
  ↓
GET /api/profile Authorization: Bearer <token>
  ↓
1. FastAPI получает Bearer-токен (OAuth2PasswordBearer)
2. Декодируем JWT (секрет + алгоритм)
3. Проверяем exp
4. Получаем user_id из sub
5. Загружаем пользователя из БД
6. Возвращаем пользователя
```

**Важно:** OAuth2 Password Flow — это **аутентификация** (кто ты), а не авторизация (что тебе можно). Для авторизации — RBAC (Role-Based Access Control).

---

## 3. Rate Limiting — защита от DDoS

### In-memory (для development)

```python
class RateLimiter:
    def __init__(self):
        self._requests: dict[str, list[float]] = {}

    async def check(self, key: str, max_calls: int = 100, period: float = 60.0):
        now = time.monotonic()
        self._requests[key] = [t for t in self._requests[key] if now - t < period]
        if len(self._requests[key]) >= max_calls:
            raise HTTPException(status_code=429, headers={"Retry-After": str(int(period))})
        self._requests[key].append(now)
```

**Проблема:** при рестарте сервера — сброс. Не работает с горизонтальным масштабированием.

### С Redis — для продакшена

```python
async def check_rate_limit(redis: Redis, key: str, max_calls: int, window: int):
    current = int(time.time()) // window
    redis_key = f"ratelimit:{key}:{current}"
    
    count = await redis.incr(redis_key)
    if count == 1:
        await redis.expire(redis_key, window + 1)
    
    if count > max_calls:
        raise HTTPException(status_code=429)
```

---

> **На собесе:** «Как работает JWT?» —  
> «JWT — это три base64-части: header (алгоритм), payload (данные + exp),
> signature (HMAC-подпись). Сервер проверяет подпись секретным ключом.
> payload читает кто угодно — не кладите туда пароли!»