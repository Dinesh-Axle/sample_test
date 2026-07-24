# dmapTableEditor Integration — Copilot Instructions

Extends [engineering-standards](../engineering-standards/copilot-instructions.md) and [pyshiny.instructions.md](./pyshiny.instructions.md). Rules here apply to every repository that embeds the `dmapTableEditor` widget.

---

## Dependency Pinning

- **Always pin to a specific commit SHA.** Never reference `main`, a branch name, or a version range.
  ```
  # requirements.txt
  datumTableEditor @ git+https://...@<commit-sha>
  ```
- **Do not upgrade without verifying compatibility.** Check the widget's changelog and run the full test suite before updating the SHA.

---

## Configuration Is the Source of Truth

- **All table behavior is defined in the JSON config file** for each widget instance. This includes: database connection, column visibility, editability, permissions, approval workflow, status labels, pagination, filtering, and sorting.
- **Never hardcode table schemas, filters, column names, or queries in the host application.** If behavior needs to change, change the config — not the Python code.
- Feature flags (`enable_*`) control opt-in behavior. New features are enabled via config, not code changes. All flags must be explicitly set — do not rely on defaults.

---

## Column Masks

- **Every user-facing column must have a display-name entry in `column_masks`** mapping the snake_case database column to a human-readable label.
- Never expose raw database column names in the UI.

```json
"column_masks": {
  "patient_id": "Patient ID",
  "date_of_collection": "Collection Date"
}
```

---

## Data Access Pattern

- **Tables rendered by the widget are always readonly.** The widget does not write directly to source tables.
- All edits flow through a dedicated `_modifications` table. Each widget instance has its own modifications table — never share one across instances.
- Each widget instance also has its own `_ui_state` table for persisting user interaction state (column visibility, sort order, etc.).

---

## Wiring the Widget

```python
# app.py — keep this the entire integration surface
from dmapTableEditor.widgets import table_editor_ui, table_editor_server

app_ui = ui.page_fluid(
    table_editor_ui("editor1", config_path="app_config.json")
)

def server(input, output, session):
    table_editor_server("editor1", config_path="app_config.json")

app = App(app_ui, server)
```

- **`app.py` must remain thin.** Do not add data transformation, direct database calls, or business logic to the host app. Everything belongs inside the widget package.
- Module IDs (e.g., `"editor1"`) are string literals passed directly — not constants or variables.

---

## Testing

- **Stub `dmapTableEditor` via `sys.modules` before importing any application module** that depends on it. See [pyshiny.instructions.md](./pyshiny.instructions.md) for the stub pattern.
- Host application tests should cover: config loading, widget wiring, and permission enforcement. Widget-internal logic is tested in the `dmapTableEditor` package itself — do not duplicate those tests here.
