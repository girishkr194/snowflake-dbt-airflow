# Day 2 — dbt Testing, Documentation, and Advanced Models

## What You Will Learn

1. **Sources with freshness** — Properly declare raw tables, check if source data is stale
2. **Schema YAML** — Describe every column, attach tests in a `.yml` file
3. **Built-in tests** — `not_null`, `unique`, `accepted_values`, `relationships`
4. **`dbt test`** — Run all tests, understand pass/fail/warn output
5. **Incremental models** — Only process new rows instead of rebuilding the full table
6. **Model documentation** — `description` fields, `dbt docs generate`, lineage graph
7. **Jinja basics** — `{{ config() }}`, `{{ ref() }}`, `{{ source() }}` explained line by line

> **Starting point:** The `ecomm_dbt` project from Day 1 — `stg_orders.sql` exists and runs.
> Today we improve it: add sources, descriptions, tests, and an incremental mart.

---

## Concept 1 — Sources (the right way)

### What we have today

Your current `stg_orders.sql` hardcodes the raw table path:

```sql
FROM ECOMM_DB.RAW_DATA.ORDERS_RAW   -- ❌ hardcoded — brittle, no lineage tracking
```

This works but has two problems:
- If the schema or database ever changes, you must find and update every model file individually
- dbt has no idea this is a "source" (raw, externally loaded data) — it doesn't appear in the lineage graph as a source node

### The fix: declare a source in YAML

Create `models/staging/sources.yml`:

```yaml
version: 2                       # always required at the top of every YAML file in dbt

sources:                         # the top-level key that tells dbt these are source declarations
  - name: raw_data               # a logical name you invent — used in {{ source('raw_data', ...) }}
                                 # does NOT have to match the Snowflake schema name exactly
    database: ECOMM_DB           # the Snowflake database where the raw table lives
    schema: RAW_DATA             # the Snowflake schema where the raw table lives
    description: >               # human-readable description shown in dbt docs
      Raw tables loaded from source systems via Snowpipe or COPY INTO.
      These tables are owned by the data engineering team and must not be modified.

    loaded_at_field: updated_at  # the column dbt uses to check source freshness
                                 # dbt compares MAX(updated_at) against the current time
                                 # if data is older than the warn/error threshold, dbt alerts you

    freshness:                   # how stale is too stale?
      warn_after:                # issue a WARNING (run still passes) if data is older than:
        count: 24
        period: hour             # 24 hours — warn if no new data in 1 day
      error_after:               # issue an ERROR (run fails) if data is older than:
        count: 48
        period: hour             # 48 hours — hard fail if no new data in 2 days

    tables:
      - name: orders_raw         # the actual Snowflake table name (case-insensitive)
        description: >
          One row per customer order. Loaded from the e-commerce platform every hour.
          Contains all order statuses including CANCELLED orders — downstream models
          must filter or handle these explicitly.
        columns:
          - name: order_id
            description: "Primary key — unique identifier for each order"
          - name: customer_id
            description: "Foreign key to the customers table"
          - name: customer_name
            description: "Full name of the customer at time of order"
          - name: product
            description: "Product name ordered"
          - name: quantity
            description: "Number of units ordered"
          - name: unit_price
            description: "Price per unit in USD at time of order"
          - name: total_amount
            description: "quantity * unit_price — computed at order time, not recalculated"
          - name: status
            description: "Order lifecycle status: CONFIRMED, SHIPPED, DELIVERED, CANCELLED"
          - name: order_date
            description: "Date the order was placed"
          - name: updated_at
            description: "Timestamp of the last update to this row — used for source freshness"
```

### Update stg_orders.sql to use source()

