# Day 2 — Interview Solutions: Testing, Documentation, and Advanced Models

> Complete answers. Read after attempting on your own.

---

**Q1 — Purpose of `schema.yml`. What two things can you define inside it?**

A `schema.yml` file is a YAML configuration file that sits alongside your `.sql` model files. It has two purposes:

1. **Descriptions** — human-readable text for models and their columns, which appears in `dbt docs`. This is the documentation layer: analysts and stakeholders can look up what every column means without asking the data engineer.

2. **Tests** — data quality assertions attached to columns. When you run `dbt test`, dbt reads these YAML declarations and generates SQL to verify them against your warehouse data.

The file name can be anything ending in `.yml` — `schema.yml` is just the convention. You can have one per folder or one per model.

---

**Q2 — The four built-in generic tests.**

| Test | What it checks |
|---|---|
| `not_null` | Every row has a non-NULL value in this column |
| `unique` | No two rows share the same value in this column |
| `accepted_values` | Every value in this column is in a predefined allowed list |
| `relationships` | Every value in this column exists as a key in another model (referential integrity) |

---

**Q3 — SQL dbt generates for `not_null` test on `order_id` in `stg_orders`.**

```sql
-- dbt auto-generates and runs this query:
SELECT COUNT(*)
FROM (
    SELECT order_id
    FROM ECOMM_DB.DBT_DEV.STG_ORDERS
    WHERE order_id IS NULL
) dbt_internal_test

-- If COUNT(*) = 0 → PASS  (no NULLs found)
-- If COUNT(*) > 0 → FAIL  (the count tells you how many NULL rows exist)
```

The test name dbt assigns is: `not_null_stg_orders_order_id`
Format: `{test_type}_{model_name}_{column_name}`

---

**Q4 — `dbt test` vs `dbt build`.**

`dbt test` runs only the data quality tests defined in schema YAML files. It does not create or update any models.

`dbt build` runs `dbt run` and `dbt test` together, in dependency order per model:
- Build `stg_orders` → test `stg_orders` → build `orders_mart` → test `orders_mart`

If `stg_orders` tests fail, dbt skips building `orders_mart` — you don't want to propagate bad data downstream. `dbt build` is what most production pipelines run because it guarantees tests pass before downstream models are populated.

---

**Q5 — `FAIL 3  test not_null_stg_orders_customer_name` — what does 3 mean?**

The `3` is the **number of rows** that violated the test — 3 rows in `stg_orders` have `customer_name IS NULL`.

To find which rows:

```sql
-- Option 1: query the model directly in Snowsight
SELECT *
FROM ECOMM_DB.DBT_DEV.STG_ORDERS
WHERE customer_name IS NULL;

-- Option 2: check the compiled test SQL that dbt saved
-- The file is at: target/compiled/ecomm_dbt/models/staging/schema.yml/
-- File name: not_null_stg_orders_customer_name.sql
-- Open it and run it in Snowsight to see the failing rows
```

Once you find the rows, trace back to `ORDERS_RAW` to understand whether:
- The source data has NULLs (data quality issue upstream — fix the EL pipeline)
- Your model's WHERE clause or JOIN is introducing NULLs (fix the model SQL)

---

**Q6 — `dbt source freshness` — how does dbt know if a source is stale?**

`dbt source freshness` connects to Snowflake and runs:
```sql
SELECT MAX(updated_at) AS last_loaded   -- updated_at = your loaded_at_field
FROM ECOMM_DB.RAW_DATA.ORDERS_RAW
```

It compares `last_loaded` to `CURRENT_TIMESTAMP()` to calculate how many hours/days ago data was last updated.

Two fields in `sources.yml` control the thresholds:

```yaml
freshness:
  warn_after:          # how old before dbt issues a WARNING (run still passes)
    count: 24
    period: hour       # warn if no new data in 24 hours
  error_after:         # how old before dbt issues an ERROR (run fails)
    count: 48
    period: hour       # hard fail if no new data in 48 hours
```

You also need `loaded_at_field: updated_at` on the source table to tell dbt which column to use for the MAX() query.

