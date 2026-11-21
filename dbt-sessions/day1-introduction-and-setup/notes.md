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
  ┌─────────────────┐
  │  RAW schema     │  ← source tables (loaded by EL tools)
  │  STAGING schema │  ← dbt models (clean, rename, cast)
  │  MARTS schema   │  ← dbt models (business logic, aggregation)
  └─────────────────┘
        │
        ▼
   [BI Tool]  ← Tableau, Power BI, Looker
```

dbt only handles the **T** in ELT. It does not move data — it transforms data already in the warehouse.

### What dbt Actually Does

When you run `dbt run`, dbt:
1. Reads your `.sql` model files
2. Wraps each one in a `CREATE TABLE AS` or `CREATE VIEW AS` statement
3. Runs them in dependency order (models that depend on other models run after)
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

**This course uses dbt Core** — you install it locally, connect to Snowflake, and run everything from the terminal. This is the most common setup in production data engineering teams.

---

## Concept 3 — Installation

### Prerequisites

- Python 3.8+
- pip
- Snowflake account (from the Snowflake module)

### Install dbt Core + Snowflake adapter

```bash
pip install dbt-snowflake
```

This installs:
- `dbt-core` — the dbt engine
- `dbt-snowflake` — the Snowflake adapter (translates dbt commands to Snowflake SQL)

Verify:
```bash
dbt --version
# dbt Core: 1.x.x
# Installed adapters: snowflake
```

---

## Concept 4 — Project Setup

### Initialize a project

```bash
dbt init ecomm_dbt
cd ecomm_dbt
```

dbt prompts you for the adapter (choose `snowflake`) and connection details. It creates:

```
ecomm_dbt/
├── dbt_project.yml        ← project config (name, model paths, materializations)
├── profiles.yml           ← connection credentials (usually lives in ~/.dbt/)
├── models/                ← your SQL transformation files
│   └── example/           ← sample models (delete these)
├── tests/                 ← custom data tests
├── macros/                ← reusable Jinja macros
├── seeds/                 ← CSV files you load as tables
├── snapshots/             ← SCD Type 2 snapshots
└── analyses/              ← ad-hoc SQL (not materialized)
```

### profiles.yml

This file holds your Snowflake connection. It lives at `~/.dbt/profiles.yml` (outside the project directory so credentials are not committed to Git).

```yaml
ecomm_dbt:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: <your-account-identifier>   # e.g. xy12345.us-east-1
      user: <your-username>
      password: <your-password>
      role: SYSADMIN
      database: ECOMM_DB
      warehouse: COMPUTE_WH
      schema: DBT_DEV                      # dbt writes models here in dev
      threads: 4
      client_session_keep_alive: false
```

> **Important:** `schema` in `profiles.yml` is the **default target schema** for dev. In production you would point this at `STAGING` or `MARTS`. In dev, dbt writes to a personal schema so you don't overwrite production tables.

### dbt_project.yml

```yaml
name: 'ecomm_dbt'
version: '1.0.0'
config-version: 2

profile: 'ecomm_dbt'   # must match the key in profiles.yml

model-paths: ["models"]
seed-paths: ["seeds"]
test-paths: ["tests"]
macro-paths: ["macros"]

models:
  ecomm_dbt:
    staging:
      +materialized: view      # staging models → views
    marts:
      +materialized: table     # mart models → tables
```

### Test the connection

```bash
dbt debug
# All checks passed!
```

---

## Concept 5 — Your First Model

### What is a model?

A **model** is a `.sql` file containing a single `SELECT` statement. dbt wraps it in a `CREATE OR REPLACE VIEW/TABLE AS` and runs it in Snowflake.

### Create the staging model

Create `models/staging/stg_orders.sql`:

```sql
-- models/staging/stg_orders.sql

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

dbt executes:
```sql
CREATE OR REPLACE VIEW ECOMM_DB.DBT_DEV.stg_orders AS (
    SELECT
        order_id,
        customer_id,
        ...
    FROM ECOMM_DB.RAW_DATA.ORDERS_RAW
    WHERE order_id IS NOT NULL
)
```

