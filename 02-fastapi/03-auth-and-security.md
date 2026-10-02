# FastAPI: Auth, Security, Rate Limiting

---

## JWT + OAuth2

```python
from datetime import datetime, timedelta, timezone
from jose import JWTError, jwt
from passlib.context import CryptContext

SECRET_KEY = "secret-please-use-env"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def get_password_hash(password: str) -> str:
    return pwd_context.hash(password)


def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
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
        user_id: int = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    user = await user_repo.get_by_id(db, user_id)
    if user is None:
        raise credentials_exception
    return user
```

---

## OAuth2 Password Flow

```python
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/auth/login")

@app.post("/api/auth/login")
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    user = await user_repo.get_by_email(db, form_data.username)
    if not user or not verify_password(form_data.password, user.hashed_password):
        raise HTTPException(status_code=401, detail="Invalid credentials")

    token = create_access_token({"sub": str(user.id)})
    return {"access_token": token, "token_type": "bearer"}
```

---

## Role-based access

```python
from functools import wraps
from typing import Literal

def require_role(role: Literal["admin", "manager"]):
    def dependency(current_user: User = Depends(get_current_user)):
        if current_user.role != role:
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return current_user
    return dependency

@app.get("/api/admin/users")
async def admin_list_users(
    current_user: User = Depends(require_role("admin")),
    db: AsyncSession = Depends(get_db),
):
    ...
```

---

## Rate Limiting (in-memory)

```python
import time
from collections import defaultdict

class RateLimiter:
    def __init__(self):
        self._requests: dict[str, list[float]] = defaultdict(list)

    async def check(self, key: str, max_calls: int = 100, period: float = 60.0):
        now = time.monotonic()
        self._requests[key] = [t for t in self._requests[key] if now - t < period]
        if len(self._requests[key]) >= max_calls:
            raise HTTPException(
                status_code=429,
                detail="Rate limit exceeded",
                headers={"Retry-After": str(int(period))},
            )
        self._requests[key].append(now)

rate_limiter = RateLimiter()

@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    client_ip = request.client.host
    await rate_limiter.check(client_ip)
    response = await call_next(request)
    return response
```

---

## Подводные камни

| ❌ Ошибка | ✅ Правильно |
|---|---|
| JWT без `exp` | Токен живёт вечно — всегда ставьте `exp` |
| Хранить пароль в plain text | bcrypt / argon2 через passlib |
| Rate limiting без key | По user_id или IP — иначе легко обойти |
| Не верифицировать после JWT decode | Всегда проверять `sub` и `exp` |

---

> **Технически:** JWT — stateless, Payload + Signature. Passlib — hashing библиотека.
> OAuth2 — протокол авторизации (не аутентификации). Rate limiting — защита от DDoS.