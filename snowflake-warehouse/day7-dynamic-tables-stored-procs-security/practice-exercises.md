# Day 7 — Practice Exercises: Dynamic Tables, Materialized Views, Stored Procedures & UDFs, Security Policies

> All exercises use the Day 4 tables: ORDERS_RAW, STG_ORDERS, ORDERS_MART.
> Run everything in Snowsight in order.
> Estimated total time: 60–75 minutes.

---

## Pre-Check

```sql
USE ROLE SYSADMIN;
USE WAREHOUSE COMPUTE_WH;

SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_RAW;
SELECT COUNT(*) FROM ECOMM_DB.STAGING.STG_ORDERS;
SELECT COUNT(*) FROM ECOMM_DB.MARTS.ORDERS_MART;
```

> If tables are empty, run Day 4 Steps 1–6 first.

---

## Exercise 1 — Dynamic Tables

**What we are testing:** Create DT, query it, insert to source, observe auto-refresh, chain DTs, suspend/resume.

### Step 1: Create a Dynamic Table on ORDERS_RAW

```sql
USE ROLE SYSADMIN;
USE SCHEMA ECOMM_DB.STAGING;

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

### Step 2: Inspect and query the Dynamic Table

```sql
-- List dynamic tables
SHOW DYNAMIC TABLES IN SCHEMA ECOMM_DB.STAGING;

-- Query it like a normal table
SELECT COUNT(*) FROM ECOMM_DB.STAGING.STG_ORDERS_DT;
SELECT * FROM ECOMM_DB.STAGING.STG_ORDERS_DT ORDER BY order_id;
-- Should match ORDERS_RAW exactly
```

### Step 3: Insert into source — observe auto-refresh

```sql
USE SCHEMA ECOMM_DB.RAW_DATA;

INSERT INTO ORDERS_RAW
    (order_id, customer_id, customer_name, product, quantity,
     unit_price, total_amount, status, order_date)
VALUES (5001, 51, 'Eva Green', 'Smart Speaker', 1, 79.99, 79.99, 'CONFIRMED', '2024-04-01');

-- Check source immediately
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_RAW;

-- Check DT immediately — may not have refreshed yet
SELECT COUNT(*) FROM ECOMM_DB.STAGING.STG_ORDERS_DT;

-- Wait ~1 minute, then re-check
SELECT COUNT(*) FROM ECOMM_DB.STAGING.STG_ORDERS_DT;
-- Should now include order 5001

SELECT * FROM ECOMM_DB.STAGING.STG_ORDERS_DT WHERE order_id = 5001;
```

### Step 4: Force a manual refresh

```sql
-- Don't want to wait 1 minute? Force it
ALTER DYNAMIC TABLE ECOMM_DB.STAGING.STG_ORDERS_DT REFRESH;

SELECT * FROM ECOMM_DB.STAGING.STG_ORDERS_DT WHERE order_id = 5001;
-- Appears immediately after manual refresh
```

### Step 5: Chain a mart Dynamic Table

```sql
USE SCHEMA ECOMM_DB.MARTS;

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

-- Force refresh and query
ALTER DYNAMIC TABLE ECOMM_DB.MARTS.ORDERS_MART_DT REFRESH;
SELECT * FROM ECOMM_DB.MARTS.ORDERS_MART_DT ORDER BY order_date;
-- Apr 01 should appear as a new row
```

### Step 6: Suspend and resume

```sql
-- Suspend to stop background compute
ALTER DYNAMIC TABLE ECOMM_DB.STAGING.STG_ORDERS_DT SUSPEND;
ALTER DYNAMIC TABLE ECOMM_DB.MARTS.ORDERS_MART_DT  SUSPEND;

SHOW DYNAMIC TABLES IN DATABASE ECOMM_DB;
-- scheduling_state = SUSPENDED for both

-- Resume
ALTER DYNAMIC TABLE ECOMM_DB.STAGING.STG_ORDERS_DT RESUME;
ALTER DYNAMIC TABLE ECOMM_DB.MARTS.ORDERS_MART_DT  RESUME;
```

---

## Exercise 2 — Materialized View

**What we are testing:** Create MV, query it (instant), verify it auto-syncs after source change.

```sql
USE ROLE SYSADMIN;
USE SCHEMA ECOMM_DB.STAGING;

