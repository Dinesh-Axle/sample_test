# PyShiny — Copilot Instructions

Extends [engineering-standards](../engineering-standards/copilot-instructions.md). Rules here apply to every repository built on Shiny for Python.

---

## Module Structure

- **UI modules are thin.** Each returns only `ui.nav_panel(title, component(...))`. Heavy layout logic lives in a `components/` subpackage, not inline.
- **Server modules expose a single registration function:**
  ```python
  def register_feature(input, output, session, shared):
      ...
  ```
  All reactive effects, outputs, and event handlers are wired inside this function. Nothing is registered at module level.
- `app.py` is an orchestrator only — it assembles modules and passes shared state. No business logic or data transformation belongs there.

---

## Reactive Patterns

| Need | Use |
|------|-----|
| Derived / computed data (cached) | `@reactive.calc` |
| Side effects, button handlers | `@reactive.effect` + `@reactive.event` |
| Table outputs | `@render.data_frame` |
| Dynamic UI | `@render.ui` |
| Guard missing/invalid input | `req()` |

- **Invalidation uses integer counter increments**, not direct UI update calls:
  ```python
  shared["refresh_trigger"].set(shared["refresh_trigger"].get() + 1)
  ```
- Never call UI update methods to force re-renders. Increment a `reactive.Value(int)` counter that dependent calcs already read.

---

## Shared State

- **All cross-module state flows through `shared`** — a `dict` of `reactive.Value` objects created in `server()` and passed by reference to every `register_*()` function.
- **No module-level mutable state for per-session data.** Module-level variables are acceptable only for cross-session read-only caches (e.g., reference data loaded once at startup).

---

## Tab Loading

- **Only the first tab initializes on app load.** All other tabs hydrate on demand:
  ```python
  @reactive.effect
  def _():
      if input.active_tab() == "my_tab":
          # initialize this tab's data and outputs
  ```
  This prevents unnecessary database queries on startup.

---

## Testing

- **Stub heavy dependencies before importing the module under test.** Packages unavailable in CI (`dmapTableEditor`, `s3service`, `sqlalchemy`, `shiny`) must be injected via `sys.modules`:
  ```python
  import sys
  from unittest.mock import MagicMock
  sys.modules["shiny"] = MagicMock()
  sys.modules["dmapTableEditor"] = MagicMock()

  # Only import application modules after stubs are in place
  from src.server import my_module
  ```
- Do not add CI-unavailable packages to test requirements to avoid this — the stub pattern is intentional.
- Use `@patch("src.server.<module>.datum")` to mock data access at the point of use, not the definition site.

---

## Deployment

- Deploy via `rsconnect deploy shiny -n <target> .`
- Run QC checks (linting, tests, coverage) before deploying. A `deploy.sh` script should orchestrate this — never deploy directly without running checks first.
- User identity in production comes from the `Shiny-Server-Credentials` HTTP header injected by Posit Connect. Always provide a local fallback (e.g., `local-dev@nih.gov`) for `DEBUG` mode.
