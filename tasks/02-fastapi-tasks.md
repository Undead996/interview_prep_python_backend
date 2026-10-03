# Задачи FastAPI — 5 задач с разбором

---

## Задача 1. CRUD с SQLAlchemy 2.0 async + Pydantic

```python
# Pydantic:
class ProductCreate(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    price: float = Field(gt=0)

class ProductResponse(BaseModel):
    id: int
    name: str
    price: float
    model_config = {"from_attributes": True}

# Endpoints:
@app.get("/api/products/{product_id}", response_model=ProductResponse)
async def get_product(product_id: int, db: AsyncSession = Depends(get_db)):
    stmt = select(Product).where(Product.id == product_id)
    result = await db.execute(stmt)
    if (product := result.scalar_one_or_none()) is None:
        raise HTTPException(404, detail="Product not found")
    return product

@app.post("/api/products", status_code=201, response_model=ProductResponse)
async def create_product(payload: ProductCreate, db: AsyncSession = Depends(get_db)):
    product = Product(**payload.model_dump())
    db.add(product)
    await db.commit()
    await db.refresh(product)
    return product

@app.put("/api/products/{product_id}", response_model=ProductResponse)
async def update_product(product_id: int, payload: ProductCreate, db: AsyncSession = Depends(get_db)):
    stmt = select(Product).where(Product.id == product_id)
    result = await db.execute(stmt)
    if (product := result.scalar_one_or_none()) is None:
        raise HTTPException(404)
    for key, value in payload.model_dump().items():
        setattr(product, key, value)
    await db.commit()
    await db.refresh(product)
    return product

@app.delete("/api/products/{product_id}", status_code=204)
async def delete_product(product_id: int, db: AsyncSession = Depends(get_db)):
    stmt = select(Product).where(Product.id == product_id)
    result = await db.execute(stmt)
    if (product := result.scalar_one_or_none()) is None:
        raise HTTPException(404)
    await db.delete(product)
    await db.commit()
```

**Что проверяют:** SQLAlchemy async, Pydantic, HTTP-статусы, обработка 404.

---

## Задача 2. JWT-аутентификация + RBAC

```python
SECRET_KEY = "your-secret-32-chars-min"
ALGORITHM = "HS256"

def create_access_token(user_id: int, role: str) -> str:
    return jwt.encode({
        "sub": str(user_id),
        "role": role,
        "exp": datetime.now(timezone.utc) + timedelta(minutes=30),
    }, SECRET_KEY, algorithm=ALGORITHM)

async def get_current_user(token: str = Depends(oauth2_scheme), db: AsyncSession = Depends(get_db)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user = await db.get(User, int(payload["sub"]))
        if not user: raise HTTPException(401)
        return user
    except (JWTError, ValueError): raise HTTPException(401)

def require_role(role: str):
    async def checker(token: str = Depends(oauth2_scheme)):
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        if payload.get("role") != role: raise HTTPException(403)
        return True
    return checker
```

---

## Задача 3. Dependency Override в тестах

```python
@pytest_asyncio.fixture
async def client():
    engine = create_async_engine("sqlite+aiosqlite://", echo=False)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    async_session = async_sessionmaker(engine, expire_on_commit=False)

    async def override():
        async with async_session() as s: yield s

    app.dependency_overrides[get_db] = override
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac
    app.dependency_overrides.clear()
    await engine.dispose()

@pytest.mark.asyncio
async def test_create_product(client: AsyncClient):
    resp = await client.post("/api/products", json={"name": "Widget", "price": 9.99})
    assert resp.status_code == 201
    assert resp.json()["name"] == "Widget"
```

---

## Задача 4. Контрактный тест OpenAPI

```python
@pytest.mark.asyncio
async def test_openapi_schema(client: AsyncClient):
    schema_resp = await client.get("/openapi.json")
    schema = schema_resp.json()

    # Все эндпоинты имеют summary
    missing = []
    for path, methods in schema["paths"].items():
        for method in methods:
            if "summary" not in methods[method]:
                missing.append(f"{method.upper()} {path}")
    assert not missing, f"Missing summary: {missing}"

    # Все response_model возвращают 200 или 201
    for path, methods in schema["paths"].items():
        for method, details in methods.items():
            if method in ("post", "put", "patch"):
                assert "201" in details.get("responses", {}) or "200" in details.get("responses", {}), \
                    f"{method.upper()} {path} has no 200/201"
```

---

## Задача 5. Rate limiter middleware с Redis

```python
@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    redis = request.app.state.redis
    ip = request.client.host if request.client else "unknown"
    key = f"rl:{ip}:{int(time.time()) // 60}"
    count = await redis.incr(key)
    if count == 1: await redis.expire(key, 61)
    if count > 100:
        return JSONResponse(status_code=429, detail="Too many requests", headers={"Retry-After": "60"})
    return await call_next(request)
```