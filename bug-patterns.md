# Bug & Quality Pattern Catalog

Check every changed file for these patterns. For each match, report the exact `file:line`, the code, why it's a problem, and a concrete fix.

## Contents

1. [Style & convention adherence](#1-style--convention-adherence)
2. [Glue: happy-path, on-point complexity](#2-glue-happy-path-on-point-complexity)
3. [Airflow best practices](#3-airflow-best-practices)
4. [NULL handling gaps](#4-null-handling-gaps)
5. [Position-dependent INSERT](#5-position-dependent-insert)
6. [JSON extraction without type guards](#6-json-extraction-without-type-guards)
7. [Schema mismatches between layers](#7-schema-mismatches-between-layers)
8. [Snapshot check_cols mismatch](#8-snapshot-check_cols-mismatch)
9. [Airflow DAG dependency misalignment](#9-airflow-dag-dependency-misalignment)
10. [Glue field extraction drift](#10-glue-field-extraction-drift)
11. [Appendix: Athena v2 vs Trino syntax](#appendix-athena-v2-vs-trino-syntax)

## 1. Style & convention adherence

**The repo's own conventions are the primary rubric.** Before critiquing anything else, load:

- `CLAUDE.md` at the repo root — plus any nested `CLAUDE.md` in the directories touched by the PR.
- `CONTRIBUTING.md` — team conventions, PR checklist, naming rules.
- The nearest existing sibling files to each changed file — use them as the implicit style reference for anything `CLAUDE.md` / `CONTRIBUTING.md` does not cover.

**Check for**:
- Naming conventions (model prefixes, column casing, file layout) that differ from neighboring files.
- Macro or config patterns the repo already uses, but the PR ignores (e.g. rolling its own logic where a shared macro exists).
- Departures from documented conventions — flag these with a direct quote from the relevant guide.

When the PR conflicts with a convention, **cite the convention source** (`CLAUDE.md:line` or the sibling file) in the finding so the author can see the precedent.

## 2. Glue: happy-path, on-point complexity

Glue jobs in this repo should do one thing: extract → transform minimally → write. Flag complexity that doesn't earn its keep, and flag code that's too thin to meet the contract.

**Over-engineering red flags** (most common — LLM-generated code tends here):
- Retry loops, circuit breakers, or custom exception hierarchies around operations Glue/Spark already retry.
- Config indirection for values that only have one production setting (e.g. environment-selector that only ever picks prod).
- Parameterized transforms where a straight-line script would read more clearly.
- Defensive `try/except` wrapping every stage — let the job fail loudly and let Airflow handle it.
- Custom logging frameworks on top of Glue's built-in logger.
- Caching / memoization inside a single-pass ETL.

**Under-engineering red flags**:
- Drops fields the downstream base model selects.
- Skips obvious transformations (type coercion, timestamp normalization) that every sibling Glue job performs.
- No partitioning / no write-mode configured, when neighbors have it.
- Writes to landing without the expected schema registration call.

**How to evaluate**: compare the changed Glue job line-for-line against the nearest existing Glue job that serves a similar source. Call out anything that's meaningfully more elaborate or meaningfully sparser.

## 3. Airflow best practices

**No hardcoding.** Airflow has first-class primitives for every value that changes between environments or runs.

Check for:
- **Hardcoded credentials / endpoints** — must use `Connection` via `BaseHook.get_connection()` or the operator's `*_conn_id` argument.
- **Hardcoded dates** — use `{{ ds }}`, `{{ data_interval_start }}`, `{{ logical_date }}` templated fields, or `pendulum` with `airflow.utils.dates`. Never `datetime.now()` or a literal `"2024-01-01"` in task logic.
- **Hardcoded paths / table names that vary by env** — use `Variable.get()` or Jinja macros.
- **Secrets in DAG files** — must come from Connections / Variables / the configured secrets backend.

**Pattern adherence**:
- Task IDs, DAG IDs, and ownership follow the naming convention used by sibling DAGs.
- `default_args` matches the repo's standard (owner, retries, retry_delay, SLA, email_on_failure).
- `catchup`, `schedule`, `start_date` values are consistent with other DAGs in the same domain.
- Uses the same operators the repo already uses (don't introduce a new `KubernetesPodOperator` pattern if the repo standardizes on `GlueJobOperator`).
- No top-level Python that calls the database, the network, or reads large files — DAG parsing must stay fast.
- No dynamic task generation inside a Python callable that runs at runtime; if dynamic, use `expand()` / task mapping.

## 4. NULL Handling Gaps

Data from source systems often contains empty strings `''` instead of `NULL`. Casting empty strings to numeric/date types fails or silently drops rows.

```sql
-- BAD
CAST(field AS INTEGER)
CAST(field AS DATE)

-- GOOD
CAST(NULLIF(TRIM(field), '') AS INTEGER)
CAST(NULLIF(TRIM(field), '') AS DATE)
```

**Also check**:
- `COALESCE` with wrong default type (e.g. `COALESCE(int_col, 'N/A')`).
- `NVL` usage — not standard SQL here; use `COALESCE`.
- Missing `WHERE field IS NOT NULL` before aggregations that don't handle nulls.

## 5. Position-Dependent INSERT

`INSERT INTO table VALUES (...)` without an explicit column list breaks silently when the table DDL gains a column.

```sql
-- BAD
INSERT INTO raw.events VALUES ('2024-01-01', 'click', 42)

-- GOOD
INSERT INTO raw.events (event_date, event_type, event_count)
VALUES ('2024-01-01', 'click', 42)
```

## 6. JSON Extraction Without Type Guards

`json_extract` / `json_extract_scalar` on polymorphic fields (external payloads) returns `NULL` silently when the shape changes.

```sql
-- BAD: assumes payload.value is always a scalar
json_extract_scalar(payload, '$.value')

-- GOOD: branch on shape
CASE
  WHEN json_format(json_extract(payload, '$.value')) LIKE '"%'
    THEN json_extract_scalar(payload, '$.value')
  WHEN json_format(json_extract(payload, '$.value')) LIKE '{%'
    THEN json_extract_scalar(payload, '$.value.amount')
  ELSE NULL
END
```

Flag every `json_extract_scalar` on fields sourced from external systems.

## 7. Schema Mismatches Between Layers

Column names in one layer don't match what the next layer expects.

- Base model `SELECT` columns vs what staging references.
- Staging output vs intermediate joins.
- Any `ref('model_name')` — read the referenced model and verify the columns exist.
- `SELECT *` in downstream models — schema changes propagate silently; flag every occurrence.
- Column aliases that shadow upstream names.

## 8. Snapshot check_cols Mismatch

dbt snapshot `check_cols` list doesn't match the model's actual output columns.

```sql
{% snapshot snap_customers %}
  {{ config(strategy='check', check_cols=['name', 'email', 'status']) }}
  SELECT * FROM {{ ref('staging_customers') }}
{% endsnapshot %}
```

If `staging_customers` renames `status` → `account_status`, the snapshot silently stops tracking that column. Read the snapshot config, then read the source model, and verify every `check_cols` entry exists.

## 9. Airflow DAG Dependency Misalignment

Task dependencies must match the data flow.

- If model B `{{ ref('model_a') }}`s model A, B's Airflow task must depend on A's task.
- Verify `>>` / `set_downstream` / `set_upstream` wiring matches the `ref()` graph.
- Missing dependencies mean B reads stale data from a previous run.

## 10. Glue Field Extraction Drift

Glue job extracts fields; base dbt model SELECTs them. They must agree.

- Read the Glue job — find field mapping / extraction logic.
- Read the base model — find the `SELECT` column list.
- Every field the base model expects must be extracted by the Glue job.
- If the PR changes field names on either side, verify the other side matches.

## Appendix: Athena v2 vs Trino syntax

Only relevant when the PR is explicitly about cross-engine portability or when a query is known to run on a mixed environment. Don't lead a review with these.

Red flags when they appear:
- `parse_datetime(...)` — Trino-only; fails on Athena v2.
- `date_parse(x, fmt)` — format-code semantics diverge.
- `from_unixtime(ts)` — return type differs across versions.
- `element_at(arr, idx)` — 1-based on Trino.
- `date_diff('unit', a, b)` — argument order differs.
- `CAST(x AS VARCHAR)` — Trino may require length.

Check which engine the model runs on before flagging.