Check in Snowsight: `ECOMM_DB.DBT_DEV.STG_ORDERS` now exists as a view.

### Run all models

```bash
dbt run
```

### What happens per model run

```
1. dbt reads models/staging/stg_orders.sql
2. Resolves ref() and source() calls (see Concept 6)
3. Checks materialization config (view or table)
4. Generates final SQL: CREATE OR REPLACE VIEW ... AS (your SELECT)
5. Executes it in Snowflake
6. Reports: OK  created view ECOMM_DB.DBT_DEV.STG_ORDERS
```

---

## Concept 6 — Sources and refs

### Why not just hardcode table names?

If you write `FROM ECOMM_DB.RAW_DATA.ORDERS_RAW` in every model, and the schema changes, you update every file. Worse — dbt can't build a dependency graph between your models and raw tables.

**Sources** and **refs** solve this.

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

Now reference it in your model:

```sql
-- models/staging/stg_orders.sql

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

`{{ source('raw_data', 'orders_raw') }}` resolves to `ECOMM_DB.RAW_DATA.ORDERS_RAW` at run time. If the database or schema ever changes, you update `sources.yml` in one place.

### ref() — reference other models

Create `models/marts/orders_mart.sql`:

```sql
-- models/marts/orders_mart.sql

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
1. Resolves to the correct schema-qualified name: `ECOMM_DB.DBT_DEV.STG_ORDERS`
2. Tells dbt that `orders_mart` **depends on** `stg_orders` — so dbt always runs `stg_orders` first

### Run all models in dependency order

```bash
dbt run
# 1 of 2 START view model DBT_DEV.stg_orders .................. OK
# 2 of 2 START table model DBT_DEV.orders_mart ................ OK
```

dbt built the dependency graph automatically from `ref()` calls.

### View the lineage graph

```bash
dbt docs generate
dbt docs serve
# Opens browser: shows stg_orders → orders_mart lineage
```

---

## Concept 7 — Materializations

A **materialization** controls how dbt builds the model in the warehouse.

| Materialization | What dbt creates | When to use |
|---|---|---|
| `view` (default) | `CREATE VIEW` — no data stored | Staging models, lightweight transformations |
| `table` | `CREATE TABLE AS SELECT` — full rebuild each run | Mart models, frequently queried aggregations |
| `incremental` | `INSERT` / `MERGE` — only new rows | Large tables where full rebuild is too slow |
| `ephemeral` | CTE only — no object created in warehouse | Intermediate logic used by one model only |

Set in `dbt_project.yml` (applies to all models in a folder) or inline per model:

```sql
-- models/marts/orders_mart.sql

{{ config(materialized='table') }}

SELECT ...
```

---

## Hands-On Summary

```bash
# 1. Install
pip install dbt-snowflake

# 2. Initialize project
dbt init ecomm_dbt
cd ecomm_dbt

# 3. Configure connection
# Edit ~/.dbt/profiles.yml with your Snowflake credentials

# 4. Test connection
dbt debug

# 5. Create models
# models/staging/sources.yml     ← declare ORDERS_RAW as a source
# models/staging/stg_orders.sql  ← clean/rename raw data
# models/marts/orders_mart.sql   ← aggregate to mart

# 6. Run
dbt run

# 7. Verify in Snowsight
# ECOMM_DB.DBT_DEV.STG_ORDERS   → view
# ECOMM_DB.DBT_DEV.ORDERS_MART  → table

# 8. View lineage
dbt docs generate && dbt docs serve
```

---

## Summary

| Concept | Key takeaway |
|---|---|
| What is dbt | Transformation framework — write SQL, dbt handles CREATE, runs in dependency order |
| dbt Core vs Cloud | Core = CLI + Git; Cloud = managed SaaS with scheduler and UI |
| profiles.yml | Connection credentials — lives outside the project, never commit to Git |
| dbt_project.yml | Project config — model paths, materializations, profile name |
| Model | A `.sql` file with a SELECT — dbt wraps it in CREATE VIEW/TABLE AS |
| source() | Declares a raw table — single place to update if schema changes |
| ref() | References another model — builds the dependency graph automatically |
| Materialization | view, table, incremental, ephemeral — controls how the model is built |
