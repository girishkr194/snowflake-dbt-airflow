# Day 1 — dbt Introduction and Setup

## What You Will Learn

1. **What is dbt?** — Where it fits in the modern data stack and what problem it solves
2. **dbt Core vs dbt Cloud** — Differences and when to use each
3. **Installation** — Install dbt Core with the Snowflake adapter
4. **Project setup** — `dbt init`, project structure, `profiles.yml`
5. **First model** — Write a SQL model, run it, understand what dbt does behind the scenes
6. **Sources and refs** — Declare raw tables as sources, reference models with `ref()`

---

## Concept 1 — What is dbt?

### The Problem dbt Solves

In a traditional data pipeline you have two separate worlds:
- **Data Engineers** load raw data into the warehouse (EL — Extract, Load)
- **SQL Analysts** write SELECT queries to build reports

The gap: analysts write ad-hoc SQL with no version control, no testing, no documentation, no dependency tracking. When something breaks, nobody knows which query depends on what.

**dbt fills this gap.** It is a transformation framework that lets analysts and engineers write SQL models as `.sql` files, version-control them with Git, test them, document them, and run them in dependency order — all inside the warehouse.

```
Raw Data (S3 / ADLS / GCS)
        │
        ▼
   [Extract & Load]  ← Fivetran, Airbyte, Snowpipe
        │
        ▼
  Warehouse (Snowflake)
  ┌─────────────────────────┐
  │  ECOMM_DB.RAW_DATA      │  ← source tables (loaded by EL tools)
  │  ECOMM_DB.STAGING       │  ← dbt models (clean, rename, cast)
  │  ECOMM_DB.MARTS         │  ← dbt models (business logic, aggregation)
  └─────────────────────────┘
        │
        ▼
   [BI Tool]  ← Tableau, Power BI, Looker
```

dbt only handles the **T** in ELT. It does not move data — it transforms data already in the warehouse.

### What dbt Actually Does

When you run `dbt run`, dbt:
1. Reads your `.sql` model files
2. Wraps each one in a `CREATE TABLE AS` or `CREATE VIEW AS` statement
3. Runs them in dependency order
4. Reports success/failure per model

You write plain `SELECT` statements. dbt handles the `CREATE OR REPLACE` boilerplate.

---

## Concept 2 — dbt Core vs dbt Cloud

| | dbt Core | dbt Cloud |
|---|---|---|
| What it is | Open-source CLI tool | Managed SaaS platform built on dbt Core |
| Cost | Free | Free tier + paid plans |
| Interface | Terminal + your IDE | Web IDE + scheduler + CI/CD |
| Scheduling | External (Airflow, cron) | Built-in scheduler |
| Best for | Engineers comfortable with CLI and Git | Teams wanting a managed, GUI-driven setup |

**This course uses dbt Core** — you install it locally, connect to Snowflake, and run everything from the terminal.

---

## Concept 3 — Installation

### Prerequisites

- Python 3.8+
- pip
- Snowflake account with `ECOMM_DB` already created (from the Snowflake module)

### Install dbt Core + Snowflake adapter

```bash
pip install dbt-snowflake
```

```bash
dbt --version
```

This installs two packages: `dbt-core` (the dbt engine) and `dbt-snowflake` (the Snowflake adapter that translates dbt commands into Snowflake SQL).

---

## Concept 4 — Project Setup

### Initialize a project

```bash
dbt init ecomm_dbt
cd ecomm_dbt
```

dbt prompts you for the adapter (choose `snowflake`) and connection details. It creates this folder structure:

```
ecomm_dbt/
├── dbt_project.yml
├── models/
│   └── example/
├── tests/
├── macros/
├── seeds/
├── snapshots/
└── analyses/
```

**Key folders:**
- `models/` — your SQL transformation files live here
- `macros/` — reusable Jinja functions
- `seeds/` — CSV files you want to load as tables
- `tests/` — custom SQL tests (beyond built-in tests)

### profiles.yml

This file holds your Snowflake connection credentials. It lives at `~/.dbt/profiles.yml` — outside the project so it is never committed to Git.

