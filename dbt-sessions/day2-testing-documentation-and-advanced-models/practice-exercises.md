# Day 2 — Practice Exercises: Testing, Documentation, and Advanced Models

> All exercises build on the `ecomm_dbt` project from Day 1.
> ORDERS_RAW must exist in Snowflake with at least 10 rows.
> Run all dbt commands from inside the `ecomm_dbt/` project directory.
> Estimated total time: 60–75 minutes.

---

## Pre-Check

```bash
# Confirm your Day 1 project still runs clean
dbt run
# Expected: PASS=2 (stg_orders, orders_mart)  ERROR=0  SKIP=0
```

---

## Exercise 1 — Add a Source Declaration

Right now `stg_orders.sql` hardcodes `FROM ECOMM_DB.RAW_DATA.ORDERS_RAW`.
We will fix that by declaring a source and using `{{ source() }}`.

### Step 1: Create `models/staging/sources.yml`

```yaml
version: 2                        # always required — tells dbt this is a v2 config file

sources:
  - name: raw_data                # logical name — used in {{ source('raw_data', ...) }}
    database: ECOMM_DB            # Snowflake database that contains the raw table
    schema: RAW_DATA              # Snowflake schema that contains the raw table
    description: >
      Raw tables loaded from source systems via Snowpipe or COPY INTO.
      Do not modify these tables directly.

    loaded_at_field: updated_at   # column dbt uses to check source freshness
                                  # dbt runs: SELECT MAX(updated_at) FROM orders_raw
                                  # and compares it to CURRENT_TIMESTAMP()

    freshness:                    # define what "stale" means for this source
      warn_after:                 # issue a WARNING if data is older than this
        count: 24
        period: hour
      error_after:                # issue an ERROR if data is older than this
        count: 48
        period: hour

    tables:
      - name: orders_raw          # actual Snowflake table name
        description: >
          One row per customer order. Contains all statuses including CANCELLED.
```

### Step 2: Update `models/staging/stg_orders.sql`

Replace the hardcoded `FROM` with `{{ source() }}`:

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
-- {{ source('raw_data', 'orders_raw') }} resolves to ECOMM_DB.RAW_DATA.ORDERS_RAW
-- 'raw_data' must match the `name:` key in sources.yml
-- 'orders_raw' must match the `name:` key under tables: in sources.yml
WHERE order_id IS NOT NULL
```

### Step 3: Run the model and verify

```bash
dbt run --select stg_orders
# Expected: OK  created sql view model DBT_DEV.stg_orders
```

### Step 4: Check source freshness

```bash
dbt source freshness
# Expected: PASS freshness of raw_data.orders_raw
# If WARN or ERROR: your ORDERS_RAW data hasn't been updated recently
```

---

## Exercise 2 — Add Tests to the Staging Model

### Step 1: Create `models/staging/schema.yml`

```yaml
version: 2

models:
  - name: stg_orders              # must exactly match the filename stg_orders.sql (no .sql)
    description: >
      Cleaned orders from ORDERS_RAW. Single source of truth for all order data.
      WHERE order_id IS NOT NULL filters out any corrupt rows from the source.

    columns:
      - name: order_id
        description: "Primary key — unique integer identifier for each order. Never NULL."
        tests:
          - not_null              # dbt checks: SELECT COUNT(*) WHERE order_id IS NULL → must be 0
          - unique                # dbt checks: no duplicate order_id values → must be 0 duplicates

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
          - accepted_values:      # checks that every status value is in this list
              values:             # dbt fails if any row has a status NOT in this list
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

### Step 2: Run the tests

```bash
dbt test --select stg_orders
# Expected: PASS=9 — one test per column (not_null for each + unique + accepted_values)
```

### Step 3: Read the output carefully

For each passing test you should see:
```
PASS test not_null_stg_orders_order_id
PASS test unique_stg_orders_order_id
PASS test not_null_stg_orders_customer_id
...
PASS test accepted_values_stg_orders_status__CONFIRMED__SHIPPED__DELIVERED__CANCELLED
```

### Step 4: Deliberately break a test to see failure

