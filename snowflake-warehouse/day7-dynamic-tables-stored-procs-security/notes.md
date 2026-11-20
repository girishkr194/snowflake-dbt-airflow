# Day 7 — Dynamic Tables, Materialized Views, Stored Procedures & UDFs, Security Policies

## What You Will Learn

1. **Dynamic Tables** — Declarative, auto-refreshing derived tables. The modern alternative to stream + task pipelines.
2. **Materialized Views** — Pre-computed query results stored as a table, automatically kept in sync.
3. **Stored Procedures & UDFs** — Reusable SQL logic. Procedures for multi-step automation, UDFs for inline functions.
4. **Row Access Policies & Column Masking** — Control who sees which rows and which column values.

All examples use the Day 4 tables: `ECOMM_DB.RAW_DATA.ORDERS_RAW`, `ECOMM_DB.STAGING.STG_ORDERS`, `ECOMM_DB.MARTS.ORDERS_MART`.

---

## Concept 1 — Dynamic Tables

### What is a Dynamic Table?

A **Dynamic Table** is a table whose contents are automatically kept in sync with its source query. You write a SELECT statement — Snowflake runs it on a schedule you define (called **target lag**) and refreshes the table's contents.

```
Source table changes  ──►  Dynamic Table refreshes automatically  ──►  always up-to-date
```

This replaces the stream + task pattern for many pipeline use cases. Instead of:
- Create stream on ORDERS_RAW
- Create task that runs MERGE every minute
- Handle offsets, WHEN conditions, task chaining

You just write:
```sql
CREATE DYNAMIC TABLE STG_ORDERS_DT
    TARGET_LAG = '1 minute'
    WAREHOUSE  = COMPUTE_WH
AS
SELECT ... FROM ORDERS_RAW;
```

Snowflake handles the refresh automatically.

### Dynamic Table vs Stream + Task

| | Stream + Task | Dynamic Table |
|---|---|---|
| How you define it | CREATE STREAM + CREATE TASK with MERGE SQL | CREATE DYNAMIC TABLE with SELECT SQL |
| Refresh trigger | Task schedule (min 1 min) or SYSTEM$STREAM_HAS_DATA | Target lag (minimum a few seconds) |
| You write | MERGE logic (3 branches for insert/update/delete) | Simple SELECT — Snowflake figures out the delta |
| Chaining | AFTER clause — same schema required | Chain DTs: one DT queries another |
| Best for | Complex CDC logic, soft deletes, custom MERGE rules | Straightforward derived tables, aggregations |
| Minimum lag | 1 minute (task schedule) | Seconds (serverless) |

### Hands-On: Create a Dynamic Table

#### Step 1: Create a Dynamic Table on ORDERS_RAW

```sql
USE ROLE SYSADMIN;
USE WAREHOUSE COMPUTE_WH;
USE SCHEMA ECOMM_DB.STAGING;

-- Dynamic Table: always reflects current active orders from RAW
CREATE OR REPLACE DYNAMIC TABLE ECOMM_DB.STAGING.STG_ORDERS_DT
    TARGET_LAG = '1 minute'
    WAREHOUSE  = COMPUTE_WH
AS
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
FROM ECOMM_DB.RAW_DATA.ORDERS_RAW;
```

#### Step 2: Inspect and query it

```sql
-- List dynamic tables
SHOW DYNAMIC TABLES IN SCHEMA ECOMM_DB.STAGING;

-- Query it like a normal table
SELECT * FROM ECOMM_DB.STAGING.STG_ORDERS_DT ORDER BY order_id;

-- Check the refresh history
SELECT *
FROM TABLE(INFORMATION_SCHEMA.DYNAMIC_TABLE_REFRESH_HISTORY(
    NAME => 'ECOMM_DB.STAGING.STG_ORDERS_DT'
))
ORDER BY REFRESH_END_TIME DESC
LIMIT 5;
```

#### Step 3: Insert into source — observe automatic refresh

