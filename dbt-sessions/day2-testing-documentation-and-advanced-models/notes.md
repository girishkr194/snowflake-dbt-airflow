# Day 2 — dbt Testing, Documentation, and Advanced Models

## What You Will Learn

1. **Dynamic schema routing** — Write staging models to `STAGING`, mart models to `MARTS` — automatically, without changing `profiles.yml`
2. **`sources.yml` vs `schema.yml`** — What each file is for, what the top-level keys mean, and why the filename doesn't matter
3. **Sources with freshness** — Properly declare raw tables, check if source data is stale
4. **Schema YAML** — Describe every column, attach tests in a `.yml` file
5. **Built-in tests** — `not_null`, `unique`, `accepted_values`, `relationships`
6. **`dbt test`** — Run all tests, understand pass/fail output
7. **Incremental models** — Only process new rows instead of rebuilding the full table
8. **Model documentation** — `description` fields, `dbt docs generate`, lineage graph
9. **Jinja basics** — `{{ config() }}`, `{{ ref() }}`, `{{ source() }}`, `{% if %}` explained

> **Starting point:** The `ecomm_dbt` project from Day 1 with `stg_orders.sql` running.
> Today we improve it: add dynamic schema routing, sources, descriptions, tests, and an incremental model.

---

## Concept 0 — Dynamic Schema Routing

### The Problem

`profiles.yml` has a single `schema:` field. Whatever you put there is where **all** models land — regardless of which folder they are in.

```
profiles.yml → schema: STAGING

Result:
  models/staging/stg_orders.sql   → ECOMM_DB.STAGING.STG_ORDERS    ✔
  models/marts/orders_mart.sql    → ECOMM_DB.STAGING.ORDERS_MART    ✘  (should go to MARTS)
```

If you change `schema: MARTS` in `profiles.yml`, the mart goes to the right place but staging breaks. You cannot fix this by editing `profiles.yml` — it only holds one schema value.

### The Solution: `+schema` in dbt_project.yml + a custom macro

dbt has a built-in hook called `generate_schema_name`. By default it takes the schema from `profiles.yml` and **appends** your folder-level schema to it, producing names like `STAGING_MARTS` — not what you want.

The fix is two steps:

**Step 1 — tell each folder which schema to use in `dbt_project.yml`:**

```yaml
models:
  ecomm_dbt:
    staging:
      +materialized: view
      +schema: STAGING
    marts:
      +materialized: table
      +schema: MARTS
```

**Step 2 — override `generate_schema_name` to use the folder schema as-is:**

Create `macros/generate_schema_name.sql`:

```sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- if custom_schema_name is none -%}
        {{ target.schema }}
    {%- else -%}
        {{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

**How it works:**

Without the macro, dbt's default logic produces: `{target.schema}_{custom_schema_name}` → `STAGING_MARTS`.

With the macro override:
- If `+schema` is set in `dbt_project.yml` → use it exactly as written (`STAGING`, `MARTS`)
- If `+schema` is not set → fall back to whatever is in `profiles.yml`

**Result after both changes:**

```
profiles.yml → schema: STAGING  (this is now just a fallback, rarely matters)

models/staging/stg_orders.sql   → ECOMM_DB.STAGING.STG_ORDERS   ✔
models/marts/orders_mart.sql    → ECOMM_DB.MARTS.ORDERS_MART     ✔
```

You never need to touch `profiles.yml` again when adding new model folders. Just add `+schema: <your_schema>` in `dbt_project.yml`.

### What each part does

`+schema: STAGING` in `dbt_project.yml` — the `+` prefix means "apply to all models in this folder". Sets the `custom_schema_name` variable that dbt passes to `generate_schema_name`.

`custom_schema_name is none` — if a model folder has no `+schema` set, `custom_schema_name` is None. We fall back to `target.schema` (the value from `profiles.yml`).

`{{ custom_schema_name | trim }}` — `trim` removes any accidental leading/trailing whitespace from the schema name before using it.

`target.schema` — the schema value from `profiles.yml` for the current target (`dev`, `prod`, etc.).

### Full working dbt_project.yml after this change

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
      +schema: STAGING
    marts:
      +materialized: table
      +schema: MARTS
```

