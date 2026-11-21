# Day 1 — Practice Exercises: dbt Introduction and Setup

> All exercises build on the Snowflake tables from the Snowflake module (ECOMM_DB.RAW_DATA.ORDERS_RAW).
> Run all dbt commands from inside the `ecomm_dbt/` project directory.
> Estimated total time: 60–75 minutes.

---

## Pre-Check

Make sure these exist in Snowflake before starting:

```sql
-- Run in Snowsight to verify
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_RAW;
-- Should return at least 10 rows (from the Snowflake module)
```

---

## Exercise 1 — Install and Initialize

### Step 1: Install dbt with the Snowflake adapter

```bash
pip install dbt-snowflake

# Verify
dbt --version
```

Expected output:
```
Core:
  - installed: 1.x.x
  - latest:    1.x.x

Plugins:
  - snowflake: 1.x.x
```

### Step 2: Initialize the project

```bash
dbt init ecomm_dbt
```

When prompted:
- Adapter: `snowflake`
- Account: your Snowflake account identifier (e.g., `xy12345.us-east-1`)
- User: your Snowflake username
- Authentication: `password`
- Password: your Snowflake password
- Role: `SYSADMIN`
- Warehouse: `COMPUTE_WH`
- Database: `ECOMM_DB`
- Schema: `DBT_DEV`
- Threads: `4`

```bash
cd ecomm_dbt
```

### Step 3: Inspect the project structure

```bash
ls -la
# or on Windows:
dir
```

Confirm these exist:
- `dbt_project.yml`
- `models/`
- `macros/`
- `seeds/`
- `tests/`

### Step 4: Check where profiles.yml was created

```bash
# On Mac/Linux:
cat ~/.dbt/profiles.yml

# On Windows:
type %USERPROFILE%\.dbt\profiles.yml
```

Confirm it has your Snowflake credentials under the `ecomm_dbt` profile key.

### Step 5: Test the connection

```bash
dbt debug
```

Every line should show `OK`. If any line shows `ERROR`, fix the credentials in `profiles.yml` and re-run.

---

## Exercise 2 — Create the Staging Model

### Step 1: Clean up the example models

```bash
# Delete the generated example folder
# On Mac/Linux:
rm -rf models/example

# On Windows:
rmdir /s /q models\example
```

### Step 2: Create the staging directory and sources file

Create `models/staging/sources.yml`:

```yaml
version: 2

sources:
  - name: raw_data
    database: ECOMM_DB
    schema: RAW_DATA
    description: "Raw tables loaded from source systems"
    tables:
      - name: orders_raw
        description: "Raw orders — one row per order"
```

### Step 3: Create the staging model

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
FROM {{ source('raw_data', 'orders_raw') }}
WHERE order_id IS NOT NULL
```

### Step 4: Set staging materialization in dbt_project.yml

Open `dbt_project.yml` and update the `models:` section:

```yaml
models:
  ecomm_dbt:
    staging:
      +materialized: view
```

### Step 5: Run the staging model

```bash
dbt run --select stg_orders
```

Expected output:
```
1 of 1 START sql view model DBT_DEV.stg_orders .................. [RUN]
1 of 1 OK created sql view model DBT_DEV.stg_orders ............. [CREATE VIEW in 1.23s]

Finished running 1 view model in 0 hours 0 minutes and 2.xx seconds (0:00:02.xx).
Completed successfully.
Done. PASS=1 WARN=0 ERROR=0 SKIP=0 TOTAL=1
```

### Step 6: Verify in Snowsight

```sql
-- Run in Snowsight
SELECT * FROM ECOMM_DB.DBT_DEV.STG_ORDERS ORDER BY order_id;
-- Should show all rows from ORDERS_RAW
```

---

## Exercise 3 — Create the Mart Model

### Step 1: Create the marts directory and model

Create `models/marts/orders_mart.sql`:

```sql
{{ config(materialized='table') }}

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

### Step 2: Run all models

```bash
dbt run
```

Expected output:
```
1 of 2 START sql view model DBT_DEV.stg_orders .................. OK
2 of 2 START sql table model DBT_DEV.orders_mart ................ OK
Completed successfully.
Done. PASS=2 WARN=0 ERROR=0 SKIP=0 TOTAL=2
```

Observe: `stg_orders` runs before `orders_mart` — dbt resolved the dependency from `ref('stg_orders')`.

### Step 3: Verify in Snowsight