```yaml
ecomm_dbt:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: <your-account-identifier>
      user: <your-username>
      password: <your-password>
      role: SYSADMIN
      database: ECOMM_DB
      warehouse: COMPUTE_WH
      schema: STAGING
      threads: 4
      client_session_keep_alive: false
```

**Key fields:**
- `account` — your Snowflake account identifier (e.g. `xy12345.us-east-1`)
- `database` — `ECOMM_DB` — the database that contains RAW_DATA, STAGING, MARTS schemas
- `schema` — `STAGING` — dbt writes staging models into this schema by default
- `role` — `SYSADMIN` — the role dbt uses to create objects
- `threads` — how many models dbt runs in parallel

### dbt_project.yml

```yaml
name: 'ecomm_dbt'
version: '1.0.0'
config-version: 2

profile: 'ecomm_dbt'

model-paths: ["models"]
seed-paths: ["seeds"]
test-paths: ["tests"]
macro-paths: ["macros"]

models:
  ecomm_dbt:
    staging:
      +materialized: view
    marts:
      +materialized: table
```

**Key fields:**
- `profile` — must exactly match the top-level key in `profiles.yml`
- `staging: +materialized: view` — all models inside `models/staging/` are created as views
- `marts: +materialized: table` — all models inside `models/marts/` are created as tables
- The `+` prefix means "apply this config to all files in this folder"

### Test the connection

```bash
dbt debug
```

Every check should show `OK`. If any line shows `ERROR`, fix `profiles.yml` and re-run.

---

## Concept 5 — Your First Model

### What is a model?

A **model** is a `.sql` file containing a single `SELECT` statement. The filename (without `.sql`) becomes the name of the view or table dbt creates in Snowflake.

### Create the staging model

Create `models/staging/stg_orders.sql`:

```sql
SELECT
    order_id,
    customer_id,
    customer_name,
    product,
    quantity,
    unit_price,
    total_amount,
    status,
    order_date,
    updated_at
FROM ECOMM_DB.RAW_DATA.ORDERS_RAW
WHERE order_id IS NOT NULL
```

### Run it

```bash
dbt run --select stg_orders
```

dbt generates and executes this in Snowflake:

```sql
CREATE OR REPLACE VIEW ECOMM_DB.STAGING.STG_ORDERS AS (
    SELECT
        order_id,
        customer_id,
        customer_name,
        product,
        quantity,
        unit_price,
        total_amount,
        status,
        order_date,
        updated_at
    FROM ECOMM_DB.RAW_DATA.ORDERS_RAW
    WHERE order_id IS NOT NULL
)
```

**What to verify in Snowsight:** `ECOMM_DB.STAGING.STG_ORDERS` now exists as a view.

The schema (`STAGING`) comes from your `profiles.yml`. You wrote only the SELECT — dbt added the `CREATE OR REPLACE VIEW` wrapper automatically.

### Run all models

```bash
dbt run
```

---

## Concept 6 — Sources and refs

### Why not hardcode table names?

The current model hardcodes `FROM ECOMM_DB.RAW_DATA.ORDERS_RAW`. If the schema name changes, you must update every model file. Also, dbt cannot build a lineage graph — it doesn't know `ORDERS_RAW` is a source (raw, externally loaded data).

`source()` and `ref()` solve both problems.

### source() — declare raw tables

Create `models/staging/sources.yml`:

```yaml
version: 2

sources:
  - name: raw_data
    database: ECOMM_DB
    schema: RAW_DATA
    tables:
      - name: orders_raw
        description: "Raw orders loaded from source systems"
```

**What each field does:**
- `name: raw_data` — the logical name you use in `{{ source('raw_data', ...) }}` — does not have to match the Snowflake schema name
- `database: ECOMM_DB` — the Snowflake database
- `schema: RAW_DATA` — the Snowflake schema
- `tables: - name: orders_raw` — the actual Snowflake table name

Now update `stg_orders.sql` to use `source()` instead of the hardcoded path:

```sql
SELECT
    order_id,
    customer_id,
    customer_name,
    product,
    quantity,
    unit_price,
    total_amount,
    status,
    order_date,
    updated_at
FROM {{ source('raw_data', 'orders_raw') }}
WHERE order_id IS NOT NULL
```