### Verify it works

```bash
dbt run
```

```sql
SHOW VIEWS  IN SCHEMA ECOMM_DB.STAGING;
SHOW TABLES IN SCHEMA ECOMM_DB.MARTS;
```

`STG_ORDERS` should appear under `STAGING`, `ORDERS_MART` under `MARTS`.

---

## Concept 1 — `sources.yml` vs `schema.yml` — What's the Difference?

Both are YAML files that live inside `models/`. They look similar but serve completely different purposes.

### File naming — can you use any name?

| File | Fixed name? |
|---|---|
| `sources.yml` | **No** — you can name it anything (`raw_sources.yml`, `my_sources.yml`). dbt identifies it by the `sources:` key inside it, not the filename. |
| `schema.yml` | **No** — same rule. You can name it `stg_orders.yml`, `models.yml`, anything. dbt identifies it by the `models:` key inside it. |

The names `sources.yml` and `schema.yml` are just **conventions** the community follows. dbt does not care about the filename — it scans all `.yml` files in `model-paths` and reads whatever top-level keys (`sources:`, `models:`) it finds.

### What each file is for

| | `sources.yml` | `schema.yml` |
|---|---|---|
| **What it covers** | Tables dbt did **not** create — raw tables loaded by Fivetran, Airbyte, Snowpipe, etc. | Models dbt **did** create — your `.sql` files |
| **Top-level key** | `sources:` | `models:` |
| **Unlocks** | `{{ source('raw_data', 'orders_raw') }}` macro in SQL | `dbt test` and `dbt docs` for your models |
| **Freshness checks** | Yes — `dbt source freshness` checks MAX(updated_at) | No |
| **Tests** | Column-level tests on source tables | Column-level tests on dbt models |
| **Lineage graph** | Shows as a **green source node** | Shows as a **blue model node** |

### Side-by-side example

```yaml
# sources.yml  — declares ORDERS_RAW which dbt did NOT build
version: 2
sources:
  - name: raw_data              # logical name used in {{ source() }}
    database: ECOMM_DB
    schema: RAW_DATA
    tables:
      - name: orders_raw        # actual Snowflake table name
```

```yaml
# schema.yml  — documents STG_ORDERS which dbt DID build
version: 2
models:
  - name: stg_orders            # must match stg_orders.sql filename exactly
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
```

### One-line rule

> `sources.yml` = "here is data I did not create, tell dbt where to find it"
> `schema.yml` = "here is data dbt created, add tests and descriptions to it"

A common pattern is to have **both files in the same folder** (`models/staging/`) — `sources.yml` declares the raw input, `schema.yml` tests and documents the staging output that was built from it.

---

## Concept 2 — Sources (the right way)

### What we have today

Your current `stg_orders.sql` hardcodes the raw table path:

```sql
FROM ECOMM_DB.RAW_DATA.ORDERS_RAW
```

This works but has two problems:
- If the schema or database ever changes, you must update every model file individually
- dbt has no idea this is a "source" — it doesn't appear in the lineage graph as a source node

### The fix: declare a source in YAML

Create `models/staging/sources.yml`:

```yaml
version: 2

sources:
  - name: raw_data
    database: ECOMM_DB
    schema: RAW_DATA
    description: "Raw tables loaded from source systems via Snowpipe or COPY INTO."
    loaded_at_field: updated_at
    freshness:
      warn_after:
        count: 24
        period: hour
      error_after:
        count: 48
        period: hour
    tables:
      - name: orders_raw
        description: "One row per customer order. Contains all statuses including CANCELLED."
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
            description: "quantity * unit_price — computed at order time"
          - name: status
            description: "Order lifecycle status: CONFIRMED, SHIPPED, DELIVERED, CANCELLED"
          - name: order_date
            description: "Date the order was placed"
          - name: updated_at
            description: "Timestamp of the last update — used for source freshness"
```