-- Create a Materialized View: pre-compute status summary from STG_ORDERS
CREATE OR REPLACE MATERIALIZED VIEW ECOMM_DB.STAGING.MV_ORDER_STATUS
AS
SELECT
    status,
    COUNT(*)          AS order_count,
    SUM(total_amount) AS total_revenue
FROM ECOMM_DB.STAGING.STG_ORDERS
WHERE is_deleted = FALSE
GROUP BY status;

-- Query the MV — instant, no warehouse compute needed
SELECT * FROM ECOMM_DB.STAGING.MV_ORDER_STATUS ORDER BY order_count DESC;

-- List materialized views
SHOW MATERIALIZED VIEWS IN SCHEMA ECOMM_DB.STAGING;

-- Insert a new row into STG_ORDERS and see the MV auto-update
INSERT INTO ECOMM_DB.STAGING.STG_ORDERS
    (order_id, customer_id, customer_name, product, quantity,
     unit_price, total_amount, status, order_date, is_deleted)
VALUES (5002, 52, 'Leo Nash', 'Keyboard', 1, 49.99, 49.99, 'CONFIRMED', '2024-04-02', FALSE);

-- Query MV again — CONFIRMED count should increase by 1
SELECT * FROM ECOMM_DB.STAGING.MV_ORDER_STATUS ORDER BY order_count DESC;

-- Clean up
DROP MATERIALIZED VIEW ECOMM_DB.STAGING.MV_ORDER_STATUS;

-- Remove the test row
DELETE FROM ECOMM_DB.STAGING.STG_ORDERS WHERE order_id = 5002;
```

---

## Exercise 3 — Stored Procedures

**What we are testing:** Create and call procedures, parameters, variables, DDL inside a procedure.

### Step 1: Simple procedure — suspend the pipeline

```sql
USE SCHEMA ECOMM_DB.RAW_DATA;

CREATE OR REPLACE PROCEDURE ECOMM_DB.RAW_DATA.SUSPEND_PIPELINE()
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
BEGIN
    ALTER TASK ECOMM_DB.RAW_DATA.STG_TO_MART_TASK SUSPEND;
    ALTER TASK ECOMM_DB.RAW_DATA.RAW_TO_STG_TASK  SUSPEND;
    RETURN 'Pipeline suspended successfully';
END;
$$;

CALL ECOMM_DB.RAW_DATA.SUSPEND_PIPELINE();

-- Confirm both tasks are suspended
SHOW TASKS IN DATABASE ECOMM_DB;
```

### Step 2: Procedure to resume pipeline

```sql
CREATE OR REPLACE PROCEDURE ECOMM_DB.RAW_DATA.RESUME_PIPELINE()
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
BEGIN
    -- Child task first, root task last
    ALTER TASK ECOMM_DB.RAW_DATA.STG_TO_MART_TASK RESUME;
    ALTER TASK ECOMM_DB.RAW_DATA.RAW_TO_STG_TASK  RESUME;
    RETURN 'Pipeline resumed successfully';
END;
$$;

CALL ECOMM_DB.RAW_DATA.RESUME_PIPELINE();

SHOW TASKS IN DATABASE ECOMM_DB;
-- Both should show state = started
```

### Step 3: Procedure with parameters and return value

```sql
-- Procedure: count orders for a given status and date range
CREATE OR REPLACE PROCEDURE ECOMM_DB.RAW_DATA.COUNT_ORDERS(
    p_status    VARCHAR,
    p_from_date DATE,
    p_to_date   DATE
)
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
DECLARE
    order_count INTEGER;
BEGIN
    SELECT COUNT(*)
    INTO   :order_count
    FROM   ECOMM_DB.RAW_DATA.ORDERS_RAW
    WHERE  status     = :p_status
      AND  order_date BETWEEN :p_from_date AND :p_to_date;

    RETURN p_status || ' orders between '
        || p_from_date || ' and ' || p_to_date
        || ': ' || order_count;
