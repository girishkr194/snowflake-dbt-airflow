# Day 2 — Practice Exercises: Testing, Documentation, and Advanced Models

> All exercises build on the `ecomm_dbt` project from Day 1.
> `ECOMM_DB.RAW_DATA.ORDERS_RAW` must exist with at least 10 rows.
> Run all dbt commands from inside the `ecomm_dbt/` project directory.
> Estimated total time: 60–75 minutes.

---

## Pre-Check

```bash
dbt run
```

Expected: `PASS=2 (stg_orders, orders_mart)  ERROR=0  SKIP=0`

---

## Exercise 0 — Fix Dynamic Schema Routing

### The Problem

`profiles.yml` has only one `schema:` field. If you set `schema: STAGING`, every model — staging and marts — lands in `ECOMM_DB.STAGING`. If you change it to `MARTS`, staging breaks. You need each folder to write to its own schema automatically.

### Step 1: Check where models are landing right now

```sql
SHOW VIEWS  IN SCHEMA ECOMM_DB.STAGING;
SHOW TABLES IN SCHEMA ECOMM_DB.STAGING;
SHOW TABLES IN SCHEMA ECOMM_DB.MARTS;
```

Notice that `ORDERS_MART` may be in `STAGING` instead of `MARTS` depending on what your `profiles.yml` currently has. This is the problem we are fixing.

### Step 2: Add `+schema` per folder in `dbt_project.yml`

Open `dbt_project.yml` and update the `models:` section:

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

`+schema: STAGING` tells dbt: every model inside `models/staging/` should go into the `STAGING` schema.
`+schema: MARTS` tells dbt: every model inside `models/marts/` should go into the `MARTS` schema.

If you run `dbt run` right now, models will land in schemas like `STAGING_STAGING` and `STAGING_MARTS` — dbt's default behaviour prepends the `profiles.yml` schema to the folder schema. We fix that in the next step.

### Step 3: Override `generate_schema_name` macro

Create the file `macros/generate_schema_name.sql`:

```sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- if custom_schema_name is none -%}
        {{ target.schema }}
    {%- else -%}
        {{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

This macro tells dbt:
- If a folder has `+schema` set → use that value exactly as written (e.g. `STAGING`, `MARTS`)
- If no `+schema` is set → fall back to the schema from `profiles.yml`

Without this macro override, dbt's default logic produces `STAGING_MARTS` (concatenating both). With it, dbt uses the folder schema as-is.

### Step 4: Run and verify

```bash
dbt run
```

```sql
SHOW VIEWS  IN SCHEMA ECOMM_DB.STAGING;
SHOW TABLES IN SCHEMA ECOMM_DB.MARTS;
```

Expected:
- `ECOMM_DB.STAGING.STG_ORDERS` → view
- `ECOMM_DB.MARTS.ORDERS_MART` → table

Each model folder now writes to its own dedicated Snowflake schema automatically. You never need to edit `profiles.yml` to switch schemas — just add `+schema:` to a folder in `dbt_project.yml`.

### Step 5: Check the compiled SQL to confirm

```bash
type target\compiled\ecomm_dbt\models\marts\orders_mart.sql
```

The FROM clause should reference `ECOMM_DB.STAGING.STG_ORDERS` (not `ECOMM_DB.STAGING_STAGING.STG_ORDERS`), confirming the macro is working correctly.

---

## Exercise 1 — Add a Source Declaration

Right now `stg_orders.sql` hardcodes `FROM ECOMM_DB.RAW_DATA.ORDERS_RAW`.
We will fix that by declaring a source and using `{{ source() }}`.

### Step 1: Create `models/staging/sources.yml`

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
```

**What each field does:**
- `name: raw_data` — the logical name used in `{{ source('raw_data', ...) }}`. Does not have to match the Snowflake schema name.
- `database: ECOMM_DB` — the Snowflake database
- `schema: RAW_DATA` — the Snowflake schema
- `loaded_at_field: updated_at` — the column dbt checks for freshness. dbt runs `SELECT MAX(updated_at)` and compares to current time.
- `warn_after` / `error_after` — how stale is too stale (24h = warning, 48h = error)
- `tables: - name: orders_raw` — the actual Snowflake table name

### Step 2: Update `models/staging/stg_orders.sql`

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
- `'raw_data'` matches the `name:` key under `sources:` in your YAML
- `'orders_raw'` matches the `name:` key under `tables:` in your YAML

### Step 3: Run the model and verify

```bash
dbt run --select stg_orders
```

Expected: `OK  created sql view model STAGING.stg_orders`

Verify in Snowsight:
```sql
SELECT * FROM ECOMM_DB.STAGING.STG_ORDERS ORDER BY order_id;
```

### Step 4: Check source freshness

```bash
dbt source freshness
```

Expected: `PASS  freshness of raw_data.orders_raw`