```sql
-- models/staging/stg_orders.sql

-- {{ source('raw_data', 'orders_raw') }} does two things:
--   1. Resolves to ECOMM_DB.RAW_DATA.ORDERS_RAW at run time
--   2. Registers this model as a consumer of the 'orders_raw' source in the lineage graph
--      so the docs page shows:  [source: orders_raw] → stg_orders → orders_mart

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
FROM {{ source('raw_data', 'orders_raw') }}   -- replaces hardcoded ECOMM_DB.RAW_DATA.ORDERS_RAW
WHERE order_id IS NOT NULL                    -- basic quality gate: skip rows with no order_id
```

### Check source freshness

```bash
dbt source freshness
# Checks MAX(updated_at) in ORDERS_RAW against the thresholds in sources.yml
# Output:
#   Found 1 source, 1 table freshness check
#   1 of 1 START freshness of raw_data.orders_raw ................... [RUN]
#   1 of 1 PASS freshness of raw_data.orders_raw .................... [PASS in 0.85s]
```

---

## Concept 2 — Schema YAML: Describing and Testing Models

### What is schema.yml?

A `schema.yml` file (you can name it anything ending in `.yml` — the name is a convention) is where you:
- Write descriptions for models and their columns (shown in `dbt docs`)
- Attach **tests** to columns that dbt runs with `dbt test`

Create `models/staging/schema.yml`:

```yaml
version: 2                       # required at the top of every dbt YAML

models:                          # list of models to describe and test
  - name: stg_orders             # must match the filename: stg_orders.sql
    description: >               # description shown in dbt docs — use > for multi-line strings
      Cleaned and validated orders from the raw ORDERS_RAW table.
      This is the single source of truth for order data consumed by all mart models.
      Hardcoded schema references have been replaced with source() for portability.

    columns:                     # list every column the model outputs

      - name: order_id
        description: "Primary key — unique integer identifier for each order"
        tests:                   # tests run against this column when you run `dbt test`
          - not_null             # FAIL if any row has order_id = NULL
          - unique               # FAIL if any two rows share the same order_id

      - name: customer_id
        description: "Integer foreign key — identifies which customer placed the order"
        tests:
          - not_null             # every order must belong to a customer

      - name: customer_name
        description: "Full name of the customer at the time the order was placed"
        tests:
          - not_null

      - name: product
        description: "Name of the product ordered"
        tests:
          - not_null

      - name: quantity
        description: "Number of units ordered — must be a positive integer"
        tests:
          - not_null

      - name: unit_price
        description: "Price per unit in USD at the time of order"
        tests:
          - not_null

      - name: total_amount
        description: "Total order value in USD — quantity * unit_price"
        tests:
          - not_null

      - name: status
        description: "Current status of the order in the fulfillment lifecycle"
        tests:
          - not_null
          - accepted_values:     # FAIL if any row has a status value not in this list
              values:            # list of all valid status strings — dbt checks every row
                - 'CONFIRMED'
                - 'SHIPPED'
                - 'DELIVERED'
                - 'CANCELLED'

      - name: order_date
        description: "Calendar date the order was placed"
        tests:
          - not_null

      - name: updated_at
        description: "Timestamp of the last modification to this order record"
        tests:
          - not_null
```

---

## Concept 3 — The Four Built-in dbt Tests

dbt ships with four generic tests you can attach to any column in any YAML file — no SQL required.

### 1. `not_null`

```yaml
- name: order_id
  tests:
    - not_null
```

**What dbt runs behind the scenes:**
```sql
SELECT COUNT(*)
FROM ECOMM_DB.DBT_DEV.STG_ORDERS
WHERE order_id IS NULL
-- PASS if result = 0
-- FAIL if result > 0 (at least one NULL found)
```

Use on: primary keys, foreign keys, any column that must always have a value.

---

### 2. `unique`

```yaml
- name: order_id
  tests:
    - unique
```

**What dbt runs behind the scenes:**
```sql
SELECT COUNT(*) FROM (
    SELECT order_id
    FROM ECOMM_DB.DBT_DEV.STG_ORDERS
    WHERE order_id IS NOT NULL
    GROUP BY order_id
    HAVING COUNT(*) > 1
)
-- PASS if result = 0  (no duplicates)
-- FAIL if result > 0  (at least one order_id appears more than once)
```

