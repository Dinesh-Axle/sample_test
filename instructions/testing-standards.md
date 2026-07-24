# Testing Standards — Copilot Instructions

Extends [engineering-standards](../engineering-standards/copilot-instructions.md). Rules here apply to all repositories. Stack-specific instruction files may add to these but must not contradict them.

---

## Principles

- **Test behavior, not implementation.** Tests should break when observable behavior changes, not when internal structure is refactored.
- **Every test must cover unique behavior.** No redundant tests. If two tests would always pass or fail together, merge them.
- **Tests are the first consumer of your API.** If a function is hard to test, the abstraction boundary is probably wrong.

---

## Framework

All projects use **pytest**. Add a `pytest.ini` (or `[tool.pytest.ini_options]` in `pyproject.toml`) at the repo root:

```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = -v --tb=short
markers =
    integration: marks tests that require a real backend (deselect with -m "not integration")
    e2e: marks end-to-end browser or full-stack tests (deselect with -m "not e2e")
```

---

## Directory Structure

`tests/` lives at the repo root in parallel with `src/` and mirrors its structure exactly. If a source file moves, its test file moves with it.

```
<repo-root>/
  src/
    models/
      user_model.py
    utils/
      sql_builder.py
  tests/
    conftest.py                        # Shared fixtures only — no test logic
    models/
      test_user_model.py
    utils/
      test_sql_builder.py
    integration/
      test_<feature>_integration.py
    e2e/
      test_<feature>_e2e.py
```

---

## Naming Conventions

| Thing | Convention | Example |
|-------|-----------|---------|
| Test file | `test_<module>.py` | `test_sql_builder.py` |
| Test class | `Test<ConceptOrClass>` | `TestSQLBuilder` |
| Test method | `test_<action>_<subject>` | `test_blocks_destructive_sql` |
| Integration file | `test_<feature>_integration.py` | `test_upload_flow_integration.py` |

- Each test method has a single-line docstring describing the unique behavior it verifies.
- Test classes are organizational only — no shared state between tests via class attributes. Use fixtures instead.

---

## Test Priority Order

Write tests in this sequence for every new feature or function, at each layer. Do not skip ahead — later tiers depend on earlier ones passing first.

| Priority | Name | What to verify |
|----------|------|----------------|
| 1 | **Happy path** | Expected inputs produce correct output. Cover the primary use case first. |
| 2 | **Sad path** | Invalid inputs, missing data, failed API calls — verify graceful error handling, correct status codes, and meaningful error messages. |
| 3 | **Edge cases** | Boundary values, empty collections, single-element lists, max-length strings, `None`/null fields. |
| 4 | **Destructive** | Concurrent writes, duplicate submissions, malformed payloads, unexpected types — confirm the system does not corrupt state. |

A feature is not considered tested until all four tiers have coverage.

---

## Test Categories

Every module or feature must have coverage across three categories. A test suite may span multiple files — organize by category, not by source file alone. Apply the priority order above within each category.

### Unit Tests
- Scope: a single function or class in isolation.
- All external I/O is mocked.
- Fast — must complete in milliseconds.
- Cover: all validation paths (success and error), default values, and edge cases that represent real failure modes.

### Integration Tests
- Scope: two or more functions or components working together.
- Use real class instances and real config objects. Mock only true I/O boundaries (network, DB, filesystem).
- For functions that execute SQL, use an in-memory SQLite backend rather than mocking the query itself:
  ```python
  class SQLiteDBClient:
      """Wraps SQLite to match the db_sql() interface."""
      def __init__(self, conn):
          self.conn = conn

      def db_sql(self, sql: str) -> dict:
          cursor = self.conn.cursor()
          cursor.execute(sql)
          return {"data": [dict(r) for r in cursor.fetchall()]}
  ```
- Verify that constraints are enforced (UNIQUE, PRIMARY KEY), retry logic behaves correctly, and state mutations propagate across the chain.
- Mark with `@pytest.mark.integration` so they can be excluded from fast CI runs.

### End-to-End Tests
- Scope: a full user-facing feature flow against a real or realistic backend.
- May use real databases, real proxy instances, or any appropriate tooling for the stack.
- Mark with `@pytest.mark.e2e`. Never run in unit CI — only in dedicated E2E pipelines or manually.

---

## What to Mock

Mock only the boundary between your code and the outside world:

| ✅ Mock | ❌ Do not mock |
|--------|---------------|
| Network calls (HTTP, JDBC) | Your own domain classes |
| Database queries | Config objects (`TableSpec`, `AppConfig`, etc.) |
| File system operations | Validation logic |
| External APIs | Pydantic models |
| Platform I/O (Spark, S3) | Fixtures and test helpers |

Heavy mocking of internal objects creates tests that pass while missing real structural bugs — wrong class, wrong method, wrong attributes. Use real instances; mock the edges.

---

## Fixtures