If you see WARN or ERROR, it means `ORDERS_RAW.updated_at` has not been updated within the threshold.

---

## Exercise 2 — Add Tests to the Staging Model

### Step 1: Create `models/staging/schema.yml`

```yaml
version: 2

models:
  - name: stg_orders
    description: >
      Cleaned orders from ORDERS_RAW. Single source of truth for all order data.

    columns:
      - name: order_id
        description: "Primary key — unique integer identifier for each order"
        tests:
          - not_null
          - unique

      - name: customer_id
        description: "Foreign key — integer ID of the customer who placed this order"
        tests:
          - not_null

      - name: customer_name
        description: "Full name of the customer at the time the order was placed"
        tests:
          - not_null

      - name: product
        description: "Name of the product that was ordered"
        tests:
          - not_null

      - name: quantity
        description: "Number of units ordered — always a positive integer"
        tests:
          - not_null

      - name: unit_price
        description: "Price per unit in USD at the time of the order"
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
        description: "Timestamp of the last modification to this row"
        tests:
          - not_null
```

**What each field does:**
- `name: stg_orders` — must exactly match the filename `stg_orders.sql` (without `.sql`)
- `tests: - not_null` — dbt generates `SELECT COUNT(*) WHERE column IS NULL`. Fails if count > 0.
- `tests: - unique` — dbt generates a GROUP BY query. Fails if any value appears more than once.
- `accepted_values: values:` — dbt checks every row's status is in this list. Fails if any other value is found.

### Step 2: Run the tests

```bash
dbt test --select stg_orders
```

Expected: `PASS=9  WARN=0  ERROR=0  SKIP=0`

Each test is named: `{test_type}_{model_name}_{column_name}`.
Example: `not_null_stg_orders_order_id`, `unique_stg_orders_order_id`

### Step 3: Deliberately break a test to see failure

Insert a bad row in Snowsight:

```sql
INSERT INTO ECOMM_DB.RAW_DATA.ORDERS_RAW
    (order_id, customer_id, customer_name, product, quantity, unit_price, total_amount, status, order_date)
VALUES (9999, 99, 'Test User', 'Widget', 1, 10.00, 10.00, 'PENDING', '2024-06-01');
```

`'PENDING'` is not in our `accepted_values` list.

Re-run:
```bash
dbt run --select stg_orders
dbt test --select stg_orders
```

Expected: `FAIL 1  test accepted_values_stg_orders_status__...`

The `1` = one row violated the test. Find it:

```sql
SELECT *
FROM ECOMM_DB.STAGING.STG_ORDERS
WHERE status NOT IN ('CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED');
```

Clean up:
```sql
DELETE FROM ECOMM_DB.RAW_DATA.ORDERS_RAW WHERE order_id = 9999;
```

Re-run:
```bash
dbt run --select stg_orders
dbt test --select stg_orders
```

Expected: `PASS=9` again.

---

## Exercise 3 — Build + Test Together with `dbt build`

```bash
dbt build
```

`dbt build` runs `dbt run` + `dbt test` per model in dependency order:
`stg_orders` runs → `stg_orders` tests pass → `orders_mart` runs → `orders_mart` tests run

If `stg_orders` tests fail, `orders_mart` is skipped — dbt prevents bad data from flowing downstream.

Expected output:
```
1 of 10 START sql view model STAGING.stg_orders ................. OK
2 of 10 START test not_null_stg_orders_order_id ................. PASS
...
10 of 10 START sql table model MARTS.orders_mart ................ OK
Done. PASS=10 WARN=0 ERROR=0 SKIP=0 TOTAL=10
```

Now add tests to the mart. Create `models/marts/schema.yml`:

```yaml
version: 2

models:
  - name: orders_mart
    description: >
      Daily order summary. One row per order_date.
      Rebuilt as a full table on every dbt run.

    columns:
      - name: order_date
        description: "Calendar date — each date appears exactly once in this table"
        tests:
          - not_null
          - unique

      - name: total_orders
        description: "Count of all orders placed on this date, all statuses included"
        tests:
          - not_null

      - name: total_revenue
        description: "Sum of total_amount for all orders on this date, in USD"
        tests:
          - not_null
```

Run build again:
```bash
dbt build
```

Now includes tests for both `stg_orders` and `orders_mart`.

---

## Exercise 4 — Incremental Model

### Step 1: Create `models/staging/stg_orders_incremental.sql`

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
- `materialized = 'incremental'` — first run builds the full table; subsequent runs only process new rows
- `unique_key = 'order_id'` — if `order_id` already exists, dbt MERGEs (updates) it instead of inserting a duplicate
- `{% if is_incremental() %}` — TRUE after the first build, FALSE on first run. The WHERE clause is only included on subsequent runs.
- `{{ this }}` — refers to the incremental table itself (`ECOMM_DB.STAGING.STG_ORDERS_INCREMENTAL`). Used to find the latest `updated_at` already loaded.
- `{% endif %}` — closes the Jinja if block

