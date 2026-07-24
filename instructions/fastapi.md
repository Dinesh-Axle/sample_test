# FastAPI — Copilot Instructions

Extends [engineering-standards](../engineering-standards/copilot-instructions.md). Rules here apply to every repository exposing a FastAPI service.

---

## Pydantic

- **Use Pydantic v2 API only.** Never use v1 equivalents.

  | ✅ v2 | ❌ v1 (never use) |
  |-------|------------------|
  | `field_validator` | `validator` |
  | `model_dump()` | `.dict()` |
  | `model_validate()` | `.parse_obj()` |
  | `model_validate_json()` | `.parse_raw()` |

- Use `Field(...)` for required fields, `Field(default=...)` for optional ones.
- Field validators use `@field_validator` with `@classmethod`.

---

## Router & Endpoint Structure

- **All endpoints are defined in a single router file.** No blueprint splitting unless the architectural need is explicitly documented.
- Endpoint handlers contain only: request parsing, role validation, delegation to a service/utility function, and response construction. No business logic inline.
- Every endpoint must validate the user's role **before** executing any database operation or business logic.

---

## Error Handling

Catch exceptions in this order — do not rearrange:

```python
try:
    ...
except HTTPException:
    raise
except ValueError as e:
    raise HTTPException(status_code=400, detail=str(e))
except Exception as e:
    raise HTTPException(status_code=500, detail=str(e))
```

- `ValueError` signals a business-rule or validation failure → 400.
- Generic `Exception` signals an unexpected internal failure → 500.
- Never swallow exceptions silently.

---

## Type Hints

Use Python 3.10+ union syntax throughout:

```python
# ✅
def get_user(email: str | None = None) -> list[str]: ...

# ❌
from typing import Optional, List
def get_user(email: Optional[str] = None) -> List[str]: ...
```

---

## Audit Logging

Every endpoint logs an `[AUDIT]` line before executing its operation:

```python
print(f"[AUDIT] user={user_email} action={action} namespace={namespace}")
```

Include user identity, action name, and the resource or namespace being acted on. Log outcome (`SUCCESS` / `FAILED`) after the operation completes.

---

## Environment & Schema

- **Schema and database names are always environment-driven** via environment variable (e.g., `CMDL_SCHEMA`, `LIMS_ENV`). Never hardcode a schema name in application code.
- Connection parameters (host, port, user, database) may be returned by info endpoints. **Passwords must never be exposed via any API endpoint.**

---

## Testing

- Use FastAPI's `TestClient` for endpoint tests.
- Patch service dependencies at the module where they are used, not at the definition site:
  ```python
  @patch("src.endpoints.datum_client")   # ✅ patch at usage site
  @patch("src.db_utils.DatumDBClient")   # ❌ patch at definition site
  ```
- Every endpoint test must cover: happy path, role-denied (403), invalid input (400), and upstream failure (500).