END;
$$;

-- Call it with different parameters
CALL ECOMM_DB.RAW_DATA.COUNT_ORDERS('CONFIRMED', '2024-01-01', '2024-12-31');
CALL ECOMM_DB.RAW_DATA.COUNT_ORDERS('SHIPPED',   '2024-01-01', '2024-12-31');
```

### Step 4: Procedure to rebuild the mart

```sql
CREATE OR REPLACE PROCEDURE ECOMM_DB.RAW_DATA.REFRESH_MART()
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
DECLARE
    row_count INTEGER;
BEGIN
    -- Suspend tasks
    ALTER TASK ECOMM_DB.RAW_DATA.STG_TO_MART_TASK SUSPEND;
    ALTER TASK ECOMM_DB.RAW_DATA.RAW_TO_STG_TASK  SUSPEND;

    -- Rebuild mart
    TRUNCATE TABLE ECOMM_DB.MARTS.ORDERS_MART;

    INSERT INTO ECOMM_DB.MARTS.ORDERS_MART
        (order_date, total_orders, confirmed_orders, shipped_orders,
         delivered_orders, cancelled_orders, total_revenue, avg_order_value)
    SELECT
        order_date,
        COUNT(*)                                                AS total_orders,
        SUM(CASE WHEN status = 'CONFIRMED'  THEN 1 ELSE 0 END) AS confirmed_orders,
        SUM(CASE WHEN status = 'SHIPPED'    THEN 1 ELSE 0 END) AS shipped_orders,
        SUM(CASE WHEN status = 'DELIVERED'  THEN 1 ELSE 0 END) AS delivered_orders,
        SUM(CASE WHEN status = 'CANCELLED'  THEN 1 ELSE 0 END) AS cancelled_orders,
        SUM(total_amount)                                       AS total_revenue,
        AVG(total_amount)                                       AS avg_order_value
    FROM ECOMM_DB.STAGING.STG_ORDERS
    WHERE is_deleted = FALSE
    GROUP BY order_date;

    SELECT COUNT(*) INTO :row_count FROM ECOMM_DB.MARTS.ORDERS_MART;

    -- Resume tasks (child first, root last)
    ALTER TASK ECOMM_DB.RAW_DATA.STG_TO_MART_TASK RESUME;
    ALTER TASK ECOMM_DB.RAW_DATA.RAW_TO_STG_TASK  RESUME;

    RETURN 'Mart rebuilt. Rows inserted: ' || row_count;
END;
$$;

CALL ECOMM_DB.RAW_DATA.REFRESH_MART();

-- Verify the mart
SELECT * FROM ECOMM_DB.MARTS.ORDERS_MART ORDER BY order_date;
```

---

## Exercise 4 — User-Defined Functions (UDFs)

**What we are testing:** SQL UDF, JavaScript UDF, inline use in SELECT and aggregations.

### Step 1: SQL UDF — classify order size

```sql
USE SCHEMA ECOMM_DB.RAW_DATA;

CREATE OR REPLACE FUNCTION ECOMM_DB.RAW_DATA.REVENUE_TIER(total NUMBER)
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
    CASE
        WHEN total < 100  THEN 'LOW'
        WHEN total < 500  THEN 'MID'
        ELSE                   'HIGH'
    END
$$;

-- Use in SELECT
SELECT
    order_id,
    customer_name,
    total_amount,
    ECOMM_DB.RAW_DATA.REVENUE_TIER(total_amount) AS tier
FROM ECOMM_DB.RAW_DATA.ORDERS_RAW
ORDER BY total_amount DESC;

-- Group by UDF result
SELECT
    ECOMM_DB.RAW_DATA.REVENUE_TIER(total_amount) AS tier,
    COUNT(*)          AS order_count,
    SUM(total_amount) AS total_revenue