```sql
-- View should exist
SHOW VIEWS IN SCHEMA ECOMM_DB.DBT_DEV;
-- Table should exist
SHOW TABLES IN SCHEMA ECOMM_DB.DBT_DEV;

-- Query the mart
SELECT * FROM ECOMM_DB.DBT_DEV.ORDERS_MART ORDER BY order_date;
```

### Step 4: Check what SQL dbt actually generated

```bash
# dbt saves compiled SQL in target/compiled/
cat target/compiled/ecomm_dbt/models/marts/orders_mart.sql
# On Windows:
type target\compiled\ecomm_dbt\models\marts\orders_mart.sql
```

You will see the SELECT with `ref()` resolved to the actual schema-qualified table name.

---

## Exercise 4 — Selectors and Partial Runs

### Step 1: Run only staging models

```bash
dbt run --select staging.*
# Only stg_orders runs
```

### Step 2: Run stg_orders and everything downstream

```bash
dbt run --select stg_orders+
# stg_orders runs, then orders_mart runs (because it depends on stg_orders)
```

### Step 3: Simulate a source change

Pretend the raw schema is renamed. Update `sources.yml` temporarily:

```yaml
# Change schema from RAW_DATA to RAW_DATA_V2 (this will cause a failure)
schema: RAW_DATA_V2
```

Run:
```bash
dbt run
# Should fail — ECOMM_DB.RAW_DATA_V2.ORDERS_RAW does not exist
```

Observe: `stg_orders` fails, `orders_mart` is marked `SKIP` — not `ERROR` — because dbt skips models whose upstream dependencies failed.

Revert `sources.yml` back to `RAW_DATA` and re-run:
```bash
dbt run
# Back to: PASS=2 WARN=0 ERROR=0 SKIP=0 TOTAL=2
```

---

## Exercise 5 — Lineage and Docs

### Step 1: Generate documentation

```bash
dbt docs generate
```

This creates `target/catalog.json` with schema metadata for all models and sources.

### Step 2: Serve the docs locally

```bash
dbt docs serve
# Opens browser at http://localhost:8080
```

Navigate to:
- **Project** tab → see `stg_orders` and `orders_mart`
- Click `orders_mart` → scroll to **Depends On** section → see `stg_orders` listed
- Click the **Lineage Graph** button (bottom right) → see the DAG:
  ```
  ORDERS_RAW (source) → stg_orders → orders_mart
  ```

Stop the docs server with `Ctrl+C`.

---

## Exercise 6 — Add a Customer Summary Model

Create a second mart model to reinforce `ref()` usage.

Create `models/marts/customer_summary.sql`:

```sql
{{ config(materialized='table') }}

SELECT
    customer_id,
    customer_name,
    COUNT(*)          AS total_orders,
    SUM(total_amount) AS lifetime_value,
    MIN(order_date)   AS first_order_date,
    MAX(order_date)   AS last_order_date
FROM {{ ref('stg_orders') }}
GROUP BY customer_id, customer_name
ORDER BY lifetime_value DESC
```

Run and verify:
```bash
dbt run

# Expected: PASS=3 (stg_orders, orders_mart, customer_summary)
```

```sql
-- Verify in Snowsight
SELECT * FROM ECOMM_DB.DBT_DEV.CUSTOMER_SUMMARY ORDER BY lifetime_value DESC;
```

Regenerate docs and check the lineage — both `orders_mart` and `customer_summary` should now appear as children of `stg_orders`.

---

## Final Project Structure

After completing all exercises your project should look like this:

```
ecomm_dbt/
├── dbt_project.yml
├── models/
│   ├── staging/
│   │   ├── sources.yml
│   │   └── stg_orders.sql
│   └── marts/
│       ├── orders_mart.sql
│       └── customer_summary.sql
├── macros/
├── seeds/
├── tests/
└── target/
    └── compiled/   ← generated SQL (do not edit)
```

---

## Summary

| Exercise | What you verified |
|---|---|
| 1 | dbt installed, project initialized, connection working (`dbt debug`) |
| 2 | First staging model with `source()`, view materialization |
| 3 | Mart model with `ref()`, table materialization, dependency ordering |
| 4 | Selectors (`stg_orders+`, `staging.*`), behavior when upstream fails |
| 5 | Lineage graph in `dbt docs serve`, DAG visualization |
| 6 | Second mart model, both `orders_mart` and `customer_summary` depend on `stg_orders` |