```sql
-- Insert a new row into ORDERS_RAW
USE SCHEMA ECOMM_DB.RAW_DATA;
INSERT INTO ORDERS_RAW
    (order_id, customer_id, customer_name, product, quantity,
     unit_price, total_amount, status, order_date)
VALUES (5001, 51, 'Eva Green', 'Smart Speaker', 1, 79.99, 79.99, 'CONFIRMED', '2024-04-01');

-- Wait ~1 minute (the target lag), then query the dynamic table
SELECT * FROM ECOMM_DB.STAGING.STG_ORDERS_DT WHERE order_id = 5001;
-- The new row appears automatically — no EXECUTE TASK, no manual MERGE
```

#### Step 4: Chain Dynamic Tables (like task DAG)

```sql
-- Daily summary mart as a Dynamic Table chained from STG_ORDERS_DT
CREATE OR REPLACE DYNAMIC TABLE ECOMM_DB.MARTS.ORDERS_MART_DT
    TARGET_LAG = '5 minutes'
    WAREHOUSE  = COMPUTE_WH
AS
SELECT
    order_date,
    COUNT(*)                                                AS total_orders,
    SUM(CASE WHEN status = 'CONFIRMED'  THEN 1 ELSE 0 END) AS confirmed_orders,
    SUM(CASE WHEN status = 'SHIPPED'    THEN 1 ELSE 0 END) AS shipped_orders,
    SUM(CASE WHEN status = 'DELIVERED'  THEN 1 ELSE 0 END) AS delivered_orders,
    SUM(CASE WHEN status = 'CANCELLED'  THEN 1 ELSE 0 END) AS cancelled_orders,
    SUM(total_amount)                                       AS total_revenue,
    AVG(total_amount)                                       AS avg_order_value
FROM ECOMM_DB.STAGING.STG_ORDERS_DT
GROUP BY order_date;

-- The chain: ORDERS_RAW → STG_ORDERS_DT (1 min lag) → ORDERS_MART_DT (5 min lag)
-- Snowflake propagates changes end-to-end automatically
SELECT * FROM ECOMM_DB.MARTS.ORDERS_MART_DT ORDER BY order_date;
```

#### Step 5: Suspend and resume a Dynamic Table

```sql
-- Suspend (stops automatic refresh, saves compute cost)
ALTER DYNAMIC TABLE ECOMM_DB.STAGING.STG_ORDERS_DT SUSPEND;

-- Resume
ALTER DYNAMIC TABLE ECOMM_DB.STAGING.STG_ORDERS_DT RESUME;

-- Manual refresh (force a refresh now regardless of lag)
ALTER DYNAMIC TABLE ECOMM_DB.STAGING.STG_ORDERS_DT REFRESH;
```

---

## Concept 2 — Materialized Views

### What is a Materialized View?

A **Materialized View (MV)** is a pre-computed query result stored physically as a table. When the underlying table changes, Snowflake automatically updates the MV in the background — you always query fresh results without paying for recomputation at query time.

```
Regular view:  query → reads ORDERS_RAW → computes result every time
Materialized:  query → reads pre-computed snapshot → instant, no compute at query time
```

### Materialized View vs Dynamic Table

Both store pre-computed results and auto-refresh. The difference is in **purpose, SQL flexibility, and control**.

| Feature | Materialized View | Dynamic Table |
|---|---|---|
| **Purpose** | Speed up a specific read-heavy query | Build a full pipeline layer (RAW → STG → MART) |
| **SQL allowed** | Simple SELECT — no subqueries, limited joins, no UNION | Any SELECT — subqueries, CTEs, UNION, complex joins |
| **Refresh trigger** | Automatically in background (Snowflake decides when) | Based on `TARGET_LAG` you set (e.g., `'1 minute'`) |
| **Manual refresh** | Not supported | `ALTER DYNAMIC TABLE ... REFRESH` |
| **Incremental updates** | Yes — only changed rows recomputed | Yes |
| **Chaining** | Cannot read from another MV | Can read from another Dynamic Table (DAG-style) |
| **Suspend / Resume** | Not supported | `ALTER DYNAMIC TABLE ... SUSPEND/RESUME` |
| **Control over lag** | No — Snowflake manages refresh timing | Yes — you set `TARGET_LAG` explicitly |
| **Snowflake edition** | Enterprise or higher | All editions (Standard and above) |
| **Cost** | Serverless background compute | Serverless background compute |

