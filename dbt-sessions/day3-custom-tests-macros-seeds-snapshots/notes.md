# Day 3 — Custom Tests, Macros, Seeds, and Snapshots

## What You Will Learn

1. **Seeds** — Load static CSV files into Snowflake as tables with `dbt seed`
2. **Custom singular tests** — Write your own SQL test files for business logic that built-in tests cannot express
3. **Custom generic tests** — Write reusable parameterised tests in Jinja that work on any model/column
4. **Macros** — Write reusable Jinja functions called from models, tests, and other macros
5. **Snapshots** — Capture row-level history (SCD Type 2) — track how a record changed over time

> **Starting point:** The `ecomm_dbt` project from Day 2.
> Today we build on top of a **brand-new self-contained dataset** (`products` + `employees`) loaded via seeds — no dependency on existing Snowflake tables.

---

## The Dataset We Use Today

We introduce two new entities loaded entirely via dbt seeds — no manual Snowflake setup required:

| Table | What it represents |
|---|---|
| `products` | A product catalogue — id, name, category, price, stock_quantity, is_active |
| `employees` | An employee list — id, name, department, salary, manager_id, status |

These are realistic enough to demonstrate all test types, macros, and snapshot scenarios.

---

## Concept 1 — Seeds

### What is a seed?

A **seed** is a CSV file inside the `seeds/` folder of your dbt project. When you run `dbt seed`, dbt:
1. Reads the CSV
2. Creates a table in Snowflake (in the schema defined in `dbt_project.yml`)
3. Truncates and reloads it on every `dbt seed` run

Seeds are for **small, slow-changing reference data** — lookup tables, test fixtures, exchange rates, country codes. They are not for large or frequently changing data.

### File naming — can I name seeds anything?

Yes. The CSV filename becomes the table name in Snowflake. `products.csv` → `SEEDS.PRODUCTS` (or whatever schema you configure).

### Create the seed files

**`seeds/products.csv`**

```csv
product_id,product_name,category,unit_price,stock_quantity,is_active
1,Laptop Pro 15,Electronics,1299.99,50,true
2,Wireless Mouse,Electronics,29.99,200,true
3,Standing Desk,Furniture,599.99,30,true
4,USB-C Hub,Electronics,49.99,150,true
5,Office Chair,Furniture,399.99,25,true
6,Webcam HD,Electronics,89.99,80,true
7,Notebook Set,Stationery,12.99,500,true
8,Mechanical Keyboard,Electronics,129.99,60,true
9,Monitor 27inch,Electronics,449.99,40,true
10,Desk Lamp,Furniture,39.99,120,true
11,Discontinued Widget,Gadgets,9.99,0,false
12,Legacy Adapter,Electronics,4.99,5,false
```

**`seeds/employees.csv`**

```csv
employee_id,employee_name,department,salary,manager_id,status
1,Alice Johnson,Engineering,95000,NULL,active
2,Bob Smith,Engineering,88000,1,active
3,Carol White,Marketing,72000,NULL,active
4,Dave Brown,Engineering,91000,1,active
5,Eve Davis,Marketing,68000,3,active
6,Frank Miller,Sales,65000,NULL,active
7,Grace Wilson,Sales,61000,6,active
8,Henry Moore,Engineering,78000,1,active
9,Ivy Taylor,HR,70000,NULL,active
10,Jack Anderson,HR,64000,9,active
11,Karen Thomas,Sales,59000,6,active
12,Leo Jackson,Marketing,55000,3,active
```

### Configure seed schema in `dbt_project.yml`

Add a `seeds:` block so seed tables land in a `SEEDS` schema instead of the default staging schema:

```yaml
seeds:
  ecomm_dbt:
    +schema: SEEDS
```

Full updated `dbt_project.yml`:

```yaml
name: 'ecomm_dbt'
version: '1.0.0'
config-version: 2

profile: 'ecomm_dbt'

model-paths: ["models"]
seed-paths:  ["seeds"]
test-paths:  ["tests"]
macro-paths: ["macros"]

models:
  ecomm_dbt:
    staging:
      +materialized: view
      +schema: STAGING
    marts:
      +materialized: table
      +schema: MARTS

seeds:
  ecomm_dbt:
    +schema: SEEDS
```

### Run the seeds

```bash
dbt seed
```

Expected output:
```
1 of 2 START seed file SEEDS.products .............. [RUN]
1 of 2 OK loaded seed file SEEDS.products .......... [INSERT 12 in 1.23s]
2 of 2 START seed file SEEDS.employees ............. [RUN]
2 of 2 OK loaded seed file SEEDS.employees ......... [INSERT 12 in 0.98s]
Done. PASS=2 WARN=0 ERROR=0 SKIP=0 TOTAL=2
```

Verify in Snowflake:
```sql
SELECT * FROM ECOMM_DB.SEEDS.PRODUCTS;
SELECT * FROM ECOMM_DB.SEEDS.EMPLOYEES;
```

### Create staging models for the seeds

**`models/staging/stg_products.sql`**

```sql
SELECT
    product_id,
    product_name,
    category,
    unit_price,
    stock_quantity,
    is_active
FROM {{ ref('products') }}
```

**`models/staging/stg_employees.sql`**

```sql
SELECT
    employee_id,
    employee_name,
    department,
    salary,
    manager_id,
    status
FROM {{ ref('employees') }}
```

`{{ ref('products') }}` points to the seed table — same `ref()` syntax as models. dbt tracks seeds in the lineage graph exactly like models.

```bash
dbt run --select stg_products stg_employees
```

---

## Concept 2 — Custom Singular Tests

### What is a singular test?

A **singular test** is a `.sql` file inside the `tests/` folder. It contains a single `SELECT` that returns the **rows that fail the test**.

- If the SELECT returns **0 rows** → PASS
- If the SELECT returns **any rows** → FAIL

No YAML config needed. The filename becomes the test name.

Singular tests are for **business logic** that built-in tests (`not_null`, `unique`) cannot express — things like "price must be positive", "stock cannot be negative", "salary must be within band".

### Test 1 — `tests/assert_product_price_is_positive.sql`

Business rule: `unit_price` must be greater than zero.

```sql
-- Returns rows where unit_price is zero or negative — test fails if any row returned
SELECT
    product_id,
    product_name,
    unit_price
FROM {{ ref('stg_products') }}
WHERE unit_price <= 0
```

### Test 2 — `tests/assert_stock_not_negative.sql`

Business rule: `stock_quantity` can be zero (out of stock) but never negative.

```sql
SELECT
    product_id,
    product_name,
    stock_quantity
FROM {{ ref('stg_products') }}
WHERE stock_quantity < 0
```

### Test 3 — `tests/assert_employee_salary_in_range.sql`

Business rule: salary must be between 30,000 and 500,000.

```sql
SELECT
    employee_id,
    employee_name,
    salary
FROM {{ ref('stg_employees') }}
WHERE salary < 30000
   OR salary > 500000
```

### Test 4 — `tests/assert_manager_exists.sql`

Business rule: if `manager_id` is set, that manager must exist as an `employee_id` in the same table.

```sql
SELECT
    e.employee_id,
    e.employee_name,
    e.manager_id
FROM {{ ref('stg_employees') }} e
WHERE e.manager_id IS NOT NULL
  AND e.manager_id NOT IN (
      SELECT employee_id FROM {{ ref('stg_employees') }}
  )
```

### Run singular tests

```bash
dbt test --select stg_products stg_employees
```

Expected output:
```
1 of 4 PASS assert_product_price_is_positive .......... [PASS in 0.42s]
2 of 4 PASS assert_stock_not_negative ................. [PASS in 0.38s]
3 of 4 PASS assert_employee_salary_in_range ........... [PASS in 0.41s]
4 of 4 PASS assert_manager_exists ..................... [PASS in 0.39s]
Done. PASS=4 WARN=0 ERROR=0 SKIP=0 TOTAL=4
```