- **Shared fixtures belong in `conftest.py`**, not duplicated across test files.
- **Fixtures provide pre-configured objects** — mock clients, sample DataFrames, temp directories. They do not assert anything.
- Name fixtures for what they represent, not how they work: `sample_request`, not `mock_dict_with_request_fields`.
- Common fixtures every project should define:

  ```python
  @pytest.fixture
  def mock_client():
      """Mock API/DB client with environment set to 'dev'."""
      client = MagicMock()
      client.environment = "dev"
      return client

  @pytest.fixture
  def mock_env_vars():
      """Patch environment variables for the duration of a test."""
      env = {"SERVICE_KEY": "test-token", "ENV": "test"}
      with patch.dict(os.environ, env, clear=False):
          yield env
  ```

---

## Dependency Stubbing

When a package is unavailable in the test environment (e.g., `dmapTableEditor`, `shiny`, `sqlalchemy` in lightweight CI), stub it via `sys.modules` **before** importing any application module that depends on it:

```python
import sys
from unittest.mock import MagicMock

sys.modules["shiny"] = MagicMock()
sys.modules["dmapTableEditor"] = MagicMock()
sys.modules["dmapTableEditor.widgets"] = MagicMock()

# Application imports come after stubs
from src.server import my_module  # noqa: E402
```

- Do not install unavailable packages into the test environment to avoid this — the stub pattern is intentional.
- Place stubs at the top of the test file, before any application imports, with a comment explaining why the stub is needed.

---

## What Not to Test

- **Third-party library behavior.** Don't test that `pandas.merge()` merges — test that your function calls it correctly.
- **Trivial variations.** If tests differ only in input values, use `@pytest.mark.parametrize`:
  ```python
  @pytest.mark.parametrize("status", ["pending", "uploaded", "verified"])
  def test_valid_status_accepted(status):
      assert is_valid_status(status) is True
  ```
- **Private implementation details.** Test the public interface. Internal helpers are covered implicitly by the functions that call them.

---

## Coverage

- Run coverage against the source package, not the test directory:
  ```bash
  pytest --cov=src --cov-report=term-missing
  ```
- Coverage gaps are acceptable for: deployment entry points (`app.py`, `main.py`), pure config files, and generated code. Document exceptions with an inline `# pragma: no cover` comment.
- **Coverage does not equal correctness.** A line being executed is not evidence it was tested meaningfully.

---

## Running Tests

```bash
# All tests
pytest

# Unit tests only
pytest -m "not integration and not e2e"

# Single file
pytest tests/utils/test_sql_builder.py

# By name pattern
pytest -k "test_blocks_destructive"

# With coverage
pytest --cov=src --cov-report=term-missing

# Fail fast — stop on first failure
pytest -x
```

---

## Manual Test Plans (XLSX)

Create a manual test plan for every PR with UI-facing changes. Store the build script and generated `.xlsx` in `dmap-docs/prompts/<project>/manual-tests/`.

### Column Schema

| Column | Purpose |
|--------|---------|
| Test ID | Short unique ID (e.g., `SS-01`, `PAG-03`) |
| Priority | `P0` (blocker), `P1` (important), `P2` (nice-to-have) |
| Type | Test path — see color coding below |
| Module / Commit(s) | Affected module or commit reference |
| Change Summary | One-line description of what changed |
| Pre-conditions | State required before running the test |
| Steps | Numbered steps to reproduce |
| Expected Result | What should happen |
| Actual Result | Filled in by tester |
| Status | Pass / Fail / Blocked / Skipped |
| Tester | Who ran the test |
| Notes | Additional observations |

### Row Color Coding by Test Type

Apply a **full-row** background fill based on the **Type** column. This makes it easy to scan sheets for coverage gaps across test paths.

| Type | Color | Hex | What it verifies |
|------|-------|-----|-----------------|
| **Happy** | Light green | `#C6EFCE` | Expected inputs produce correct output |
| **Sad** | Light orange | `#FDE9D9` | Invalid/missing data → graceful error handling |
| **Edge** | Light yellow | `#FFFFCC` | Boundary values, empty states, single-element lists |
| **Regression** | Light purple | `#E4DFEC` | Unchanged behavior still works after refactor |
| **Destructive** | Light red | `#F4CCCC` | Concurrent writes, stale state, session corruption |

### Build Script Pattern

Each plan has a Python build script (`build_manual_test_plan_pr<N>.py`) that generates the XLSX using `openpyxl`. This keeps test data version-controlled and reproducible. Use `TYPE_FILLS` dict for color mapping:

```python
from openpyxl.styles import PatternFill

TYPE_FILLS = {
    "Happy":      PatternFill("solid", fgColor="C6EFCE"),
    "Sad":        PatternFill("solid", fgColor="FDE9D9"),
    "Edge":       PatternFill("solid", fgColor="FFFFCC"),
    "Regression": PatternFill("solid", fgColor="E4DFEC"),
    "Destructive":PatternFill("solid", fgColor="F4CCCC"),
}
```