**Simple rule:**
- Use a **Materialized View** when you have one specific expensive query that runs frequently on a large table, and that query is simple enough to fit MV's restrictions.
- Use a **Dynamic Table** when you are building a transformation pipeline, need flexible SQL, need to chain transformations, or want explicit control over refresh lag.

```
Example: order status summary query runs 1000 times/day
  → MV: pre-computes once, all 1000 reads are instant ✔

Example: RAW → STAGING → MART pipeline
  → Dynamic Tables: chain them, Snowflake propagates changes end-to-end ✔
  → MV would not work here — cannot chain MVs, SQL is too restricted ✗
```

### Hands-On: Materialized View

```sql
USE ROLE SYSADMIN;
USE SCHEMA ECOMM_DB.STAGING;

-- Create a materialized view: pre-compute active order count per status
CREATE OR REPLACE MATERIALIZED VIEW ECOMM_DB.STAGING.MV_ORDER_STATUS_SUMMARY AS
SELECT
    status,
    COUNT(*)          AS order_count,
    SUM(total_amount) AS total_revenue
FROM ECOMM_DB.STAGING.STG_ORDERS
WHERE is_deleted = FALSE
GROUP BY status;

-- Query it — instant, no compute
SELECT * FROM ECOMM_DB.STAGING.MV_ORDER_STATUS_SUMMARY ORDER BY order_count DESC;

-- Inspect
SHOW MATERIALIZED VIEWS IN SCHEMA ECOMM_DB.STAGING;

-- Check the last refresh time
SELECT *
FROM TABLE(INFORMATION_SCHEMA.MATERIALIZED_VIEW_REFRESH_HISTORY(
    DATE_RANGE_START => DATEADD('hours', -1, CURRENT_TIMESTAMP()),
    BASE_OBJECT_NAME => 'STG_ORDERS'
));

-- Drop the MV
DROP MATERIALIZED VIEW ECOMM_DB.STAGING.MV_ORDER_STATUS_SUMMARY;
```

---

## Concept 3 — Stored Procedures & UDFs

### Stored Procedures

A **Stored Procedure** is reusable SQL logic stored in Snowflake. It can run multiple SQL statements, use variables, loops, conditionals, and return a result. Use it to automate multi-step operations you run repeatedly.

```sql
USE SCHEMA ECOMM_DB.RAW_DATA;

-- Procedure: suspend both Day 4 tasks in one call
CREATE OR REPLACE PROCEDURE ECOMM_DB.RAW_DATA.SUSPEND_PIPELINE()
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
BEGIN
    ALTER TASK ECOMM_DB.RAW_DATA.STG_TO_MART_TASK SUSPEND;
    ALTER TASK ECOMM_DB.RAW_DATA.RAW_TO_STG_TASK  SUSPEND;
    RETURN 'Pipeline suspended';
END;
$$;

-- Call it
CALL ECOMM_DB.RAW_DATA.SUSPEND_PIPELINE();

-- Procedure: resume in correct order (child first, root last)
CREATE OR REPLACE PROCEDURE ECOMM_DB.RAW_DATA.RESUME_PIPELINE()
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
BEGIN
    ALTER TASK ECOMM_DB.RAW_DATA.STG_TO_MART_TASK RESUME;
    ALTER TASK ECOMM_DB.RAW_DATA.RAW_TO_STG_TASK  RESUME;
    RETURN 'Pipeline resumed';
END;
$$;

CALL ECOMM_DB.RAW_DATA.RESUME_PIPELINE();
```

#### Procedure with parameters and logic