**What each field does:**
- `name: raw_data` — logical name used in `{{ source('raw_data', ...) }}`. Does not have to match the Snowflake schema name exactly.
- `database: ECOMM_DB` — Snowflake database where the raw table lives
- `schema: RAW_DATA` — Snowflake schema where the raw table lives
- `loaded_at_field: updated_at` — the column dbt uses to check freshness. dbt runs `SELECT MAX(updated_at) FROM orders_raw` and compares to current time
- `freshness: warn_after` — issue a WARNING (run still passes) if no new data for 24 hours
- `freshness: error_after` — issue an ERROR (run fails) if no new data for 48 hours
- `tables: - name: orders_raw` — the actual Snowflake table name (case-insensitive)

### Update stg_orders.sql to use source()

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

`{{ source('raw_data', 'orders_raw') }}` resolves to `ECOMM_DB.RAW_DATA.ORDERS_RAW` at run time.
- First argument `'raw_data'` matches the `name:` key under `sources:` in your YAML
- Second argument `'orders_raw'` matches the `name:` key under `tables:` in your YAML

If the schema ever changes, you update `sources.yml` in one place and all models pick it up automatically.

### Check source freshness

```bash
dbt source freshness
```

Expected output:
```
1 of 1 START freshness of raw_data.orders_raw .................. [RUN]
1 of 1 PASS  freshness of raw_data.orders_raw .................. [PASS in 0.85s]
```

If you see WARN or ERROR, it means `ORDERS_RAW.updated_at` has not been updated recently enough.

---

## Concept 3 — Schema YAML: Describing and Testing Models

### What is schema.yml?

A `schema.yml` file is where you:
- Write descriptions for models and their columns (shown in `dbt docs`)
- Attach **tests** to columns that dbt runs with `dbt test`

Create `models/staging/schema.yml`:

```yaml
version: 2

models:
  - name: stg_orders
    description: >
      Cleaned and validated orders from ORDERS_RAW.
      Single source of truth for order data consumed by all mart models.

    columns:
      - name: order_id
        description: "Primary key — unique integer identifier for each order"
        tests:
          - not_null
          - unique

      - name: customer_id
        description: "Integer foreign key — identifies which customer placed the order"
        tests:
          - not_null

      - name: customer_name
        description: "Full name of the customer at the time the order was placed"
        tests:
          - not_null

      - name: product
        description: "Name of the product ordered"
        tests:
          - not_null

      - name: quantity
        description: "Number of units ordered — always a positive integer"
        tests:
          - not_null

      - name: unit_price
        description: "Price per unit in USD at the time of order"
        tests:
          - not_null

      - name: total_amount
        description: "Total order value in USD — quantity * unit_price, pre-tax"
        tests:
          - not_null

      - name: status
        description: "Current fulfillment status of the order"
        tests:
          - not_null
          - accepted_values:
              values:
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

**What each field does:**
- `name: stg_orders` — must exactly match the filename `stg_orders.sql` (without `.sql`)
- `description` — free text shown in `dbt docs`. Use `>` for multi-line strings
- `columns: - name:` — list every column the model outputs
- `tests:` — list of test names to run against this column
- `accepted_values: values:` — the allowed list. dbt fails if any row has a value outside this list

---

## Concept 4 — The Four Built-in dbt Tests

dbt ships with four generic tests you can attach to any column in any YAML file — no SQL required.

### 1. `not_null`

```yaml
- name: order_id
  tests:
    - not_null
```

The SQL dbt generates and runs behind the scenes:

```sql
SELECT COUNT(*)
FROM ECOMM_DB.STAGING.STG_ORDERS
WHERE order_id IS NULL
```

PASS if count = 0. FAIL if count > 0 (at least one NULL found).

Use on: primary keys, foreign keys, any column that must always have a value.

---

### 2. `unique`

```yaml
- name: order_id
  tests:
    - unique
```

The SQL dbt generates and runs behind the scenes:

```sql
SELECT COUNT(*) FROM (
    SELECT order_id
    FROM ECOMM_DB.STAGING.STG_ORDERS
    WHERE order_id IS NOT NULL
    GROUP BY order_id
    HAVING COUNT(*) > 1
)
```

PASS if count = 0 (no duplicates). FAIL if count > 0 (at least one duplicate found).

Use on: primary keys — any column that must be unique across all rows.

---

### 3. `accepted_values`

```yaml
- name: status
  tests:
    - accepted_values:
        values: ['CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED']