Use on: primary keys — any column that must be unique across all rows.

---

### 3. `accepted_values`

```yaml
- name: status
  tests:
    - accepted_values:
        values: ['CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED']
```

**What dbt runs behind the scenes:**
```sql
SELECT COUNT(*) FROM (
    SELECT DISTINCT status
    FROM ECOMM_DB.DBT_DEV.STG_ORDERS
    WHERE status NOT IN ('CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED')
      AND status IS NOT NULL
)
-- PASS if result = 0  (no unexpected values)
-- FAIL if result > 0  (a new status appeared that is not in the allowed list)
```

Use on: status columns, category columns, any enum-like field with a known set of valid values.

---

### 4. `relationships`

```yaml
- name: customer_id
  tests:
    - relationships:
        to: ref('stg_customers')   # the model that owns the primary key
        field: customer_id         # the column in that model to check against
```

**What dbt runs behind the scenes:**
```sql
SELECT COUNT(*) FROM (
    SELECT customer_id
    FROM ECOMM_DB.DBT_DEV.STG_ORDERS
    WHERE customer_id NOT IN (
        SELECT customer_id FROM ECOMM_DB.DBT_DEV.STG_CUSTOMERS
    )
    AND customer_id IS NOT NULL
)
-- PASS if result = 0  (every customer_id in orders exists in customers)
-- FAIL if result > 0  (orphan orders — orders with no matching customer)
```

Use on: foreign key columns to verify referential integrity between models.

---

## Concept 4 — Running Tests

```bash
# Run ALL tests in the project
dbt test

# Run tests only for a specific model
dbt test --select stg_orders

# Run tests only for a specific test type
dbt test --select test_type:not_null

# Run models first, then immediately test them
dbt build           # equivalent to: dbt run + dbt test in dependency order
```

### Understanding test output

```
Running with dbt=1.x.x
Found 2 models, 9 tests

Concurrency: 4 threads

1 of 9 START test not_null_stg_orders_order_id .............. [RUN]
1 of 9 PASS  test not_null_stg_orders_order_id .............. [PASS in 0.45s]
2 of 9 START test unique_stg_orders_order_id ................ [RUN]
2 of 9 PASS  test unique_stg_orders_order_id ................ [PASS in 0.38s]
3 of 9 START test accepted_values_stg_orders_status ......... [RUN]
3 of 9 PASS  test accepted_values_stg_orders_status ......... [PASS in 0.41s]
...
Done. PASS=9 WARN=0 ERROR=0 SKIP=0 TOTAL=9
```

Each test name is auto-generated: `{test_type}_{model_name}_{column_name}`.

### What to do when a test fails

```
4 of 9 FAIL test not_null_stg_orders_customer_name .......... [FAIL 2]
```

The number after `FAIL` is the **row count** that violated the test — here, 2 rows have a NULL `customer_name`. Find the failing rows:

```sql
-- dbt saves the failing rows query in target/compiled/
-- Or run this manually in Snowsight:
SELECT *
FROM ECOMM_DB.DBT_DEV.STG_ORDERS
WHERE customer_name IS NULL;
```

---

## Concept 5 — Incremental Models

### Why incremental?

For the mart model, `materialized='table'` does a **full rebuild** every run:
```sql
-- What dbt does on every run with materialized='table':
CREATE OR REPLACE TABLE ECOMM_DB.DBT_DEV.ORDERS_MART AS (
    SELECT ... FROM stg_orders GROUP BY order_date
)
-- If stg_orders has 10 million rows, all 10 million are scanned every run
```

For small datasets this is fine. For large tables (millions of rows, running every 5 minutes) it wastes compute. **Incremental models** solve this by only processing rows that are new since the last run.

### How incremental works