`{{ source('raw_data', 'orders_raw') }}` resolves to `ECOMM_DB.RAW_DATA.ORDERS_RAW` at run time. If the database or schema changes, you update `sources.yml` in one place and every model automatically picks it up.

### ref() — reference other models

Create `models/marts/orders_mart.sql`:

```sql
SELECT
    order_date,
    COUNT(*)                                                AS total_orders,
    SUM(CASE WHEN status = 'CONFIRMED'  THEN 1 ELSE 0 END) AS confirmed_orders,
    SUM(CASE WHEN status = 'SHIPPED'    THEN 1 ELSE 0 END) AS shipped_orders,
    SUM(CASE WHEN status = 'DELIVERED'  THEN 1 ELSE 0 END) AS delivered_orders,
    SUM(CASE WHEN status = 'CANCELLED'  THEN 1 ELSE 0 END) AS cancelled_orders,
    SUM(total_amount)                                       AS total_revenue,
    AVG(total_amount)                                       AS avg_order_value
FROM {{ ref('stg_orders') }}
GROUP BY order_date
ORDER BY order_date
```

`{{ ref('stg_orders') }}` does two things:
1. Resolves to `ECOMM_DB.STAGING.STG_ORDERS` at run time
2. Registers a dependency — dbt will always run `stg_orders` before `orders_mart`

### Run all models in dependency order

```bash
dbt run
```

Expected output:
```
1 of 2 START sql view model STAGING.stg_orders .................. OK
2 of 2 START sql table model MARTS.orders_mart .................. OK
Completed successfully.
Done. PASS=2 WARN=0 ERROR=0 SKIP=0 TOTAL=2
```

dbt built the dependency graph automatically from the `ref()` call and ran models in the right order.

### View the lineage graph

```bash
dbt docs generate
dbt docs serve
```

Opens browser. The lineage graph shows: `[source: orders_raw] → stg_orders → orders_mart`

---

## Concept 7 — Materializations

A **materialization** controls how dbt builds the model in Snowflake.

| Materialization | What dbt creates | When to use |
|---|---|---|
| `view` | `CREATE OR REPLACE VIEW` — no data stored | Staging models, lightweight transformations |
| `table` | `CREATE OR REPLACE TABLE AS SELECT` — full rebuild each run | Mart models, frequently queried aggregations |
| `incremental` | `INSERT` / `MERGE` — only new rows each run | Large tables where full rebuild is too slow |
| `ephemeral` | CTE only — no object created in Snowflake | Intermediate logic used by exactly one downstream model |

Set for a whole folder in `dbt_project.yml` (shown above). Override for a single model using the `config()` block at the top of the SQL file:

```sql
{{ config(materialized='table') }}

SELECT ...
```

**Why staging = view and marts = table:**
- Staging models are lightweight (just renaming/filtering) — a view is fine, no data needs to be stored
- Mart models are aggregations queried by BI tools hundreds of times a day — storing the result as a table means the aggregation runs once per `dbt run`, not once per BI query

---

## Hands-On Summary

```bash
pip install dbt-snowflake
dbt init ecomm_dbt
cd ecomm_dbt
dbt debug
dbt run
dbt docs generate && dbt docs serve
```

**Objects created in Snowflake after `dbt run`:**
- `ECOMM_DB.STAGING.STG_ORDERS` → view
- `ECOMM_DB.MARTS.ORDERS_MART` → table

---

## Summary

| Concept | Key takeaway |
|---|---|
| What is dbt | Transformation framework — write SQL SELECT, dbt handles CREATE, runs in dependency order |
| dbt Core vs Cloud | Core = CLI + Git; Cloud = managed SaaS with built-in scheduler |
| profiles.yml | Snowflake connection credentials — lives at `~/.dbt/`, never commit to Git |
| dbt_project.yml | Project config — model paths, default materializations per folder |
| Model | A `.sql` file with a SELECT — dbt wraps it in CREATE VIEW/TABLE AS |
| source() | Declares a raw table — one place to update if schema or database changes |
| ref() | References another model — builds the dependency graph automatically |
| Materialization | view / table / incremental / ephemeral — controls what dbt creates in Snowflake |