```

The SQL dbt generates and runs behind the scenes:

```sql
SELECT COUNT(*) FROM (
    SELECT DISTINCT status
    FROM ECOMM_DB.STAGING.STG_ORDERS
    WHERE status NOT IN ('CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED')
      AND status IS NOT NULL
)
```

PASS if count = 0. FAIL if count > 0 (a value outside the allowed list was found).

Use on: status columns, category columns, any enum-like field with a known set of valid values.

---

### 4. `relationships`

```yaml
- name: customer_id
  tests:
    - relationships:
        to: ref('stg_customers')
        field: customer_id
```

The SQL dbt generates and runs behind the scenes:

```sql
SELECT COUNT(*) FROM (
    SELECT customer_id
    FROM ECOMM_DB.STAGING.STG_ORDERS
    WHERE customer_id NOT IN (
        SELECT customer_id FROM ECOMM_DB.STAGING.STG_CUSTOMERS
    )
    AND customer_id IS NOT NULL
)
```

PASS if count = 0. FAIL if count > 0 (orphan rows — foreign key has no matching primary key).

Use on: foreign key columns to verify referential integrity between models.

---

## Concept 5 — Running Tests

```bash
dbt test
dbt test --select stg_orders
dbt build
```

`dbt test` runs only tests — does not rebuild models.
`dbt test --select stg_orders` runs tests only for that one model.
`dbt build` runs `dbt run` + `dbt test` together in dependency order — if a model's tests fail, its downstream models are skipped.

### Understanding test output

```
1 of 9 PASS test not_null_stg_orders_order_id .............. [PASS in 0.45s]
2 of 9 PASS test unique_stg_orders_order_id ................ [PASS in 0.38s]
3 of 9 PASS test accepted_values_stg_orders_status ......... [PASS in 0.41s]
...
Done. PASS=9 WARN=0 ERROR=0 SKIP=0 TOTAL=9
```

Each test name is auto-generated as `{test_type}_{model_name}_{column_name}`.

### What to do when a test fails

```
4 of 9 FAIL 2  test not_null_stg_orders_customer_name
```

The number after `FAIL` is the row count that violated the test — here, 2 rows have a NULL `customer_name`.

Find the failing rows in Snowsight:

```sql
SELECT *
FROM ECOMM_DB.STAGING.STG_ORDERS
WHERE customer_name IS NULL;
```

Once found, trace back to `ORDERS_RAW` to decide: fix the source data (EL pipeline issue) or fix the model SQL.

---

## Concept 6 — Incremental Models

### Why incremental?

With `materialized='table'`, dbt does a full rebuild on every run:

```sql
CREATE OR REPLACE TABLE ECOMM_DB.MARTS.ORDERS_MART AS (
    SELECT ... FROM ECOMM_DB.STAGING.STG_ORDERS GROUP BY order_date
)
```

For large tables this scans millions of rows on every run. An incremental model only processes rows that are new since the last run.

**How it works:**
- First run: dbt creates the full table (like `materialized='table'`)
- Every subsequent run: dbt applies a WHERE filter to get only new/changed rows, then INSERTs or MERGEs them into the existing table

### Create an incremental staging model

Create `models/staging/stg_orders_incremental.sql`:

```sql
{{
    config(
        materialized = 'incremental',
        unique_key   = 'order_id'
    )
}}

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