### Break a singular test intentionally

Add a row with negative price to the CSV, re-seed, and re-test:

```csv
13,Broken Product,Gadgets,-5.00,10,true
```

```bash
dbt seed
dbt test --select stg_products
```

Expected failure:
```
FAIL 1  assert_product_price_is_positive
```

The number `1` = how many rows the SELECT returned (one product with negative price).

Remove row 13 from the CSV and re-seed to restore clean state.

---

## Concept 3 — Custom Generic Tests

### What is a generic test?

A **generic test** is a reusable test written as a Jinja macro in the `macros/` folder. Unlike singular tests (one SQL file = one test on one model), a generic test can be **attached to any column on any model** via YAML — just like `not_null` or `unique`.

The pattern:

```sql
{% test <test_name>(model, column_name, <extra_params>) %}
    SELECT ...
    FROM {{ model }}
    WHERE <condition that identifies failing rows>
{% endtest %}
```

### Generic Test 1 — `not_negative`

Business rule: a numeric column must never be negative.

**`macros/test_not_negative.sql`**

```sql
{% test not_negative(model, column_name) %}

SELECT
    {{ column_name }}
FROM {{ model }}
WHERE {{ column_name }} < 0

{% endtest %}
```

Attach it in `schema.yml` exactly like a built-in test:

```yaml
- name: stock_quantity
  tests:
    - not_negative
```

### Generic Test 2 — `accepted_range`

Business rule: a numeric column must be between `min_value` and `max_value`.

**`macros/test_accepted_range.sql`**

```sql
{% test accepted_range(model, column_name, min_value, max_value) %}

SELECT
    {{ column_name }}
FROM {{ model }}
WHERE {{ column_name }} < {{ min_value }}
   OR {{ column_name }} > {{ max_value }}

{% endtest %}
```

Attach it in `schema.yml`:

```yaml
- name: unit_price
  tests:
    - accepted_range:
        min_value: 0.01
        max_value: 10000
```

### Generic Test 3 — `not_empty_string`

Business rule: a text column must not be an empty string `''`.

**`macros/test_not_empty_string.sql`**

```sql
{% test not_empty_string(model, column_name) %}

SELECT
    {{ column_name }}
FROM {{ model }}
WHERE TRIM({{ column_name }}) = ''

{% endtest %}
```

### Full `models/staging/schema.yml` with generic tests attached

```yaml
version: 2

models:
  - name: stg_products
    description: "Cleaned product catalogue loaded from seeds."
    columns:
      - name: product_id
        tests:
          - not_null
          - unique
      - name: product_name
        tests:
          - not_null
          - not_empty_string
      - name: category
        tests:
          - not_null
          - not_empty_string
      - name: unit_price
        tests:
          - not_null
          - not_negative
          - accepted_range:
              min_value: 0.01
              max_value: 10000
      - name: stock_quantity
        tests:
          - not_null
          - not_negative

  - name: stg_employees
    description: "Cleaned employee list loaded from seeds."
    columns:
      - name: employee_id
        tests:
          - not_null
          - unique
      - name: employee_name
        tests:
          - not_null
          - not_empty_string
      - name: department
        tests:
          - not_null
      - name: salary
        tests:
          - not_null
          - accepted_range:
              min_value: 30000
              max_value: 500000
      - name: status
        tests:
          - not_null
          - accepted_values:
              values: ['active', 'inactive', 'terminated']
```

### Run all tests (singular + generic)

```bash
dbt test --select stg_products stg_employees
```

---

## Concept 4 — Macros

### What is a macro?

A **macro** is a reusable Jinja function defined in the `macros/` folder and called from models, tests, or other macros using `{{ macro_name(...) }}`.

