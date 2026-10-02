# Задачи FastAPI

---

## Задача 1. CRUD с SQLAlchemy 2.0

**Условие:** Напишите эндпоинты GET/POST/PUT/DELETE для `Product(id, name, price)`
с SQLAlchemy 2.0 async + Dependency Override.

<details>
<summary>Решение</summary>

```python
# models.py
class Product(Base):
    __tablename__ = "products"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    price: Mapped[float] = mapped_column(Float)

# schemas.py
class ProductCreate(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    price: float = Field(gt=0)

# main.py
from sqlalchemy import select, delete

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
</details>

---

## Задача 2. JWT-аутентификация

**Условие:** Напишите логин + защищённый эндпоинт `/api/profile`
с Bearer JWT-токеном.

<details>
<summary>Решение</summary>

```python
from jose import JWTError, jwt
from passlib.context import CryptContext
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm

SECRET_KEY = "secret"
ALGORITHM = "HS256"
pwd_context = CryptContext(schemes=["bcrypt"])
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/auth/login")

def create_token(user_id: int) -> str:
    return jwt.encode({"sub": str(user_id), "exp": datetime.utcnow() + timedelta(hours=1)}, SECRET_KEY, algorithm=ALGORITHM)

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

@app.post("/api/auth/login")
async def login(form: OAuth2PasswordRequestForm = Depends()):
    user = await repo.get_by_email(db, form.username)
    if not user or not pwd_context.verify(form.password, user.hashed_password):
        raise HTTPException(status_code=401)
    return {"access_token": create_token(user.id), "token_type": "bearer"}

@app.get("/api/profile")
async def get_profile(current_user: User = Depends(get_current_user)):
    return current_user
```
</details>

---

## Задача 3. Контрактный тест OpenAPI

**Условие:** Напишите тест, проверяющий, что все эндпоинты FastAPI имеют
`summary` и ответ соответствует схеме.

<details>
<summary>Решение</summary>

```python
from jsonschema import validate, ValidationError

def test_all_endpoints_have_summary(client):
    schema = client.get("/openapi.json").json()
    missing = []
    for path, methods in schema["paths"].items():
        for method in methods:
            if "summary" not in methods[method]:
                missing.append(f"{method.upper()} {path}")
    assert not missing, f"Missing summary: {missing}"

def test_create_user_response_matches_schema(client):
    schema = client.get("/openapi.json").json()
    post_schema = schema["paths"]["/api/users"]["post"]["responses"]["201"]["content"]["application/json"]["schema"]
    resp = client.post("/api/users", json={"name": "Alice", "email": "alice@ex.com", "age": 30})
    assert resp.status_code == 201
    try:
        validate(instance=resp.json(), schema=post_schema)
    except ValidationError as e:
        pytest.fail(f"Schema mismatch: {e}")
```
</details>

---

> **На собесе:** Задачи FastAPI проверяют два навыка: SQLAlchemy async + DI.
> Покажи, что умеешь использовать Depends, умеешь писать тесты с TestClient
> и dependency_overrides.