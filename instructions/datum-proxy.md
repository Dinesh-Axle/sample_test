# Datum Proxy Integration — Copilot Instructions

Extends [engineering-standards](../engineering-standards/copilot-instructions.md). Rules here apply to every service — backend API, frontend app, or ETL pipeline — that reads from or writes to PostgreSQL through the Datum proxy.

---

## Core Constraint

**Never connect to PostgreSQL directly.** All database access routes through the Datum proxy's `/proxy_request` endpoint. Do not use `psycopg2`, `psycopg`, `sqlalchemy.create_engine`, or any direct JDBC connection in application code.

---

## Request Structure

All requests to the proxy are wrapped in a `DatumProxyRequest` envelope:

```python
{
    "service": "<service_name>",
    "endpoint": "<endpoint_name>",
    "payload": { ... }   # endpoint-specific body
}
```

- **Request bodies arriving from the Datum proxy are raw JSON strings**, not pre-parsed objects. Always parse explicitly:
  ```python
  # ✅
  data = MyModel.model_validate_json(raw_body)

  # ❌ — do not rely on framework automatic body parsing
  async def endpoint(body: MyModel): ...
  ```

---

## Transactions

**Write SQL must be explicitly wrapped in a transaction.** The proxy does not auto-commit.

```sql
BEGIN;
INSERT INTO ...;
COMMIT;
```

Never issue `INSERT`, `UPDATE`, or DDL statements outside a `BEGIN; ... COMMIT;` block.

---

## SQL Safety Gate

- `validate_sql_safety()` must run on every SQL string **before it is sent to the proxy**. This is non-negotiable and must not be bypassed or weakened.
- Blocked operations: `DROP`, `TRUNCATE`, `DELETE FROM`, `ALTER ... DROP`, `ROLLBACK`.
- Raise a typed exception (e.g., `DestructiveSqlError`) when a blocked pattern is detected — never silently discard the query.

---

## Access Control

- **Default policy is `"deny"`** in every `roles_config.json`. Every new namespace and action requires an explicit role mapping before it will be accessible.
- Role validation must occur **before** the SQL is constructed or sent — not after.

---

## SQL Construction

- Use parameterized placeholders (`:param` or `%s`) for all user-supplied values.
- Where parameterization is unavailable (e.g., `SET search_path`), validate the value as alphanumeric + underscores before interpolation and document the exception inline.
- Escape string literals manually only as a last resort: `value.replace("'", "''")`. Prefer parameterized queries.

---

## Testing

- **Unit tests must never call the Datum proxy or PostgreSQL.** Mock `DatumDBClient.db_sql()` or the equivalent client method.
- The effective interface contract for any DB client mock:
  ```python
  def db_sql(self, sql: str) -> dict:
      return {"data": [  # list of row dicts
          {"col": "value", ...},
      ]}
  ```
- Integration and E2E tests may use a real proxy instance; mark them explicitly (e.g., `@pytest.mark.integration`) so they can be excluded from fast CI runs.