{% if is_incremental() %}
WHERE updated_at > (
    SELECT MAX(updated_at)
    FROM {{ this }}
)
{% endif %}
```

**Line-by-line explanation:**

`{{ config(materialized='incremental') }}` — tells dbt to build this model as an incremental table instead of rebuilding it from scratch every run.

`unique_key = 'order_id'` — if a row with this `order_id` already exists in the target table, UPDATE it (MERGE) instead of inserting a duplicate. Without `unique_key`, dbt would INSERT every row even if it already exists.

`{% if is_incremental() %}` — Jinja statement block. `is_incremental()` returns `TRUE` after the first full build. On the first run it returns `FALSE` — the WHERE block is skipped and all rows are loaded. On every subsequent run it returns `TRUE` — only the WHERE block rows are processed.

`{{ this }}` — refers to the incremental table itself (`ECOMM_DB.STAGING.STG_ORDERS_INCREMENTAL`). Used to query the existing table's MAX timestamp so we know which rows are already loaded.

`WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})` — filters the source to only rows with a newer `updated_at` than the latest row already in the table. This is how incremental logic works: compare source against what is already loaded.

`{% endif %}` — closes the Jinja if block.

### Run the incremental model

```bash
dbt run --select stg_orders_incremental
```

First run: builds the full table. Subsequent runs: only new rows.

```bash
dbt run --select stg_orders_incremental --full-refresh
```

`--full-refresh` forces dbt to drop and rebuild the entire table from scratch — use this when you fix a bug in historical logic or change the model's schema.

---

## Concept 7 — Jinja Templating Basics

dbt uses **Jinja** — a Python templating language — inside SQL files. Everything between `{{ }}` or `{% %}` is Jinja that dbt evaluates before sending SQL to Snowflake.

### `{{ }}` — expressions (output a value into SQL)

```sql
FROM {{ source('raw_data', 'orders_raw') }}
FROM {{ ref('stg_orders') }}
{{ config(materialized='table') }}
```

Each `{{ }}` block is replaced by its output value in the final SQL sent to Snowflake. `config()` is a special case — it outputs nothing in the SQL but tells dbt how to build the model.

### `{% %}` — statements (logic, no output)

```sql
{% if is_incremental() %}
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

`{% %}` blocks control the SQL structure. They produce no output themselves — only the SQL between them is included or excluded based on the condition.

### `{{ this }}` — reference to the current model's table

Inside an incremental model, `{{ this }}` resolves to the existing target table. You cannot use `ref('stg_orders_incremental')` here because that would create a circular dependency — the model would depend on itself.

---

## Concept 8 — Model Documentation

### Add descriptions to the mart model

Create `models/marts/schema.yml`:

```yaml
version: 2

models:
  - name: orders_mart
    description: >
      Daily order summary aggregated from stg_orders.
      One row per order_date. Used by the BI team for revenue dashboards.

    columns:
      - name: order_date
        description: "The calendar date of the orders summarized in this row"
        tests:
          - not_null
          - unique

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
        description: "Sum of total_amount for all orders on this date, in USD"
        tests:
          - not_null

      - name: avg_order_value
        description: "Average total_amount per order on this date, in USD"
```

### Generate and view the docs

```bash
dbt docs generate
dbt docs serve
```

`dbt docs generate` reads all `schema.yml` files and queries Snowflake for column metadata. Output goes to `target/catalog.json`.

`dbt docs serve` opens `http://localhost:8080`.

**What you will see:**
- Each model page: description, column list with descriptions and test names
- Sources tab: `raw_data` → `orders_raw` with freshness config
- Lineage graph (bottom right): `[source: orders_raw] → stg_orders → orders_mart`

Stop the server with `Ctrl+C`.

---

## Final File Structure After Day 2

```
ecomm_dbt/
├── dbt_project.yml
├── models/
│   ├── staging/
│   │   ├── sources.yml
│   │   ├── schema.yml
│   │   ├── stg_orders.sql
│   │   └── stg_orders_incremental.sql
│   └── marts/
│       ├── schema.yml
│       └── orders_mart.sql
```

---

## Summary

| Concept | Key takeaway |
|---|---|
| `sources.yml` | Declare raw tables once — `source()` resolves them everywhere |
| `loaded_at_field` + `freshness` | dbt checks MAX(updated_at) against warn/error thresholds |
| `schema.yml` | Descriptions + tests live alongside your models |
| `not_null` | Fails if any row has NULL in this column |
| `unique` | Fails if any value appears more than once |
| `accepted_values` | Fails if a value outside the allowed list appears |
| `relationships` | Fails if a foreign key has no matching row in the referenced model |
| `dbt test` | Runs all tests; `dbt build` runs models + tests in dependency order |
| `materialized='incremental'` | First run = full build; subsequent runs = only new rows |
| `is_incremental()` | TRUE after the first build, FALSE on first run |
| `{{ this }}` | Inside incremental models — refers to the existing target table |
| `{{ }}` | Jinja expression — outputs a value into the SQL |
| `{% %}` | Jinja statement — controls SQL structure (if/for), no output |