FROM ECOMM_DB.RAW_DATA.ORDERS_RAW
GROUP BY tier
ORDER BY total_revenue DESC;
```

### Step 2: SQL UDF — format order date

```sql
CREATE OR REPLACE FUNCTION ECOMM_DB.RAW_DATA.FRIENDLY_DATE(d DATE)
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
    TO_VARCHAR(d, 'DD Mon YYYY')
$$;

SELECT
    order_id,
    order_date,
    ECOMM_DB.RAW_DATA.FRIENDLY_DATE(order_date) AS display_date
FROM ECOMM_DB.RAW_DATA.ORDERS_RAW
ORDER BY order_date
LIMIT 5;
-- Shows dates like: 10 Jan 2024
```

### Step 3: JavaScript UDF — mask a name

```sql
CREATE OR REPLACE FUNCTION ECOMM_DB.RAW_DATA.MASK_NAME(name VARCHAR)
RETURNS VARCHAR
LANGUAGE JAVASCRIPT
AS
$$
    if (!NAME) return NAME;
    var parts = NAME.split(' ');
    if (parts.length === 1) return NAME[0] + '***';
    return NAME[0] + '*** ' + parts[parts.length - 1];
$$;

-- Use it
SELECT
    order_id,
    customer_name,
    ECOMM_DB.RAW_DATA.MASK_NAME(customer_name) AS masked_name
FROM ECOMM_DB.RAW_DATA.ORDERS_RAW
ORDER BY order_id
LIMIT 5;
-- Alice Johnson → A*** Johnson
-- Bob Smith     → B*** Smith

-- List all UDFs in this schema
SHOW FUNCTIONS IN SCHEMA ECOMM_DB.RAW_DATA;
```

---

## Exercise 5 — Row Access Policy & Column Masking

**What we are testing:** Create policies, attach, test, remove.

### Step 1: Row Access Policy — filter by status

```sql
USE ROLE SYSADMIN;
USE SCHEMA ECOMM_DB.RAW_DATA;

-- Policy: SYSADMIN sees all; anyone else only sees CONFIRMED orders
CREATE OR REPLACE ROW ACCESS POLICY ECOMM_DB.RAW_DATA.CONFIRMED_ONLY_POLICY
    AS (status VARCHAR) RETURNS BOOLEAN ->
        CASE
            WHEN CURRENT_ROLE() = 'SYSADMIN' THEN TRUE
            ELSE status = 'CONFIRMED'
        END;

-- Attach to ORDERS_RAW
ALTER TABLE ECOMM_DB.RAW_DATA.ORDERS_RAW
    ADD ROW ACCESS POLICY ECOMM_DB.RAW_DATA.CONFIRMED_ONLY_POLICY
    ON (status);

-- As SYSADMIN — sees all statuses
SELECT status, COUNT(*) FROM ORDERS_RAW GROUP BY status ORDER BY status;

-- Verify policy is attached
SHOW ROW ACCESS POLICIES IN SCHEMA ECOMM_DB.RAW_DATA;

-- Remove the policy before moving on
ALTER TABLE ECOMM_DB.RAW_DATA.ORDERS_RAW
    DROP ROW ACCESS POLICY ECOMM_DB.RAW_DATA.CONFIRMED_ONLY_POLICY;

-- Query again — should see all statuses
SELECT status, COUNT(*) FROM ORDERS_RAW GROUP BY status ORDER BY status;
```

### Step 2: Column Masking Policy — hide customer names

```sql
USE SCHEMA ECOMM_DB.RAW_DATA;

-- Policy: SYSADMIN sees real name; anyone else sees '***MASKED***'
CREATE OR REPLACE MASKING POLICY ECOMM_DB.RAW_DATA.MASK_CUSTOMER_NAME
    AS (val VARCHAR) RETURNS VARCHAR ->
        CASE
            WHEN CURRENT_ROLE() = 'SYSADMIN' THEN val
            ELSE '***MASKED***'
        END;

-- Apply to customer_name in ORDERS_RAW
ALTER TABLE ECOMM_DB.RAW_DATA.ORDERS_RAW
    MODIFY COLUMN customer_name
    SET MASKING POLICY ECOMM_DB.RAW_DATA.MASK_CUSTOMER_NAME;