```
First run:  dbt builds the full table (like materialized='table')
Every subsequent run:
  1. dbt runs your SELECT with the WHERE filter to get only new/changed rows
  2. dbt INSERTs those rows into the existing table (or MERGEs if unique_key is set)
  3. Old rows already in the table are untouched
```

### Create an incremental staging model

Create `models/staging/stg_orders_incremental.sql`:

```sql
-- models/staging/stg_orders_incremental.sql
--
-- This model is INCREMENTAL: on the first run it builds the full table.
-- On every subsequent run it only processes rows where updated_at is newer
-- than the latest updated_at already in the target table.

{{                                     -- opening double curly brace: start of a Jinja block
    config(                            -- config() sets model-level options
        materialized = 'incremental', -- tells dbt: build as a table, then INSERT on subsequent runs
                                       -- (instead of CREATE OR REPLACE TABLE every time)
        unique_key   = 'order_id'      -- if a row with this order_id already exists in the table,
                                       -- UPDATE it instead of inserting a duplicate.
                                       -- Without unique_key, dbt only INSERTs — duplicates possible
    )
}}                                     -- closing Jinja block

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
FROM {{ source('raw_data', 'orders_raw') }}   -- same source as stg_orders

{% if is_incremental() %}
-- is_incremental() returns TRUE on every run AFTER the first full build.
-- On the first run it returns FALSE — the whole WHERE block is skipped
-- and dbt processes ALL rows to build the initial table.
--
-- On subsequent runs:
--   this_model (alias for the current incremental table) is queried to find
--   the most recent updated_at already loaded. Only rows newer than that are processed.

WHERE updated_at > (
    SELECT MAX(updated_at)        -- find the latest timestamp already in the table
    FROM {{ this }}               -- {{ this }} refers to the incremental table itself
                                  -- resolves to ECOMM_DB.DBT_DEV.STG_ORDERS_INCREMENTAL
)
{% endif %}
-- The {% if %}...{% endif %} block is Jinja templating — dbt evaluates it at compile time
-- and either includes or excludes the WHERE clause depending on is_incremental()
```

### Run the incremental model

```bash
# First run: builds full table (is_incremental() = FALSE, no WHERE filter)
dbt run --select stg_orders_incremental

# Subsequent runs: only new rows (is_incremental() = TRUE, WHERE filter applied)
dbt run --select stg_orders_incremental

# Force a full rebuild (ignores is_incremental, re-creates the whole table)
dbt run --select stg_orders_incremental --full-refresh
```

### Incremental vs Table vs View — decision guide

```
Your model reads 100 rows from ORDERS_RAW?
  → materialized='view'   — no data stored, just a query definition

Your model reads 100k rows and runs every hour?
  → materialized='table'  — rebuild is fast enough, simplest to reason about

Your model reads 10 million rows and runs every 5 minutes?
  → materialized='incremental'  — only new rows each run, 1000x less compute
```

---

## Concept 6 — Jinja Templating Basics

dbt uses **Jinja** — a Python templating language — inside SQL files. Everything between `{{ }}` or `{% %}` is Jinja that dbt evaluates before sending SQL to Snowflake.

### `{{ }}` — expressions (output a value)

```sql
FROM {{ source('raw_data', 'orders_raw') }}
-- dbt evaluates source() and outputs: ECOMM_DB.RAW_DATA.ORDERS_RAW

FROM {{ ref('stg_orders') }}
-- dbt evaluates ref() and outputs: ECOMM_DB.DBT_DEV.STG_ORDERS

{{ config(materialized='table') }}
-- dbt reads the config and sets materialization — outputs nothing in the SQL
```

### `{% %}` — statements (logic, no output)

```sql
{% if is_incremental() %}
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
-- dbt evaluates the if condition. The WHERE clause is included or excluded.
-- Nothing between {% %} is sent to Snowflake as-is.
```

### `{{ this }}` — reference to the current model's table