Insert a bad row in Snowsight:
```sql
-- Run in Snowsight
INSERT INTO ECOMM_DB.RAW_DATA.ORDERS_RAW
    (order_id, customer_id, customer_name, product, quantity, unit_price, total_amount, status, order_date)
VALUES (9999, 99, 'Test User', 'Widget', 1, 10.00, 10.00, 'PENDING', '2024-06-01');
-- 'PENDING' is NOT in our accepted_values list
```

Re-run the model and test:
```bash
dbt run --select stg_orders
dbt test --select stg_orders
# Expected: FAIL 1  test accepted_values_stg_orders_status__...
#           The number 1 = one row with an unexpected status value
```

Find the failing row:
```sql
-- Run in Snowsight to see which rows failed
SELECT *
FROM ECOMM_DB.DBT_DEV.STG_ORDERS
WHERE status NOT IN ('CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED');
-- Should return order_id = 9999
```

Clean up:
```sql
-- Remove the bad row from the source
DELETE FROM ECOMM_DB.RAW_DATA.ORDERS_RAW WHERE order_id = 9999;
```

Re-run:
```bash
dbt run --select stg_orders
dbt test --select stg_orders
# Expected: PASS=9 again
```

---

## Exercise 3 — Build + Test Together with `dbt build`

```bash
# dbt build = dbt run + dbt test per model, in dependency order
# stg_orders runs → stg_orders tests pass → orders_mart runs → orders_mart tests (none yet)
dbt build

# Expected:
# 1 of 2 START sql view model DBT_DEV.stg_orders .................. OK
# 2 of 9 START test not_null_stg_orders_order_id .................. PASS
# ...
# 10 of 10 START sql table model DBT_DEV.orders_mart .............. OK
# Done. PASS=10 WARN=0 ERROR=0 SKIP=0 TOTAL=10
```

Now add tests to the mart model. Create `models/marts/schema.yml`:

```yaml
version: 2

models:
  - name: orders_mart
    description: >
      Daily order summary. One row per order_date.
      Rebuilt as a full table on every dbt run from stg_orders.

    columns:
      - name: order_date
        description: "Calendar date — each date appears exactly once in this table"
        tests:
          - not_null
          - unique               # the mart aggregates by order_date so each date = 1 row

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
# Now runs tests for both stg_orders and orders_mart
```

---

## Exercise 4 — Incremental Model

### Step 1: Create `models/staging/stg_orders_incremental.sql`

```sql
-- models/staging/stg_orders_incremental.sql

{{                                    -- open Jinja block
    config(
        materialized = 'incremental', -- first run: CREATE TABLE AS SELECT (all rows)
                                      -- subsequent runs: INSERT/MERGE only new rows
        unique_key   = 'order_id'     -- if order_id already exists in the target table,
                                      -- UPDATE that row instead of inserting a duplicate
    )
}}                                    -- close Jinja block

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
-- This block is only included AFTER the first run.
-- is_incremental() returns TRUE when the target table already exists.
-- On the first run, is_incremental() = FALSE → no WHERE clause → all rows loaded.
--
-- On subsequent runs:
--   {{ this }} = the incremental table we are updating (stg_orders_incremental)
--   We find the most recent updated_at already loaded, then filter to rows newer than that.
WHERE updated_at > (
    SELECT MAX(updated_at)   -- find the latest timestamp already in the incremental table
    FROM {{ this }}           -- {{ this }} = ECOMM_DB.DBT_DEV.STG_ORDERS_INCREMENTAL
)
{% endif %}
```

### Step 2: First run — builds the full table

```bash
dbt run --select stg_orders_incremental
# is_incremental() = FALSE — all rows from ORDERS_RAW are loaded
# dbt creates: ECOMM_DB.DBT_DEV.STG_ORDERS_INCREMENTAL as a TABLE
```

Verify:
```sql
SELECT COUNT(*) FROM ECOMM_DB.DBT_DEV.STG_ORDERS_INCREMENTAL;
-- Should match ORDERS_RAW count
```

### Step 3: Insert a new order into ORDERS_RAW

```sql
-- Run in Snowsight
INSERT INTO ECOMM_DB.RAW_DATA.ORDERS_RAW
    (order_id, customer_id, customer_name, product, quantity, unit_price, total_amount, status, order_date)
VALUES (6001, 61, 'Sam Rivera', 'Wireless Charger', 2, 29.99, 59.98, 'CONFIRMED', '2024-05-01');
```