-- As SYSADMIN: real names visible
SELECT order_id, customer_name, status FROM ORDERS_RAW ORDER BY order_id LIMIT 5;

-- Inspect which columns the policy is applied to
SELECT *
FROM TABLE(INFORMATION_SCHEMA.POLICY_REFERENCES(
    POLICY_NAME => 'ECOMM_DB.RAW_DATA.MASK_CUSTOMER_NAME'
));

-- Remove the masking policy from the column
ALTER TABLE ECOMM_DB.RAW_DATA.ORDERS_RAW
    MODIFY COLUMN customer_name
    UNSET MASKING POLICY;

-- Drop the policy object
DROP MASKING POLICY ECOMM_DB.RAW_DATA.MASK_CUSTOMER_NAME;

-- Query to confirm names are visible again
SELECT order_id, customer_name FROM ORDERS_RAW ORDER BY order_id LIMIT 5;
```

---

## Final Consistency Check

```sql
-- All three layers still in sync after all exercises
SELECT
    (SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_RAW)                        AS raw_count,
    (SELECT COUNT(*) FROM ECOMM_DB.STAGING.STG_ORDERS WHERE is_deleted = FALSE) AS stg_active,
    (SELECT COUNT(*) FROM ECOMM_DB.STAGING.STG_ORDERS_DT)                       AS dt_count,
    (SELECT SUM(total_amount)  FROM ECOMM_DB.RAW_DATA.ORDERS_RAW)               AS raw_revenue,
    (SELECT SUM(total_revenue) FROM ECOMM_DB.MARTS.ORDERS_MART)                 AS mart_revenue;
-- raw_count = stg_active = dt_count, raw_revenue = mart_revenue
```

---

## Teardown

```sql
USE ROLE SYSADMIN;

-- Suspend tasks
ALTER TASK ECOMM_DB.RAW_DATA.RAW_TO_STG_TASK  SUSPEND;
ALTER TASK ECOMM_DB.RAW_DATA.STG_TO_MART_TASK SUSPEND;

-- Drop Dynamic Tables
DROP DYNAMIC TABLE ECOMM_DB.STAGING.STG_ORDERS_DT;
DROP DYNAMIC TABLE ECOMM_DB.MARTS.ORDERS_MART_DT;

-- Drop Procedures and UDFs created today
DROP PROCEDURE ECOMM_DB.RAW_DATA.SUSPEND_PIPELINE();
DROP PROCEDURE ECOMM_DB.RAW_DATA.RESUME_PIPELINE();
DROP PROCEDURE ECOMM_DB.RAW_DATA.COUNT_ORDERS(VARCHAR, DATE, DATE);
DROP PROCEDURE ECOMM_DB.RAW_DATA.REFRESH_MART();
DROP FUNCTION  ECOMM_DB.RAW_DATA.REVENUE_TIER(NUMBER);
DROP FUNCTION  ECOMM_DB.RAW_DATA.FRIENDLY_DATE(DATE);
DROP FUNCTION  ECOMM_DB.RAW_DATA.MASK_NAME(VARCHAR);

-- Clean up test row inserted in Exercise 1
DELETE FROM ECOMM_DB.RAW_DATA.ORDERS_RAW WHERE order_id = 5001;
EXECUTE TASK ECOMM_DB.RAW_DATA.RAW_TO_STG_TASK;
```

---

## Summary

| Exercise | What you verified |
|---|---|
| 1 | Dynamic Table creation, manual REFRESH, auto-refresh after INSERT, chained DT, suspend/resume |
| 2 | Materialized View creation, instant query, auto-sync after source change |
| 3 | Stored procedures with/without params, variables, DDL inside proc, CALL |
| 4 | SQL UDF, JavaScript UDF, inline use in SELECT and GROUP BY |
| 5 | Row access policy (attach/detach), column masking (apply/unset/drop) |

**Course complete.** Topics for further study: Snowpark (Python/Java/Scala), Data Replication & Failover, Cortex AI functions, External Functions, Snowflake Marketplace deep dive.
