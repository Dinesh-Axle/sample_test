# Spark Pipelines — Copilot Instructions

Extends [engineering-standards](../engineering-standards/copilot-instructions.md). Rules here apply to every repository running Apache Spark / PySpark transformation pipelines orchestrated by Apache Airflow.

---

## Execution Environment

- **Always run Spark tasks inside Docker containers** via Airflow's `DockerOperator`. Never run `spark-submit` directly against a cluster from a host machine.
- **Inter-service communication requires the Docker network.** Never call service APIs (Trino, Hive, Spark, Marquez) from the host. Use `docker exec <container> <command>` for any command that needs network access.
- **Run tests inside Docker**, not with a local `pytest` or `pip install`:
  ```bash
  bash tests/run-tests-in-docker.sh [unit|integration|all] [pytest-args...]
  ```
  The first argument must be the suite name. All remaining arguments pass directly to pytest.

---

## SDK-First I/O

- **Never use raw `spark.read` or `spark.write`.** All table reads and writes go through the project SDK abstraction (e.g., `DataPipeline`, `read_table`, `write_table`).
- **Always use `DataPipeline.from_config()` and `load_table_config()`** for pipeline setup. Table keys in Python must match keys in the companion `*_tables.yaml`.

---

## Script Structure

Every transformation script follows this pattern:

```python
def compute(input_df):
    # Pure transformation logic only — no I/O
    ...
    return result_df

def main(argv):
    # Argument parsing, pipeline setup, read → compute → write only
    ...
```

- `compute()` must be a **pure function**: DataFrame in, DataFrame out. No reads, writes, or side effects.
- `main()` handles only argument parsing, pipeline setup, and orchestrating `read → compute → write`.
- Keep transformation logic out of `main()`; keep I/O out of `compute()`.

---

## Imports

```python
from pyspark.sql import functions as F, types as T
from dmap_data_sdk import DataPipeline
from dmap_data_sdk.table_config import load_table_config
```

---

## Pipeline Definition (`pipeline.yaml`)

- `pipeline.yaml` is the **authoritative DAG definition**. Do not replicate its dependency logic in code.
- Every task requires: a unique `id`, an optional `depends_on` list, and an `execution.command`.
- Respect the dependency chain when adding tasks — verify the full DAG before submitting.

---

## Data Lineage Stages

All data domains follow a strict 5-stage lineage. Use the correct prefix at each stage:

| Stage | Prefix | Description |
|-------|--------|-------------|
| Raw | `dbo_*` / `mdl_lims_*` | Ingested from source, no transformation |
| Clean | `clean_*` | Standardized columns, types, naming |
| Processed | `processed_*` | Business logic applied |
| Export | `export_*` | Final renames, unions, placeholder columns |
| Write | *(task suffix `_postgres`)* | JDBC write to PostgreSQL |

---

## Column Naming

- All column names use **snake_case**. Enforce via `snakecase_cols()` / `clean_df()` at the clean stage.
- Rename ambiguous ID columns explicitly: `id` → `request_id`, `id` → `sample_id`, etc.
- Use `sex` rather than `gender` for biological sex fields.

---

## Debugging

| Symptom | Action |
|---------|--------|
| Exit code 137 (OOM) | Increase `resources.memory` or reduce `concurrency` in pipeline config |
| Module not found | Add `--py-files` for custom modules (e.g., `utils.zip`) |
| Log4j not working | Ensure `./log4j2.properties` is at the artifact root |
