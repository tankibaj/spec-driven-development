---
name: add-alembic-migration
description: Add a database migration with Alembic in a backend workspace. Use when adding or modifying database tables during Phase 4.
---

# Skill: Add an Alembic Migration

Use this skill when adding or modifying a database table in a backend workspace.

---

## Steps

### 1. Update the ORM model (`src/models/{entity}.py`)

```python
from sqlalchemy import String, Integer, ForeignKey, func
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy.dialects.postgresql import UUID as PGUUID
from src.models.base import Base
import uuid

class Order(Base):
    __tablename__ = "orders"

    id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4
    )
    reference: Mapped[str] = mapped_column(String(30), unique=True, nullable=False)
    status: Mapped[str] = mapped_column(String(20), nullable=False, default="pending")
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
```

Export from `src/models/__init__.py` so Alembic's `env.py` sees it.

### 2. Generate the migration

```bash
uv run alembic revision --autogenerate -m "add orders table"
```

### 3. Review the generated file (`alembic/versions/*.py`)

**Always review before applying.** Autogenerate misses:
- `CheckConstraint` — add manually in `upgrade()`
- Custom index names — verify they match your naming convention
- Enum types on PostgreSQL — may need `sa.Enum(...)` instead of `String`
- Cascade rules on foreign keys — verify they are correct

Both `upgrade()` and `downgrade()` must be complete and reversible.

### 4. Apply locally

```bash
uv run alembic upgrade head
```

### 5. Write a migration integration test (`tests/integration/test_migrations.py`)

```python
async def test_orders_table_exists(db_session):
    result = await db_session.execute(
        text("SELECT column_name, data_type FROM information_schema.columns "
             "WHERE table_name = 'orders'")
    )
    columns = {row.column_name: row.data_type for row in result}
    assert "id" in columns
    assert "reference" in columns
    assert columns["status"] == "character varying"
```

---

## Common Pitfalls

- Never separate the migration file from the model change in a commit — commit both together
- If two developers generate migrations simultaneously, Alembic will create a branch — resolve with `alembic merge heads`
- `nullable=False` columns added to an existing table need a `server_default` or a data migration step — otherwise `upgrade()` will fail on non-empty tables
- Always run `alembic downgrade -1` then `alembic upgrade head` locally to verify the round-trip

---

## Checklist before moving on

- [ ] Migration reviewed manually — not just accepted from autogenerate
- [ ] `upgrade()` and `downgrade()` both work cleanly
- [ ] Model and migration committed together
- [ ] Migration integration test written