---

**Q7 — What is an incremental model? First run vs subsequent runs.**

An **incremental model** (`materialized='incremental'`) is a table that is built fully on the first run, then only updated with new or changed rows on every subsequent run.

**First run (`is_incremental()` = FALSE):**
- dbt creates the table with `CREATE TABLE AS SELECT ...`
- The full SELECT runs — all rows from the source are processed
- No WHERE filter is applied

**Subsequent runs (`is_incremental()` = TRUE):**
- The table already exists
- dbt applies your `{% if is_incremental() %}` WHERE filter to select only new rows
- Those rows are INSERTed (or MERGEd if `unique_key` is set) into the existing table
- Rows already in the table are untouched

Result: instead of scanning 10 million rows every run, you scan only the 500 new rows that arrived since the last run.

---

**Q8 — Write the incremental model `stg_orders_incremental` with full explanation.**

```sql
-- models/staging/stg_orders_incremental.sql

{{
    config(
        -- materialized='incremental': first run = full CREATE TABLE AS SELECT
        -- subsequent runs = INSERT/MERGE only new rows
        materialized = 'incremental',

        -- unique_key='order_id': if a row with this order_id already exists in the table,
        -- UPDATE it (MERGE) instead of inserting a second copy.
        -- Without unique_key, dbt only does INSERT — existing rows with same order_id
        -- would be duplicated if the source sends the same row again.
        unique_key = 'order_id'
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
-- This block is ONLY included on runs after the first full build.
-- On the first run, is_incremental() = FALSE, so this WHERE clause is skipped
-- and dbt processes all rows from orders_raw.
--
-- On subsequent runs, is_incremental() = TRUE:
--   {{ this }} = the incremental table that already exists (stg_orders_incremental)
--   MAX(updated_at) = the latest timestamp already loaded into the table
--   The WHERE filters the source to only rows newer than what we already have
WHERE updated_at > (
    SELECT MAX(updated_at)
    FROM {{ this }}
)
{% endif %}
```

---

**Q9 — What does `{{ this }}` resolve to? Why not use `ref('stg_orders_incremental')`?**

`{{ this }}` resolves to the **fully-qualified name of the current model's existing table** — the table that dbt is in the process of incrementally updating. For example: `ECOMM_DB.DBT_DEV.STG_ORDERS_INCREMENTAL`.

You cannot use `ref('stg_orders_incremental')` because:
- `ref()` is for **referencing other models** — using it on the current model creates a circular dependency (the model depends on itself)
- dbt would detect this circular reference and raise an error
- `{{ this }}` is specifically designed for self-reference within incremental models — it bypasses the dependency graph and directly targets the existing table

---

**Q10 — Difference between `{% if %}` and `{{ }}` in Jinja.**

`{{ }}` is an **expression block** — it evaluates the expression inside and **outputs** the result directly into the SQL:

```sql
FROM {{ source('raw_data', 'orders_raw') }}
-- outputs: FROM ECOMM_DB.RAW_DATA.ORDERS_RAW

FROM {{ ref('stg_orders') }}
-- outputs: FROM ECOMM_DB.DBT_DEV.STG_ORDERS
```

`{% %}` is a **statement block** — it contains logic (if/for/set) that **controls the SQL structure** but produces no output itself:

```sql
{% if is_incremental() %}
WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
-- The {% if %} and {% endif %} tags produce no SQL
-- Only the WHERE line inside is included (or excluded) based on the condition
```

Rule of thumb: `{{ }}` = "put this value here", `{% %}` = "decide what to include".

---

**Q11 — Incremental model running for 6 months, bug found in transformation logic. How to fix?**

Run a **full refresh**:

```bash
dbt run --select stg_orders_incremental --full-refresh
```

`--full-refresh` tells dbt to:
1. Drop the existing incremental table
2. Rebuild it from scratch with `CREATE TABLE AS SELECT`
3. `is_incremental()` returns FALSE — the WHERE filter is skipped
4. All 6 months of source data is reprocessed with the corrected logic