```sql
-- Procedure: load orders from a stage file and return how many rows were loaded
CREATE OR REPLACE PROCEDURE ECOMM_DB.RAW_DATA.LOAD_ORDERS_FROM_STAGE(file_name VARCHAR)
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
DECLARE
    rows_loaded INTEGER;
BEGIN
    COPY INTO ECOMM_DB.RAW_DATA.ORDERS_RAW
        (order_id, customer_id, customer_name, product, quantity,
         unit_price, total_amount, status, order_date)
    FROM @ECOMM_DB.RAW_DATA.ORDERS_RAW_STAGE
    FILES = (:file_name)
    FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1);

    SELECT COUNT(*) INTO :rows_loaded FROM ECOMM_DB.RAW_DATA.ORDERS_RAW;
    RETURN 'Loaded. Total rows in ORDERS_RAW: ' || rows_loaded;
END;
$$;

-- Call with a specific file
CALL ECOMM_DB.RAW_DATA.LOAD_ORDERS_FROM_STAGE('orders_batch1.csv.gz');
```

### User-Defined Functions (UDFs)

A **UDF** is a reusable function you can call inline in a SELECT statement — just like `SUM()`, `UPPER()`, etc. Use it for custom calculations or transformations you repeat across queries.

```sql
USE SCHEMA ECOMM_DB.RAW_DATA;

-- UDF: classify an order's total amount into a size bucket
CREATE OR REPLACE FUNCTION ECOMM_DB.RAW_DATA.ORDER_SIZE(total NUMBER)
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
    CASE
        WHEN total < 50   THEN 'SMALL'
        WHEN total < 200  THEN 'MEDIUM'
        WHEN total < 500  THEN 'LARGE'
        ELSE                   'ENTERPRISE'
    END
$$;

-- Use it in a query
SELECT
    order_id,
    customer_name,
    total_amount,
    ECOMM_DB.RAW_DATA.ORDER_SIZE(total_amount) AS order_size
FROM ECOMM_DB.RAW_DATA.ORDERS_RAW
ORDER BY total_amount DESC;
```

#### UDF with JavaScript (more complex logic)

```sql
-- UDF: mask middle digits of a phone number (data privacy)
CREATE OR REPLACE FUNCTION ECOMM_DB.RAW_DATA.MASK_PHONE(phone VARCHAR)
RETURNS VARCHAR
LANGUAGE JAVASCRIPT
AS
$$
    if (!PHONE || PHONE.length < 7) return PHONE;
    return PHONE.slice(0, 3) + '****' + PHONE.slice(-4);
$$;

-- Use it
SELECT ECOMM_DB.RAW_DATA.MASK_PHONE('9876543210') AS masked_phone;
-- Returns: 987****3210
```

### Stored Procedure vs UDF

| | Stored Procedure | UDF |
|---|---|---|
| Called with | `CALL proc()` | Inline in SELECT: `func(col)` |
| Returns | Single value or table | Single value per row |
| Multi-statement | Yes — full procedural logic | No — single expression |
| Use | Automation, pipelines, admin tasks | Per-row transformations, calculations |
| Can run DDL | Yes (`CREATE`, `ALTER`, `DROP`) | No |

---

## Concept 4 — Row Access Policies & Column Masking Policies

### Why Security Policies?

Roles and grants control **which objects** a user can access. Security policies control **which data within those objects** they can see — without duplicating tables or creating multiple filtered views.

```
Role ANALYST can SELECT from ORDERS_RAW
  → Without a policy: sees ALL rows, ALL columns
  → With row access policy: sees only their region's rows
  → With column masking: sees customer_name as '****' instead of real name
```

### Row Access Policy

A Row Access Policy is a function that returns `TRUE` or `FALSE` for each row. Only rows where it returns `TRUE` are visible to a given user.