Macros eliminate copy-paste SQL. Instead of writing the same CASE WHEN block in five models, you write it once as a macro and call it everywhere.

### Macro 1 — `cents_to_dollars`

Converts an integer cents value to a decimal dollars value. Useful when source systems store prices as integers (e.g. `12999` = $129.99).

**`macros/cents_to_dollars.sql`**

```sql
{% macro cents_to_dollars(column_name) %}
    ({{ column_name }} / 100.0)
{% endmacro %}
```

Usage in a model:

```sql
SELECT
    product_id,
    product_name,
    {{ cents_to_dollars('unit_price_cents') }} AS unit_price
FROM {{ ref('products_raw') }}
```

### Macro 2 — `classify_price`

Returns a price tier label based on the value of a column.

**`macros/classify_price.sql`**

```sql
{% macro classify_price(column_name) %}
    CASE
        WHEN {{ column_name }} < 50    THEN 'Budget'
        WHEN {{ column_name }} < 200   THEN 'Mid-range'
        WHEN {{ column_name }} < 1000  THEN 'Premium'
        ELSE 'Enterprise'
    END
{% endmacro %}
```

Usage in a model:

```sql
SELECT
    product_id,
    product_name,
    unit_price,
    {{ classify_price('unit_price') }} AS price_tier
FROM {{ ref('stg_products') }}
```

### Macro 3 — `is_working_day` (conditional macro with parameters)

Returns TRUE if a date column falls on a weekday (Mon–Fri).

**`macros/is_working_day.sql`**

```sql
{% macro is_working_day(column_name) %}
    DAYOFWEEK({{ column_name }}) NOT IN (0, 6)
{% endmacro %}
```

Usage:

```sql
SELECT
    order_id,
    order_date,
    {{ is_working_day('order_date') }} AS placed_on_working_day
FROM {{ ref('stg_orders') }}
```

### Macro 4 — `generate_surrogate_key` (multi-column composite key)

Generates a surrogate key by hashing multiple columns together. Useful when a table has no single primary key.

**`macros/generate_surrogate_key.sql`**

```sql
{% macro generate_surrogate_key(columns) %}
    MD5(
        CONCAT_WS('|',
            {% for col in columns %}
                COALESCE(CAST({{ col }} AS VARCHAR), '')
                {%- if not loop.last %}, {% endif %}
            {% endfor %}
        )
    )
{% endmacro %}
```

Usage:

```sql
SELECT
    {{ generate_surrogate_key(['product_id', 'category']) }} AS product_key,
    product_id,
    product_name,
    category
FROM {{ ref('stg_products') }}
```

### Create a mart model that uses macros

**`models/marts/products_mart.sql`**

```sql
SELECT
    product_id,
    product_name,
    category,
    unit_price,
    stock_quantity,
    is_active,
    {{ classify_price('unit_price') }}                          AS price_tier,
    {{ generate_surrogate_key(['product_id', 'category']) }}    AS product_key,
    CASE WHEN stock_quantity = 0  THEN 'Out of Stock'
         WHEN stock_quantity < 20 THEN 'Low Stock'
         ELSE 'In Stock'
    END                                                         AS stock_status
FROM {{ ref('stg_products') }}
```

```bash
dbt run --select products_mart
```

Verify in Snowflake:
```sql
SELECT product_name, unit_price, price_tier, stock_status
FROM ECOMM_DB.MARTS.PRODUCTS_MART;
```

---

## Concept 5 — Snapshots

### What is a snapshot?

A **snapshot** captures the state of a row at a point in time. Every time you run `dbt snapshot`, dbt:
1. Compares the current source data to the last snapshot
2. If a row has **changed** → closes the old version (sets `dbt_valid_to`) and inserts a new version
3. If a row is **new** → inserts it
4. If a row is **unchanged** → does nothing

This is **SCD Type 2** (Slowly Changing Dimension Type 2) — you keep the full history of every change, not just the current state.

### Snapshot columns dbt adds automatically

