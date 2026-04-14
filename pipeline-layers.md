# Pipeline Layer Classification

Use these rules to classify each changed file into a pipeline layer. Match top-to-bottom; first match wins.

## Layer Rules

### Landing DDL
- **Pattern**: `**/landing/**/*.sql`, `**/landing/**/*.ddl`
- **Content signal**: `CREATE EXTERNAL TABLE`, `LOCATION 's3://...landing...'`
- **Role**: Schema definitions for raw data landing zones (data arrives here from source systems)

### Raw DDL/DML
- **Pattern**: `**/raw/**/*.sql`, `**/raw_vault/**/*.sql`
- **Content signal**: `CREATE TABLE`, `INSERT INTO`, references to landing tables
- **Role**: Raw data layer — first persistent store after landing, minimal transformation

### Base (dbt)
- **Pattern**: `**/models/base/**/*.sql`, `**/models/sources/**/*.sql`
- **Content signal**: `{{ source(`, column renaming, basic type casting
- **Role**: First SQL layer over raw sources — 1:1 mapping, renaming, type casting only

### Staging (dbt)
- **Pattern**: `**/models/staging/**/*.sql`, `**/models/stg_*.sql`
- **Content signal**: `{{ ref('base_`, cleaning logic, deduplication
- **Role**: Cleaned and standardized from base — business-friendly names, null handling, dedup

### Intermediate (dbt)
- **Pattern**: `**/models/intermediate/**/*.sql`, `**/models/int_*.sql`
- **Content signal**: Multiple `{{ ref(` joins, business logic, CTEs
- **Role**: Joins and business logic — combines multiple staging models

### Refined (dbt)
- **Pattern**: `**/models/refined/**/*.sql`, `**/models/ref_*.sql`
- **Content signal**: Aggregations, enrichments, complex business rules
- **Role**: Enriched and aggregated — business metrics, KPIs, derived fields

### Mart (dbt)
- **Pattern**: `**/models/mart/**/*.sql`, `**/models/marts/**/*.sql`, `**/models/dim_*.sql`, `**/models/fct_*.sql`
- **Content signal**: Final SELECT for consumers, wide denormalized tables
- **Role**: Consumer-facing tables — dashboards, reports, APIs read from these

### Snapshot (dbt)
- **Pattern**: `**/snapshots/**/*.sql`
- **Content signal**: `{% snapshot`, `strategy=`, `check_cols=`
- **Role**: Slowly changing dimension tracking — captures historical state

### Glue
- **Pattern**: `**/glue/**/*.py`, `**/glue_jobs/**/*.py`, `**/etl/**/*.py`
- **Content signal**: `from awsglue`, `GlueContext`, `DynamicFrame`, field extraction logic
- **Role**: AWS Glue ETL jobs — extract/transform data between landing and raw layers

### DAG (Airflow)
- **Pattern**: `**/dags/**/*.py`, `**/airflow/**/*.py`
- **Content signal**: `from airflow`, `DAG(`, `@task`, `PythonOperator`, `BashOperator`
- **Role**: Airflow DAG definitions — orchestrate pipeline execution order

### Config (dbt)
- **Pattern**: `**/*sources*.yml`, `**/*schema*.yml`, `**/dbt_project.yml`, `**/*properties*.yml`
- **Content signal**: `sources:`, `models:`, `columns:`, `tests:`, `description:`
- **Role**: dbt project config, source definitions, schema/column documentation, test definitions

### Test (dbt)
- **Pattern**: `**/tests/**/*.sql`, `**/tests/**/*.yml`, `**/macros/test_*.sql`
- **Content signal**: `SELECT` with failure conditions, `{{ test(`, custom test macros
- **Role**: Data quality tests — assertions that must pass for pipeline health

### Other
- **Pattern**: Everything not matching above
- **Examples**: `*.md`, `*.txt`, `.github/**`, `Makefile`, `requirements.txt`, `*.cfg`
- **Role**: Documentation, CI/CD, tooling — no pipeline tracing needed

## Classification Instructions

For each changed file in the PR:

1. Check the file path against the patterns above (top to bottom, first match wins)
2. If the path is ambiguous, read the file content and check for content signals
3. Record the layer assignment — you will use this to decide which tracing steps apply
4. Files classified as **Other** do not need upstream/downstream tracing — review them for general code quality only