```sql
USE ROLE SYSADMIN;
USE SCHEMA ECOMM_DB.RAW_DATA;

-- Policy: SYSADMIN sees all rows; ANALYST role sees only CONFIRMED orders
CREATE OR REPLACE ROW ACCESS POLICY ECOMM_DB.RAW_DATA.ORDERS_STATUS_POLICY
    AS (status VARCHAR) RETURNS BOOLEAN ->
        CASE
            WHEN CURRENT_ROLE() = 'SYSADMIN' THEN TRUE
            WHEN CURRENT_ROLE() = 'ANALYST'  THEN status = 'CONFIRMED'
            ELSE FALSE
        END;

-- Attach the policy to ORDERS_RAW on the status column
ALTER TABLE ECOMM_DB.RAW_DATA.ORDERS_RAW
    ADD ROW ACCESS POLICY ECOMM_DB.RAW_DATA.ORDERS_STATUS_POLICY
    ON (status);

-- As SYSADMIN: sees all rows
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_RAW;

-- If you switch to ANALYST role:
-- USE ROLE ANALYST;
-- SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_RAW;
-- Only CONFIRMED rows are visible

-- Remove the policy
ALTER TABLE ECOMM_DB.RAW_DATA.ORDERS_RAW
    DROP ROW ACCESS POLICY ECOMM_DB.RAW_DATA.ORDERS_STATUS_POLICY;
```

### Column Masking Policy

A Column Masking Policy replaces a column's value with a masked version for certain roles — without changing the actual stored data.

```sql
USE SCHEMA ECOMM_DB.RAW_DATA;

-- Policy: SYSADMIN sees real name; everyone else sees '***MASKED***'
CREATE OR REPLACE MASKING POLICY ECOMM_DB.RAW_DATA.MASK_CUSTOMER_NAME
    AS (val VARCHAR) RETURNS VARCHAR ->
        CASE
            WHEN CURRENT_ROLE() = 'SYSADMIN' THEN val
            ELSE '***MASKED***'
        END;

-- Apply it to the customer_name column in ORDERS_RAW
ALTER TABLE ECOMM_DB.RAW_DATA.ORDERS_RAW
    MODIFY COLUMN customer_name
    SET MASKING POLICY ECOMM_DB.RAW_DATA.MASK_CUSTOMER_NAME;

-- As SYSADMIN: sees real names
SELECT order_id, customer_name FROM ECOMM_DB.RAW_DATA.ORDERS_RAW LIMIT 5;

-- A non-SYSADMIN role would see: '***MASKED***' in the customer_name column

-- Remove the masking policy
ALTER TABLE ECOMM_DB.RAW_DATA.ORDERS_RAW
    MODIFY COLUMN customer_name
    UNSET MASKING POLICY;

-- Drop the policy object
DROP MASKING POLICY ECOMM_DB.RAW_DATA.MASK_CUSTOMER_NAME;
```

### Inspect policies

```sql
-- See all row access policies in the account
SHOW ROW ACCESS POLICIES IN SCHEMA ECOMM_DB.RAW_DATA;

-- See all masking policies
SHOW MASKING POLICIES IN SCHEMA ECOMM_DB.RAW_DATA;

-- See which tables a policy is attached to
SELECT *
FROM TABLE(INFORMATION_SCHEMA.POLICY_REFERENCES(
    POLICY_NAME => 'ECOMM_DB.RAW_DATA.MASK_CUSTOMER_NAME'
));
```

---

## Summary

| Concept | Key command | Use |
|---|---|---|
| Dynamic Table | `CREATE DYNAMIC TABLE ... TARGET_LAG = '1 minute' AS SELECT ...` | Auto-refreshing derived table, replaces stream+task for simple pipelines |
| Materialized View | `CREATE MATERIALIZED VIEW ... AS SELECT ...` | Pre-compute heavy read queries; auto-synced |
| Stored Procedure | `CREATE PROCEDURE ... LANGUAGE SQL AS $$ BEGIN ... END; $$` | Multi-step automation, pipeline control |
| UDF | `CREATE FUNCTION ... RETURNS ... AS $$ expr $$` | Per-row inline transformation in SELECT |
| Row Access Policy | `CREATE ROW ACCESS POLICY ... AS (col) RETURNS BOOLEAN ->` | Filter rows by role/attribute |
| Column Masking | `CREATE MASKING POLICY ... AS (val) RETURNS type ->` | Hide or mask sensitive column values |

**Course complete.** Topics for further study: Snowpark (Python/Java/Scala in Snowflake), Data Sharing Marketplace, Replication & Failover, Cortex AI functions, External Functions.
