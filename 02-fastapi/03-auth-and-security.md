# FastAPI: Auth, Security, Rate Limiting — глубоко

> **Цель:** понять, как работает JWT (не просто скопировать код), построить правильную аутентификацию и защиту от атак.

---

## 1. JWT — что внутри токена и как он работает

JWT — это **три base64url-закодированные строки**, разделённые точкой:

```
header.payload.signature
```

### Header
```json
{"alg": "HS256", "typ": "JWT"}
```
Указывает алгоритм подписи (HS256 = HMAC-SHA256) или RS256 (RSA).

### Payload (claims)
```json
{
  "sub": "42",              // subject — ID пользователя
  "exp": 1712345678,        // expiration — когда истекает (Unix-время)
  "iat": 1712342078,        // issued at — когда создан
  "iss": "auth-service",    // issuer — кто выдал
  "aud": "api",             // audience — для кого
  "role": "admin"           // кастомные claims
}
```

### Signature
```
HMAC-SHA256(base64url(header) + "." + base64url(payload), SECRET_KEY)
```

**Ключевое:** payload **читает кто угодно** — base64url это кодирование, не шифрование. Signature нужна чтобы гарантировать, что токен **не подделан**: сервер вычисляет подпись заново и сравнивает. Если совпадает — токен настоящий.

### Почему `exp` обязателен?

```python
# ❌ Без exp — токен живёт вечно
token = jwt.encode({"sub": "1"}, SECRET_KEY, algorithm="HS256")
# Через год — всё ещё валиден!

# ✅ С exp:
from datetime import datetime, timedelta, timezone

token = jwt.encode(
    {
        "sub": "1",
        "exp": datetime.now(timezone.utc) + timedelta(minutes=30),
        "iat": datetime.now(timezone.utc),
    },
    SECRET_KEY,
    algorithm="HS256",
)
```

### Refresh token — зачем и как

```
Access token (30 min) — для запросов к API
Refresh token (7 days) — для получения нового access-токена без логина

Процесс:
1. Логин → получаем access + refresh
2. Access истёк → отправляем refresh на /auth/refresh
3. Сервер проверяет refresh (в БД/Redis) → выдаёт новую пару
4. Если refresh скомпрометирован — отзываем в БД
```

---

## 2. OAuth2 Password Flow — реализация

```
POST /api/auth/login
Body: {username, password}
  ↓
1. Ищем пользователя по email
2. Сравниваем пароль через bcrypt
3. Создаём JWT: access (30 min) + refresh (7 days)
4. Возвращаем {"access_token": "...", "refresh_token": "...", "token_type": "bearer"}
  ↓
GET /api/profile
Header: Authorization: Bearer <access_token>
  ↓
1. OAuth2PasswordBearer извлекает токен из заголовка
2. jwt.decode(token, SECRET_KEY)
3. Проверка exp
4. Поиск пользователя по sub
5. Возврат пользователя
```

```python
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from passlib.context import CryptContext
from jose import JWTError, jwt
from datetime import datetime, timedelta, timezone

SECRET_KEY = "your-secret-key-at-least-32-chars"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE = timedelta(minutes=30)
REFRESH_TOKEN_EXPIRE = timedelta(days=7)

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/auth/login")

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def create_token(data: dict, expires: timedelta) -> str:
    to_encode = data.copy()
    to_encode.update({
        "exp": datetime.now(timezone.utc) + expires,
        "iat": datetime.now(timezone.utc),
    })
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    credentials_exception = HTTPException(
        status_code=401,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: str | None = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    user = await db.get(User, int(user_id))
    if user is None:
        raise credentials_exception
    return user
```

---

## 3. RBAC (Role-Based Access Control)

```python
from enum import StrEnum

class Role(StrEnum):
    ADMIN = "admin"
    MANAGER = "manager"
    USER = "user"

def require_role(*allowed: str):
    """Depends-фабрика: проверяет роль текущего пользователя."""
    async def role_checker(current_user: User = Depends(get_current_user)) -> User:
        if current_user.role not in allowed:
            raise HTTPException(
                status_code=403,
                detail=f"Role {current_user.role!r} not in {allowed}",
            )
        return current_user
    return role_checker

@app.get("/admin/dashboard")
async def admin_dashboard(_: User = Depends(require_role(Role.ADMIN))):
    return {"secret": "admin-only data"}

@app.get("/manage/reports")
async def reports(_: User = Depends(require_role(Role.ADMIN, Role.MANAGER))):
    return {"data": "managers and admins"}
```