This is the standard recovery path for incremental models when historical data is affected. The trade-off is cost: a full refresh can be expensive on very large tables. For tables with years of data, you may instead write a backfill query that only corrects the affected rows.

---

**Q12 — New status `RETURNED` added. What breaks, what to change, in what order?**

**What breaks:** The `accepted_values` test on `status` will FAIL because `RETURNED` is not in the allowed list. If you run `dbt build`, the test failure will cause downstream models to be SKIPPED.

**What to change:**

1. Add `RETURNED` to the `accepted_values` list in `schema.yml`:
```yaml
- accepted_values:
    values:
      - 'CONFIRMED'
      - 'SHIPPED'
      - 'DELIVERED'
      - 'CANCELLED'
      - 'RETURNED'    # ← add this
```

2. Update `orders_mart.sql` to handle the new status in the CASE statements:
```sql
SUM(CASE WHEN status = 'RETURNED' THEN 1 ELSE 0 END) AS returned_orders
```

3. Update `schema.yml` description for the `status` column to document the new value.

**Order:** Update `schema.yml` first (stops the test failure), then update the mart model (adds the new column), then run `dbt build` to verify everything passes.

---

**Q13 — `sources.yml` vs `schema.yml`. Can they be in the same file?**

| | `sources.yml` | `schema.yml` |
|---|---|---|
| Declares | Raw tables loaded externally (not by dbt) | dbt models you wrote |
| Top-level key | `sources:` | `models:` |
| Used by | `{{ source('name', 'table') }}` | `{{ ref('model_name') }}` |
| Freshness checks | Yes — `loaded_at_field` and `freshness:` | No |

**Yes, you can put them in the same file** — dbt YAML files support both `sources:` and `models:` top-level keys in the same file. Convention is to separate them (sources in `sources.yml`, models in `schema.yml`) for readability, but dbt does not enforce this. The file name does not matter — only the YAML keys matter.

---

**Q14 — How does a new analyst find out what `total_amount` means?**

Using **dbt docs**:

```bash
# Data engineer already ran this (usually in CI/CD):
dbt docs generate && dbt docs serve
```

The analyst opens `http://localhost:8080` (or the hosted dbt Cloud docs URL), navigates to the `stg_orders` model, and reads the column-level description:

```
total_amount — Total order value in USD — quantity * unit_price.
Computed at order time, not recalculated. Pre-tax, in USD.
```

This is exactly why descriptions in `schema.yml` matter — they turn the docs site into a self-service data dictionary. The analyst gets the answer immediately without opening Slack or interrupting the data engineer.

---

**Q15 — `unique_key='order_id'`. Order status changes from CONFIRMED to SHIPPED. What happens?**

Because `unique_key='order_id'` is set, dbt performs a **MERGE** (not a plain INSERT) on the next run:

```sql
-- dbt generates something like this in Snowflake:
MERGE INTO ECOMM_DB.DBT_DEV.STG_ORDERS_INCREMENTAL AS target
USING (
    -- the incremental SELECT with the WHERE filter
    SELECT order_id, customer_id, ..., 'SHIPPED' AS status, ...
    FROM ECOMM_DB.RAW_DATA.ORDERS_RAW
    WHERE updated_at > (SELECT MAX(updated_at) FROM target)
) AS source
ON target.order_id = source.order_id     -- match on the unique_key

WHEN MATCHED THEN
    UPDATE SET                           -- row already exists → UPDATE it
        target.status     = source.status,     -- CONFIRMED → SHIPPED
        target.updated_at = source.updated_at,
        ...

WHEN NOT MATCHED THEN
    INSERT (order_id, customer_id, ..., status, ...)   -- new row → INSERT it
    VALUES (source.order_id, source.customer_id, ..., source.status, ...)
```

**Result:** The existing row for that `order_id` is **updated in place** — `status` changes from `CONFIRMED` to `SHIPPED`. No duplicate row is created.

Without `unique_key`, dbt would do a plain INSERT — the old row (CONFIRMED) stays and a new row (SHIPPED) is added, giving you two rows for the same order. `unique_key` prevents this.
