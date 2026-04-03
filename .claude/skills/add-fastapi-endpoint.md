# Skill: Add a FastAPI Endpoint

Use this skill when implementing a new API endpoint in a backend workspace.

---

## Steps

### 1. Define schemas (`src/schemas/{resource}.py`)

```python
from pydantic import BaseModel, Field
from uuid import UUID

class CreateOrderRequest(BaseModel):
    # fields matching the OpenAPI request body exactly
    ...

class OrderResponse(BaseModel):
    # fields matching the OpenAPI response body exactly
    id: UUID
    ...

    model_config = {"from_attributes": True}  # enables ORM → schema conversion
```

Export from `src/schemas/__init__.py`.

### 2. Add the repository method (`src/repositories/{resource}.py`)

DB access only — no business logic.

```python
from sqlalchemy.ext.asyncio import AsyncSession
from src.models.order import Order

class OrderRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def create(self, data: dict) -> Order:
        obj = Order(**data)
        self.session.add(obj)
        await self.session.flush()
        await self.session.refresh(obj)
        return obj
```

### 3. Add the service method (`src/services/{resource}.py`)

Business logic only — call repositories, raise domain exceptions (not HTTP exceptions).

```python
from src.repositories.order import OrderRepository
from src.schemas.order import CreateOrderRequest, OrderResponse

class OrderService:
    def __init__(self, repo: OrderRepository):
        self.repo = repo

    async def create_order(self, request: CreateOrderRequest) -> OrderResponse:
        # business logic here
        order = await self.repo.create(request.model_dump())
        return OrderResponse.model_validate(order)
```

### 4. Add the route (`src/api/v1/{resource}.py`)

Map domain exceptions to HTTP exceptions here — not in the service.

```python
from fastapi import APIRouter, Depends, HTTPException, status
from src.schemas.order import CreateOrderRequest, OrderResponse
from src.services.order import OrderService
from src.dependencies import get_order_service

router = APIRouter()

@router.post("/", response_model=OrderResponse, status_code=status.HTTP_201_CREATED)
async def create_order(
    request: CreateOrderRequest,
    service: OrderService = Depends(get_order_service),
):
    try:
        return await service.create_order(request)
    except SomeDomainException as e:
        raise HTTPException(status_code=409, detail=str(e))
```

### 5. Mount the route (`src/api/router.py`)

```python
from src.api.v1.orders import router as orders_router
router.include_router(orders_router, prefix="/v1/orders", tags=["Orders"])
```

### 6. Write tests

- `tests/unit/services/test_{resource}.py` — mock the repository, test service logic
- `tests/integration/test_{resource}_api.py` — full request through the stack with real DB

---

## Checklist before moving on

- [ ] Response shape matches the OpenAPI spec exactly (field names, types, nullability)
- [ ] Domain exceptions are caught in the route and mapped to correct HTTP status codes
- [ ] Route is mounted in `src/api/router.py`
- [ ] Unit test covers the service method
- [ ] Integration test covers the full HTTP request → DB → response path