---

## 4. Rate Limiting

### In-memory (для разработки, тестов)

```python
import time
from collections import defaultdict

class InMemoryRateLimiter:
    def __init__(self):
        self._requests: dict[str, list[float]] = defaultdict(list)

    async def check(self, key: str, max_calls: int = 100, window: float = 60.0) -> bool:
        now = time.monotonic()
        # Удаляем устаревшие
        self._requests[key] = [t for t in self._requests[key] if now - t < window]
        if len(self._requests[key]) >= max_calls:
            return False
        self._requests[key].append(now)
        return True
```

**Проблема:** при рестарте — сброс. Не работает с горизонтальным масштабированием.

### Redis — sliding window (для продакшена)

```python
import time

async def check_rate_limit(
    redis: Redis,
    key: str,
    max_calls: int = 100,
    window: int = 60,
) -> bool:
    """Sliding window rate limiter с Redis. Возвращает True если можно."""
    current = int(time.time()) // window
    redis_key = f"ratelimit:{key}:{current}"

    count = await redis.incr(redis_key)
    if count == 1:
        await redis.expire(redis_key, window + 1)

    if count > max_calls:
        return False
    return True

# Middleware:
@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    redis = request.app.state.redis
    ip = request.client.host if request.client else "unknown"

    allowed = await check_rate_limit(redis, ip, max_calls=100, window=60)
    if not allowed:
        return JSONResponse(
            status_code=429,
            content={"detail": "Too many requests"},
            headers={"Retry-After": "60"},
        )

    return await call_next(request)
```

### Token bucket (алгоритм)

```python
# Token bucket: добавляем токены с фиксированной скоростью.
# Каждый запрос "тратит" 1 токен. Если токенов нет — 429.

async def token_bucket_check(
    redis: Redis,
    key: str,
    rate: int = 10,       # токенов в секунду
    burst: int = 20,      # максимальный burst
) -> bool:
    """Token bucket rate limiter. Refill rate: rate/sec, max burst: burst."""
    now = time.monotonic()
    lua_script = """
    local key = KEYS[1]
    local rate = tonumber(ARGV[1])
    local burst = tonumber(ARGV[2])
    local now = tonumber(ARGV[3])

    local data = redis.call('HMGET', key, 'tokens', 'last_refill')
    local tokens = tonumber(data[1]) or burst
    local last_refill = tonumber(data[2]) or now

    -- Refill tokens
    local elapsed = now - last_refill
    local refill = math.floor(elapsed * rate)
    tokens = math.min(burst, tokens + refill)

    if tokens < 1 then
        return 0  -- rate limited
    end

    tokens = tokens - 1
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    redis.call('EXPIRE', key, math.ceil(burst / rate) + 1)
    return 1
    """
    result = await redis.eval(lua_script, 1, key, rate, burst, now)
    return bool(result)
```

---

## 5. Security headers + CORS

```python
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from starlette.middleware.base import BaseHTTPMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://frontend.example.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type", "X-Request-ID"],
)

app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["api.example.com", "localhost"],
)

# Security headers:
@app.middleware("http")
async def security_headers(request: Request, call_next):
    response = await call_next(request)
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["X-XSS-Protection"] = "1; mode=block"
    response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
    return response
```

---

> **На собесе:** «Как работает JWT?» —
> «JWT — это три base64url-части: header (алгоритм), payload (claims: sub, exp, iat, role...), signature (HMAC-подпись от header+payload секретным ключом). Сервер проверяет подпись при каждом запросе — подделать невозможно без ключа. Payload читает кто угодно (это кодирование, не шифрование) — поэтому пароли в payload не кладут. Access-токен короткий (15–30 мин), refresh — длинный (7 дней) для обновления без повторного логина.»