```sql
SELECT MAX(updated_at) FROM {{ this }}
-- In an incremental model, {{ this }} = the existing table being incrementally updated
-- Resolves to: ECOMM_DB.DBT_DEV.STG_ORDERS_INCREMENTAL
```

### Why Jinja matters for a data engineer

Jinja lets you write SQL that is:
- **Portable** — `{{ source() }}` and `{{ ref() }}` resolve differently in dev vs prod
- **Conditional** — `{% if is_incremental() %}` changes behavior based on context
- **Reusable** — macros (covered in Day 3) let you define logic once and call it everywhere

---

## Concept 7 — Model Documentation

### Add descriptions to the mart model

Create `models/marts/schema.yml`:

```yaml
version: 2

models:
  - name: orders_mart                    # must match orders_mart.sql filename exactly
    description: >
      Daily order summary aggregated from stg_orders.
      One row per order_date. Used by the BI team for revenue dashboards.
      Rebuilt as a full table on every dbt run.

    columns:
      - name: order_date
        description: "The calendar date of the orders summarized in this row"
        tests:
          - not_null
          - unique               # each date should appear exactly once in the mart

      - name: total_orders
        description: "Total number of orders placed on this date, all statuses"
        tests:
          - not_null

      - name: confirmed_orders
        description: "Orders with status = CONFIRMED on this date"

      - name: shipped_orders
        description: "Orders with status = SHIPPED on this date"

      - name: delivered_orders
        description: "Orders with status = DELIVERED on this date"

      - name: cancelled_orders
        description: "Orders with status = CANCELLED on this date"

      - name: total_revenue
        description: "Sum of total_amount for all orders on this date"
        tests:
          - not_null

      - name: avg_order_value
        description: "Average total_amount per order on this date"
```

### Generate and view the docs

```bash
# Step 1: generate the catalog — reads all schema.yml files and warehouse metadata
dbt docs generate

# Step 2: serve the docs locally
dbt docs serve
# Opens http://localhost:8080

# What you will see:
# - Project tab: list of all models and sources
# - Each model page: description, column list with descriptions and tests
# - Lineage graph (bottom right): visual DAG
#   [source: orders_raw] → stg_orders → orders_mart
#                        → stg_orders_incremental
```

---

## Final File Structure After Day 2

```
ecomm_dbt/
├── dbt_project.yml
├── models/
│   ├── staging/
│   │   ├── sources.yml              ← NEW: declares orders_raw as a source with freshness
│   │   ├── schema.yml               ← NEW: stg_orders column descriptions + tests
│   │   ├── stg_orders.sql           ← UPDATED: now uses {{ source() }} instead of hardcoded path
│   │   └── stg_orders_incremental.sql  ← NEW: incremental model with Jinja {% if %}
│   └── marts/
│       ├── schema.yml               ← NEW: orders_mart column descriptions + tests
│       ├── orders_mart.sql          ← unchanged from Day 1
│       └── customer_summary.sql     ← unchanged from Day 1
```

---

## Summary

| Concept | Key takeaway |
|---|---|
| `sources.yml` | Declare raw tables once — `source()` resolves them everywhere, change in one place |
| Source freshness | `dbt source freshness` checks if raw data is stale using `loaded_at_field` |
| `schema.yml` | Descriptions + tests live alongside your models — not in a separate system |
| `not_null` | Fails if any row has NULL in this column |
| `unique` | Fails if any value appears more than once |
| `accepted_values` | Fails if a value outside the allowed list appears |
| `relationships` | Fails if a foreign key value has no matching row in the referenced model |
| `dbt test` | Runs all tests; `dbt build` runs models + tests together |
| `incremental` | First run = full build; subsequent runs = only new rows via WHERE filter |
| `is_incremental()` | Jinja function — TRUE on runs after the first build, FALSE on first run |
| `{{ this }}` | Inside incremental models — refers to the existing target table |
| Jinja `{{ }}` | Expression block — outputs a value into SQL |
| Jinja `{% %}` | Statement block — logic (if/for) that controls SQL structure, no output |