### Step 2: First run — builds the full table

```bash
dbt run --select stg_orders_incremental
```

`is_incremental()` = FALSE — all rows from ORDERS_RAW are loaded. dbt creates `ECOMM_DB.STAGING.STG_ORDERS_INCREMENTAL` as a table.

Verify:
```sql
SELECT COUNT(*) FROM ECOMM_DB.STAGING.STG_ORDERS_INCREMENTAL;
```

Should match the row count in `ECOMM_DB.RAW_DATA.ORDERS_RAW`.

### Step 3: Insert a new order

```sql
INSERT INTO ECOMM_DB.RAW_DATA.ORDERS_RAW
    (order_id, customer_id, customer_name, product, quantity, unit_price, total_amount, status, order_date)
VALUES (6001, 61, 'Sam Rivera', 'Wireless Charger', 2, 29.99, 59.98, 'CONFIRMED', '2024-05-01');
```

### Step 4: Second run — only new rows

```bash
dbt run --select stg_orders_incremental
```

`is_incremental()` = TRUE — dbt applies the WHERE filter, picks up order 6001, and INSERTs it.

Verify:
```sql
SELECT * FROM ECOMM_DB.STAGING.STG_ORDERS_INCREMENTAL WHERE order_id = 6001;

SELECT COUNT(*) FROM ECOMM_DB.STAGING.STG_ORDERS_INCREMENTAL;
```

Count should be original + 1.

### Step 5: Update an existing order

```sql
UPDATE ECOMM_DB.RAW_DATA.ORDERS_RAW
SET status = 'SHIPPED', updated_at = CURRENT_TIMESTAMP()
WHERE order_id = 6001;
```

Run again:
```bash
dbt run --select stg_orders_incremental
```

The WHERE filter picks up order 6001 (its `updated_at` just changed). Because `unique_key='order_id'`, dbt MERGEs: the existing row is updated in place — no duplicate.

Verify:
```sql
SELECT order_id, status, updated_at
FROM ECOMM_DB.STAGING.STG_ORDERS_INCREMENTAL
WHERE order_id = 6001;
```

Status should now be `SHIPPED`. Count should be unchanged.

### Step 6: Force a full refresh

```bash
dbt run --select stg_orders_incremental --full-refresh
```

Drops and rebuilds the entire table from scratch. Use this when you fix a bug in historical transformation logic.

---

## Exercise 5 — Generate and Browse Documentation

### Step 1: Generate the catalog

```bash
dbt docs generate
```

Reads all `schema.yml` files and queries Snowflake for column metadata. Output: `target/catalog.json`.

### Step 2: Serve the docs

```bash
dbt docs serve
```

Opens `http://localhost:8080`.

### Step 3: Explore the docs UI

Navigate through these pages:

1. **Project tab** — expand `models/staging/` and `models/marts/`
   Click `stg_orders` — see the model description and column list

2. **Column tests** — click the `order_id` column
   Should show tests: `not_null`, `unique`

3. **Sources tab** — click `raw_data` → `orders_raw`
   See the source description and freshness config

4. **Lineage graph** — click the graph icon (bottom right of any model page)
   Should show: `[source: orders_raw] → stg_orders → orders_mart`
   `stg_orders_incremental` should also appear connected to `orders_raw`

5. **Search** — type `total_amount`
   Both `stg_orders` and `orders_mart` columns should appear

Stop the server with `Ctrl+C`.

---

## Final Verification

```bash
dbt build
```

Expected: `PASS = all models + all tests  WARN=0  ERROR=0  SKIP=0`

Verify objects in Snowsight:
```sql
SHOW VIEWS  IN SCHEMA ECOMM_DB.STAGING;
SHOW TABLES IN SCHEMA ECOMM_DB.STAGING;
SHOW TABLES IN SCHEMA ECOMM_DB.MARTS;
```

Expected:
- `ECOMM_DB.STAGING.STG_ORDERS` → view
- `ECOMM_DB.STAGING.STG_ORDERS_INCREMENTAL` → table
- `ECOMM_DB.MARTS.ORDERS_MART` → table

---

## Summary

| Exercise | What you verified |
|---|---|
| 1 | `sources.yml` created, `stg_orders.sql` updated to use `{{ source() }}`, freshness check passing |
| 2 | `schema.yml` with column descriptions and tests, `dbt test` passing, deliberate failure and recovery |
| 3 | `dbt build` = run + test in dependency order; mart `schema.yml` with tests |
| 4 | Incremental model — first full build, INSERT on new row, MERGE on update, `--full-refresh` |
| 5 | `dbt docs generate` + `dbt docs serve`, lineage graph, source node, column-level descriptions |