### Step 4: Second run — only new rows

```bash
dbt run --select stg_orders_incremental
# is_incremental() = TRUE — only rows with updated_at newer than MAX(updated_at) are processed
# dbt MERGEs on order_id — the new row (6001) is inserted
```

Verify:
```sql
-- New row should appear
SELECT * FROM ECOMM_DB.DBT_DEV.STG_ORDERS_INCREMENTAL WHERE order_id = 6001;

-- Count should be original + 1
SELECT COUNT(*) FROM ECOMM_DB.DBT_DEV.STG_ORDERS_INCREMENTAL;
```

### Step 5: Update an existing order status

```sql
-- Run in Snowsight — change order 6001 from CONFIRMED to SHIPPED
UPDATE ECOMM_DB.RAW_DATA.ORDERS_RAW
SET status = 'SHIPPED', updated_at = CURRENT_TIMESTAMP()
WHERE order_id = 6001;
```

Run again:
```bash
dbt run --select stg_orders_incremental
# The WHERE filter picks up the updated row (updated_at just changed)
# Because unique_key='order_id', dbt MERGEs: existing row is UPDATED, not duplicated
```

Verify:
```sql
-- Status should now be SHIPPED, not CONFIRMED
SELECT order_id, status, updated_at
FROM ECOMM_DB.DBT_DEV.STG_ORDERS_INCREMENTAL
WHERE order_id = 6001;

-- Still same total count — no duplicate row was inserted
SELECT COUNT(*) FROM ECOMM_DB.DBT_DEV.STG_ORDERS_INCREMENTAL;
```

### Step 6: Force a full refresh

```bash
# Use --full-refresh to drop and rebuild the entire incremental table from scratch
# Use this when: bug in historical logic, schema change, or data corruption
dbt run --select stg_orders_incremental --full-refresh

# is_incremental() = FALSE even though the table exists
# dbt drops the table, rebuilds it completely
```

---

## Exercise 5 — Generate and Browse Documentation

### Step 1: Generate the catalog

```bash
dbt docs generate
# Reads all schema.yml files + queries Snowflake for column metadata
# Writes output to target/catalog.json and target/manifest.json
```

### Step 2: Serve the docs

```bash
dbt docs serve
# Opens browser at http://localhost:8080
```

### Step 3: Explore the docs UI

Navigate through these pages:

1. **Project tab (left panel)** — expand `models/staging/` and `models/marts/`
   - Click `stg_orders` — see the description and column list with descriptions

2. **Column tests** — click `order_id` column in `stg_orders`
   - Should show: Tests: `not_null`, `unique`

3. **Sources tab** — click `raw_data` → `orders_raw`
   - Should show the source description and freshness config

4. **Lineage graph** — click the graph icon (bottom right of any model page)
   - Should show: `[source: orders_raw] → stg_orders → orders_mart`
   - `stg_orders_incremental` should also appear as a consumer of `orders_raw`

5. **Search** — type `total_amount` in the search bar
   - Both `stg_orders` and `orders_mart` columns should appear

Stop the server with `Ctrl+C`.

---

## Final Verification

```bash
# Run everything — all models + all tests
dbt build

# Expected:
# PASS = all models + all tests
# WARN = 0
# ERROR = 0
# SKIP = 0
```

```sql
-- Verify all objects exist in Snowsight
SHOW VIEWS IN SCHEMA ECOMM_DB.DBT_DEV;
-- stg_orders, stg_orders_incremental (wait — incremental is a TABLE not VIEW)

SHOW TABLES IN SCHEMA ECOMM_DB.DBT_DEV;
-- stg_orders_incremental, orders_mart, customer_summary
```

---

## Summary

| Exercise | What you verified |
|---|---|
| 1 | `sources.yml` created, `stg_orders.sql` updated to use `{{ source() }}`, freshness check passing |
| 2 | `schema.yml` with column descriptions and tests, `dbt test` passing, deliberate failure and recovery |
| 3 | `dbt build` = run + test together in order; mart schema.yml with tests |
| 4 | Incremental model: first full run, INSERT on new row, MERGE on update, `--full-refresh` |
| 5 | `dbt docs generate` + `dbt docs serve`, lineage graph, source node, column-level descriptions |