| Column | Meaning |
|---|---|
| `dbt_scd_id` | Unique hash for this version of this row |
| `dbt_updated_at` | When dbt detected this version |
| `dbt_valid_from` | When this version became current |
| `dbt_valid_to` | When this version was superseded (NULL = current row) |

### Create a snapshot for products

Snapshots live in the `snapshots/` folder.

**`snapshots/products_snapshot.sql`**

```sql
{% snapshot products_snapshot %}

{{
    config(
        target_schema = 'SNAPSHOTS',
        unique_key    = 'product_id',
        strategy      = 'check',
        check_cols    = ['unit_price', 'stock_quantity', 'is_active']
    )
}}

SELECT
    product_id,
    product_name,
    category,
    unit_price,
    stock_quantity,
    is_active
FROM {{ ref('stg_products') }}

{% endsnapshot %}
```

**Config explained:**

| Config | Value | Meaning |
|---|---|---|
| `target_schema` | `SNAPSHOTS` | Snapshot tables land in a dedicated schema |
| `unique_key` | `product_id` | The column that identifies a unique row |
| `strategy` | `check` | Detect changes by comparing specific columns |
| `check_cols` | `['unit_price', 'stock_quantity', 'is_active']` | Only these columns trigger a new snapshot version |

**Two strategies available:**

| Strategy | How it detects changes | Use when |
|---|---|---|
| `check` | Compares specific columns you list | Source has no `updated_at` column, or you only care about specific field changes |
| `timestamp` | Compares an `updated_at` timestamp | Source has a reliable `updated_at` column |

### Run the snapshot for the first time

```bash
dbt snapshot
```

Expected output:
```
1 of 1 START snapshot SNAPSHOTS.products_snapshot ........... [RUN]
1 of 1 OK snapshotted SNAPSHOTS.products_snapshot ........... [INSERT 12 in 2.1s]
```

Verify in Snowflake — all 12 rows, all with `dbt_valid_to = NULL` (all current):

```sql
SELECT product_id, product_name, unit_price, dbt_valid_from, dbt_valid_to
FROM ECOMM_DB.SNAPSHOTS.PRODUCTS_SNAPSHOT
ORDER BY product_id;
```

### Simulate a change and re-snapshot

**Step 1 — update the seed CSV** to simulate a price change and a product going inactive:

```csv
-- Change product_id=1: price from 1299.99 → 1199.99
1,Laptop Pro 15,Electronics,1199.99,50,true

-- Change product_id=11: is_active from false → true (restocked)
11,Discontinued Widget,Gadgets,9.99,15,true
```

**Step 2 — re-seed and re-run the staging model:**

```bash
dbt seed
dbt run --select stg_products
```

**Step 3 — run the snapshot again:**

```bash
dbt snapshot
```

Expected output:
```
1 of 1 OK snapshotted SNAPSHOTS.products_snapshot ........... [UPDATE 2 in 1.9s]
```

**Step 4 — verify the history in Snowflake:**

```sql
SELECT
    product_id,
    product_name,
    unit_price,
    is_active,
    dbt_valid_from,
    dbt_valid_to
FROM ECOMM_DB.SNAPSHOTS.PRODUCTS_SNAPSHOT
WHERE product_id IN (1, 11)
ORDER BY product_id, dbt_valid_from;
```

Expected result:

```
product_id  unit_price  is_active  dbt_valid_from        dbt_valid_to
1           1299.99     true       2026-11-25 10:00:00   2026-11-25 10:30:00  ← old version, now closed
1           1199.99     true       2026-11-25 10:30:00   NULL                 ← current version
11          9.99        false      2026-11-25 10:00:00   2026-11-25 10:30:00  ← old version, now closed
11          9.99        true       2026-11-25 10:30:00   NULL                 ← current version (restocked)
```

**Query only the current version of every product:**

