# Bug Pattern Catalog

Check every changed SQL/Python file for these patterns. For each match, report the exact file:line, the problematic code, why it's a problem, and a suggested fix.

## 1. SQL Function Compatibility (Athena vs Trino)

Athena v2 uses Presto syntax; Athena v3 / Trino has breaking changes.

**Pattern**: Look for functions that differ between engines.

| Athena v2 (Presto) | Trino / Athena v3 | Fix |
|---------------------|-------------------|-----|
| `date_parse(x, fmt)` | `date_parse(x, fmt)` (same, but watch format codes) | Verify format string matches engine |
| `from_unixtime(ts)` | `from_unixtime(ts)` | Check if return type changed (timestamp vs varchar) |
| `json_extract(col, '$.path')` | `json_extract(col, '$.path')` | Verify JSON path syntax matches engine |
| `CAST(x AS VARCHAR)` | `CAST(x AS VARCHAR)` | Check `VARCHAR` vs `VARCHAR(n)` — Trino may require length |
| `element_at(arr, idx)` | `element_at(arr, idx)` | Trino uses 1-based indexing; verify |
| `date_diff('day', a, b)` | `date_diff('day', a, b)` | Check argument order (some engines swap start/end) |
| `approx_distinct(col)` | `approx_distinct(col)` | Precision parameter may differ |

**Red flag**: Any `parse_datetime` call — this is Trino-only, will fail on Athena v2.

## 2. NULL Handling Gaps

**Pattern**: Data arriving from source systems often contains empty strings `''` instead of `NULL`. Casting empty strings to numeric/date types fails silently or throws errors.

**Check for**:
```sql
-- BAD: Empty string cast fails
CAST(field AS INTEGER)
CAST(field AS DATE)
CAST(field AS DOUBLE)

-- GOOD: Nullify empty strings first
CAST(NULLIF(TRIM(field), '') AS INTEGER)
CAST(NULLIF(TRIM(field), '') AS DATE)
```

**Also check**:
- `COALESCE` with wrong default type (e.g., `COALESCE(int_col, 'N/A')`)
- `NVL` usage (not standard SQL — use `COALESCE`)
- Missing `WHERE field IS NOT NULL` before aggregations that don't handle nulls

## 3. Position-Dependent INSERT

**Pattern**: `INSERT INTO table VALUES (...)` without an explicit column list.

```sql
-- BAD: Column order depends on table DDL — breaks silently if DDL changes
INSERT INTO raw.events VALUES ('2024-01-01', 'click', 42)

-- GOOD: Explicit column list — survives DDL reordering
INSERT INTO raw.events (event_date, event_type, event_count)
VALUES ('2024-01-01', 'click', 42)
```

**Why it breaks**: If someone adds a column to the table, the INSERT shifts all values right, putting data in wrong columns with no error.

## 4. JSON Extraction Without Type Guards

**Pattern**: `json_extract` or `json_extract_scalar` on polymorphic fields that can be different JSON types.

```sql
-- BAD: Assumes payload.value is always a string
json_extract_scalar(payload, '$.value')

-- PROBLEM: If value is sometimes an object {"amount": 10}, this returns NULL silently
-- or if value is an array [1,2,3], same issue

-- GOOD: Check type first or handle both cases
CASE
  WHEN json_format(json_extract(payload, '$.value')) LIKE '"%'
    THEN json_extract_scalar(payload, '$.value')
  WHEN json_format(json_extract(payload, '$.value')) LIKE '{%'
    THEN json_extract_scalar(payload, '$.value.amount')
  ELSE NULL
END
```

**Check for**: Any `json_extract_scalar` on fields that come from external systems — these are frequently polymorphic.

## 5. Schema Mismatches Between Layers

**Pattern**: Column names in one layer don't match what the next layer expects.

**Check**:
- Base model `SELECT` columns vs what staging model references
- Staging model output vs what intermediate model joins on
- Any `ref('model_name')` usage — read the referenced model and verify the columns exist

```sql
-- In staging_orders.sql
SELECT order_id, customer_id, total_amount FROM {{ ref('base_orders') }}

-- But base_orders.sql actually outputs: order_id, cust_id, amount
-- This query FAILS: customer_id and total_amount don't exist
```

**Also check**:
- `SELECT *` in downstream models — if upstream adds/removes/reorders columns, `SELECT *` silently changes
- Column aliases that shadow upstream names

## 6. Snapshot check_cols Mismatch

**Pattern**: dbt snapshot `check_cols` list doesn't match the model's actual output columns.

```sql
-- In snapshots/snap_customers.sql
{% snapshot snap_customers %}
  {{ config(
    strategy='check',
    check_cols=['name', 'email', 'status']
  ) }}
  SELECT * FROM {{ ref('staging_customers') }}
{% endsnapshot %}

-- If staging_customers renames 'status' to 'account_status',
-- the snapshot silently stops tracking status changes
```

**Check**: Read the snapshot config, then read the source model — verify every `check_cols` entry exists in the model output.

## 7. Airflow DAG Dependency Misalignment

**Pattern**: Airflow task dependencies don't match the actual data flow.

**Check**:
- If model B depends on model A (`{{ ref('model_a') }}`), then the Airflow task for B must depend on the task for A
- Look for `>>` or `set_downstream`/`set_upstream` operators
- Missing dependencies mean B might run before A finishes, reading stale data

## 8. Glue Field Extraction Drift

**Pattern**: AWS Glue job extracts fields from source data, but the field names don't match what the base dbt model expects.

**Check**:
- Read the Glue job script — find field mapping/extraction logic
- Read the corresponding base model — find the `SELECT` column list
- Verify every field the base model expects is extracted by the Glue job
- If the PR changes field names in either the Glue job or the base model, check the other side matches
