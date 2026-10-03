# Задачи FastAPI — расширенный разбор

---

## Задача 1. CRUD с SQLAlchemy 2.0

**Условие:** Эндпоинты GET/POST/PUT/DELETE для `Product(id, name, price)` с SQLAlchemy 2.0 async + Dependency Override.

```python
# Product model
class Product(Base):
    __tablename__ = "products"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    price: Mapped[float] = mapped_column(Float)

# Pydantic schema
class ProductCreate(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    price: float = Field(gt=0)

# Endpoint
@app.get("/api/products/{product_id}")
async def get_product(product_id: int, db: AsyncSession = Depends(get_db)):
    stmt = select(Product).where(Product.id == product_id)
    result = await db.execute(stmt)
    product = result.scalar_one_or_none()
    if not product:
        raise HTTPException(status_code=404)
    return product

@app.post("/api/products", status_code=201)
async def create_product(payload: ProductCreate, db: AsyncSession = Depends(get_db)):
    product = Product(**payload.model_dump())
    db.add(product)
    await db.commit()
    await db.refresh(product)
    return product
```

**Что проверяют:** SQLAlchemy async, Pydantic-валидация, HTTP-статусы, обработка not found.

---

## Задача 2. JWT-аутентификация

**Условие:** Логин + защищённый эндпоинт `/api/profile` с Bearer JWT-токеном.

```python
from jose import JWTError, jwt
from passlib.context import CryptContext

SECRET_KEY = "secret"
ALGORITHM = "HS256"
pwd_context = CryptContext(schemes=["bcrypt"])
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/auth/login")

def create_token(user_id: int) -> str:
    return jwt.encode(
        {"sub": str(user_id), "exp": datetime.utcnow() + timedelta(hours=1)},
        SECRET_KEY, algorithm=ALGORITHM
    )

async def get_current_user(token: str = Depends(oauth2_scheme), db: AsyncSession = Depends(get_db)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id = int(payload.get("sub"))
    except (JWTError, ValueError):
        raise HTTPException(status_code=401)
    user = await db.get(User, user_id)
    if not user:
        raise HTTPException(status_code=401)
    return user
```

**Что проверяют:** JWT encode/decode, bcrypt, OAuth2PasswordBearer, dependency override.

---

## Задача 3. Контрактный тест OpenAPI

**Условие:** Проверить, что все эндпоинты имеют `summary` и ответ соответствует схеме.

```python
def test_all_endpoints_have_summary(client):
    schema = client.get("/openapi.json").json()
    missing = []
    for path, methods in schema["paths"].items():
        for method in methods:
            if "summary" not in methods[method]:
                missing.append(f"{method.upper()} {path}")
    assert not missing, f"Missing summary: {missing}"
```

**Что проверяют:** понимание контрактного тестирования, jsonschema, OpenAPI.