```sql
SELECT *
FROM ECOMM_DB.SNAPSHOTS.PRODUCTS_SNAPSHOT
WHERE dbt_valid_to IS NULL;
```

**Query what a product looked like at a specific point in time:**

```sql
SELECT *
FROM ECOMM_DB.SNAPSHOTS.PRODUCTS_SNAPSHOT
WHERE product_id = 1
  AND dbt_valid_from <= '2026-11-25 10:15:00'
  AND (dbt_valid_to > '2026-11-25 10:15:00' OR dbt_valid_to IS NULL);
```

---

## Final File Structure After Day 3

```
ecomm_dbt/
├── dbt_project.yml
├── seeds/
│   ├── products.csv
│   └── employees.csv
├── models/
│   ├── staging/
│   │   ├── sources.yml
│   │   ├── schema.yml          ← updated with generic tests
│   │   ├── stg_orders.sql
│   │   ├── stg_orders_incremental.sql
│   │   ├── stg_products.sql    ← new
│   │   └── stg_employees.sql   ← new
│   └── marts/
│       ├── schema.yml
│       ├── orders_mart.sql
│       └── products_mart.sql   ← new
├── tests/
│   ├── assert_product_price_is_positive.sql   ← singular test
│   ├── assert_stock_not_negative.sql          ← singular test
│   ├── assert_employee_salary_in_range.sql    ← singular test
│   └── assert_manager_exists.sql             ← singular test
├── macros/
│   ├── generate_schema_name.sql
│   ├── cents_to_dollars.sql
│   ├── classify_price.sql
│   ├── is_working_day.sql
│   ├── generate_surrogate_key.sql
│   ├── test_not_negative.sql       ← generic test macro
│   ├── test_accepted_range.sql     ← generic test macro
│   └── test_not_empty_string.sql   ← generic test macro
└── snapshots/
    └── products_snapshot.sql
```

---

## Key Differences — Singular vs Generic Tests

| | Singular test | Generic test |
|---|---|---|
| **Location** | `tests/` folder | `macros/` folder |
| **File prefix** | No convention | Must start with `test_` |
| **Reusable?** | No — one test, one model | Yes — attach to any column on any model via YAML |
| **Parameters** | No — hardcoded model/column | Yes — `column_name`, `min_value`, etc. |
| **Best for** | Complex business logic specific to one model | Validation rules you apply repeatedly across models |
| **How it runs** | `dbt test` picks it up automatically from `tests/` | `dbt test` runs it when referenced in `schema.yml` |

---

## Run Everything Together

```bash
dbt seed                                # load products.csv and employees.csv
dbt run --select stg_products stg_employees products_mart
dbt test --select stg_products stg_employees
dbt snapshot                            # capture first products snapshot
```

Simulate a data change, then:

```bash
dbt seed
dbt run --select stg_products
dbt snapshot                            # dbt_valid_to filled for changed rows
```

---

## Summary

| Concept | Key takeaway |
|---|---|
| Seeds | CSV files in `seeds/` → `dbt seed` loads them as tables. Filename = table name. |
| `ref()` on seeds | Same `{{ ref('products') }}` syntax as models — seeds appear in lineage graph |
| Singular tests | SQL file in `tests/` — SELECT returns failing rows. 0 rows = PASS. |
| Generic tests | Jinja macro in `macros/` prefixed `test_` — reusable, parameterised, attached in YAML |
| Macros | Jinja functions in `macros/` — called with `{{ macro_name(...) }}` in models or tests |
| `{% for %}` in macros | Loops over a list of columns — used in `generate_surrogate_key` |
| Snapshots | SCD Type 2 — `dbt snapshot` tracks row-level history with `dbt_valid_from` / `dbt_valid_to` |
| `strategy: check` | Detects changes by comparing listed columns — no `updated_at` needed |
| `strategy: timestamp` | Detects changes via an `updated_at` column — faster on large tables |
| `dbt_valid_to IS NULL` | Filter to get only the current version of each row from a snapshot |
