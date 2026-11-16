# Day 4 — Streams and Tasks: Hands-On Pipeline

## What You Will Build

A fully automated RAW → STAGING → MART pipeline for an e-commerce orders dataset.

You will insert, update, and delete data in the RAW layer, watch Streams capture every change, and see Tasks automatically propagate those changes through Staging into the Mart — without you running anything manually after the initial setup.

**The pipeline:**
```
ORDERS_RAW  ──[stream]──►  STG_ORDERS  ──[stream]──►  ORDERS_MART
     │                          │
  INSERT                    MERGE (upsert)
  UPDATE                        │
  DELETE                    MERGE (upsert)
     │
  Task 1 (every 1 min)       Task 2 (after Task 1)
```

---

## Step 1 — Set Up the Schema and Tables

### Database and schema

```sql
USE ROLE SYSADMIN;
USE WAREHOUSE COMPUTE_WH;

-- Use the same DB from Day 2/3 or create fresh
CREATE DATABASE IF NOT EXISTS ECOMM_DB;

CREATE SCHEMA IF NOT EXISTS ECOMM_DB.RAW_DATA;
CREATE SCHEMA IF NOT EXISTS ECOMM_DB.STAGING;
CREATE SCHEMA IF NOT EXISTS ECOMM_DB.MARTS;
```

### RAW table — the source of truth for incoming data

```sql
USE SCHEMA ECOMM_DB.RAW_DATA;

CREATE TABLE IF NOT EXISTS ORDERS_RAW (
    order_id       NUMBER         PRIMARY KEY,
    customer_id    NUMBER         NOT NULL,
    customer_name  VARCHAR(200),
    product        VARCHAR(200),
    quantity       NUMBER         DEFAULT 1,
    unit_price     NUMBER(10,2),
    total_amount   NUMBER(12,2),
    status         VARCHAR(50),   -- PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
    order_date     DATE,
    updated_at     TIMESTAMP_NTZ  DEFAULT CURRENT_TIMESTAMP()
);
```

### STAGING table — cleaned, deduplicated, always has latest state

```sql
USE SCHEMA ECOMM_DB.STAGING;

CREATE TABLE IF NOT EXISTS STG_ORDERS (
    order_id       NUMBER         PRIMARY KEY,
    customer_id    NUMBER,
    customer_name  VARCHAR(200),
    product        VARCHAR(200),
    quantity       NUMBER,
    unit_price     NUMBER(10,2),
    total_amount   NUMBER(12,2),
    status         VARCHAR(50),
    order_date     DATE,
    is_deleted     BOOLEAN        DEFAULT FALSE,   -- soft delete flag
    stg_updated_at TIMESTAMP_NTZ  DEFAULT CURRENT_TIMESTAMP()
);
```

### MART table — aggregated, business-ready

```sql
USE SCHEMA ECOMM_DB.MARTS;

CREATE TABLE IF NOT EXISTS ORDERS_MART (
    order_date        DATE           PRIMARY KEY,
    total_orders      NUMBER,
    confirmed_orders  NUMBER,
    shipped_orders    NUMBER,
    delivered_orders  NUMBER,
    cancelled_orders  NUMBER,
    total_revenue     NUMBER(14,2),
    avg_order_value   NUMBER(10,2),
    mart_updated_at   TIMESTAMP_NTZ  DEFAULT CURRENT_TIMESTAMP()
);
```

---

## Step 2 — Create Streams

One stream on each layer to capture changes as they flow through.

```sql
-- Stream on RAW: captures everything that changes in ORDERS_RAW
CREATE STREAM IF NOT EXISTS ECOMM_DB.RAW_DATA.ORDERS_RAW_STREAM
  ON TABLE ECOMM_DB.RAW_DATA.ORDERS_RAW
  COMMENT = 'Captures INSERT / UPDATE / DELETE on ORDERS_RAW';

-- Stream on STAGING: captures changes after the MERGE writes to STG_ORDERS
CREATE STREAM IF NOT EXISTS ECOMM_DB.STAGING.STG_ORDERS_STREAM
  ON TABLE ECOMM_DB.STAGING.STG_ORDERS
  COMMENT = 'Captures changes written into STG_ORDERS by Task 1';
```

Verify both streams:
```sql
SHOW STREAMS IN DATABASE ECOMM_DB;
```

---

## Step 3 — The MERGE Logic

### MERGE 1: RAW → STAGING

Handles all three change types from the stream:

```sql
MERGE INTO ECOMM_DB.STAGING.STG_ORDERS AS tgt
USING (
    SELECT *
    FROM   ECOMM_DB.RAW_DATA.ORDERS_RAW_STREAM
) AS src
ON tgt.order_id = src.order_id

-- Row exists in staging AND this is a live row (INSERT action = new value from an update or a fresh insert)
WHEN MATCHED AND src.METADATA$ACTION = 'INSERT' THEN
    UPDATE SET
        tgt.customer_id    = src.customer_id,
        tgt.customer_name  = src.customer_name,
        tgt.product        = src.product,
        tgt.quantity       = src.quantity,
        tgt.unit_price     = src.unit_price,
        tgt.total_amount   = src.total_amount,
        tgt.status         = src.status,
        tgt.order_date     = src.order_date,
        tgt.is_deleted     = FALSE,
        tgt.stg_updated_at = CURRENT_TIMESTAMP()

-- Row exists in staging AND this is a DELETE (hard delete from RAW)
WHEN MATCHED AND src.METADATA$ACTION = 'DELETE' AND src.METADATA$ISUPDATE = FALSE THEN
    UPDATE SET
        tgt.is_deleted     = TRUE,
        tgt.stg_updated_at = CURRENT_TIMESTAMP()

-- Row does not exist in staging — brand new order
WHEN NOT MATCHED AND src.METADATA$ACTION = 'INSERT' THEN
    INSERT (order_id, customer_id, customer_name, product, quantity,
            unit_price, total_amount, status, order_date, is_deleted, stg_updated_at)
    VALUES (src.order_id, src.customer_id, src.customer_name, src.product, src.quantity,
            src.unit_price, src.total_amount, src.status, src.order_date, FALSE, CURRENT_TIMESTAMP());
```

**Key design decisions:**
- Deletes in RAW become **soft deletes** in staging (`is_deleted = TRUE`). This preserves history — you never lose the fact that an order existed.
- The `DELETE` rows from an UPDATE (`METADATA$ISUPDATE = TRUE`) are ignored — the matching `INSERT` row carries the new values and handles it via the `WHEN MATCHED AND METADATA$ACTION = 'INSERT'` branch.

### MERGE 2: STAGING → MART

Rebuilds the daily summary from staging, respecting soft deletes:

```sql
MERGE INTO ECOMM_DB.MARTS.ORDERS_MART AS tgt
USING (
    SELECT
        order_date,
        COUNT(*)                                                   AS total_orders,
        SUM(CASE WHEN status = 'CONFIRMED'  THEN 1 ELSE 0 END)    AS confirmed_orders,
        SUM(CASE WHEN status = 'SHIPPED'    THEN 1 ELSE 0 END)    AS shipped_orders,
        SUM(CASE WHEN status = 'DELIVERED'  THEN 1 ELSE 0 END)    AS delivered_orders,
        SUM(CASE WHEN status = 'CANCELLED'  THEN 1 ELSE 0 END)    AS cancelled_orders,
        SUM(total_amount)                                          AS total_revenue,
        AVG(total_amount)                                          AS avg_order_value
    FROM ECOMM_DB.STAGING.STG_ORDERS
    WHERE is_deleted = FALSE                -- exclude soft-deleted rows
    GROUP BY order_date
) AS src
ON tgt.order_date = src.order_date
WHEN MATCHED THEN
    UPDATE SET
        tgt.total_orders     = src.total_orders,
        tgt.confirmed_orders = src.confirmed_orders,
        tgt.shipped_orders   = src.shipped_orders,
        tgt.delivered_orders = src.delivered_orders,
        tgt.cancelled_orders = src.cancelled_orders,
        tgt.total_revenue    = src.total_revenue,
        tgt.avg_order_value  = src.avg_order_value,
        tgt.mart_updated_at  = CURRENT_TIMESTAMP()
WHEN NOT MATCHED THEN
    INSERT (order_date, total_orders, confirmed_orders, shipped_orders,
            delivered_orders, cancelled_orders, total_revenue, avg_order_value, mart_updated_at)
    VALUES (src.order_date, src.total_orders, src.confirmed_orders, src.shipped_orders,
            src.delivered_orders, src.cancelled_orders, src.total_revenue, src.avg_order_value,
            CURRENT_TIMESTAMP());
```

---

## Step 4 — Create the Task Chain

```sql
-- Task 1 (ROOT): RAW → STAGING
-- Runs every minute, only if the raw stream has changes
CREATE TASK IF NOT EXISTS ECOMM_DB.RAW_DATA.RAW_TO_STG_TASK
    WAREHOUSE = COMPUTE_WH
    SCHEDULE  = '1 MINUTE'
    WHEN SYSTEM$STREAM_HAS_DATA('ECOMM_DB.RAW_DATA.ORDERS_RAW_STREAM')
AS
MERGE INTO ECOMM_DB.STAGING.STG_ORDERS AS tgt
USING (SELECT * FROM ECOMM_DB.RAW_DATA.ORDERS_RAW_STREAM) AS src
ON tgt.order_id = src.order_id
WHEN MATCHED AND src.METADATA$ACTION = 'INSERT' THEN
    UPDATE SET
        tgt.customer_id = src.customer_id, tgt.customer_name = src.customer_name,
        tgt.product = src.product, tgt.quantity = src.quantity,
        tgt.unit_price = src.unit_price, tgt.total_amount = src.total_amount,
        tgt.status = src.status, tgt.order_date = src.order_date,
        tgt.is_deleted = FALSE, tgt.stg_updated_at = CURRENT_TIMESTAMP()
WHEN MATCHED AND src.METADATA$ACTION = 'DELETE' AND src.METADATA$ISUPDATE = FALSE THEN
    UPDATE SET tgt.is_deleted = TRUE, tgt.stg_updated_at = CURRENT_TIMESTAMP()
WHEN NOT MATCHED AND src.METADATA$ACTION = 'INSERT' THEN
    INSERT (order_id, customer_id, customer_name, product, quantity,
            unit_price, total_amount, status, order_date, is_deleted, stg_updated_at)
    VALUES (src.order_id, src.customer_id, src.customer_name, src.product, src.quantity,
            src.unit_price, src.total_amount, src.status, src.order_date, FALSE, CURRENT_TIMESTAMP());


-- Task 2 (CHILD): STAGING → MART
-- Runs after Task 1 completes — no schedule, no WHEN needed
CREATE TASK IF NOT EXISTS ECOMM_DB.STAGING.STG_TO_MART_TASK
    WAREHOUSE = COMPUTE_WH
    AFTER     ECOMM_DB.RAW_DATA.RAW_TO_STG_TASK
AS
MERGE INTO ECOMM_DB.MARTS.ORDERS_MART AS tgt
USING (
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
    GROUP BY order_date
) AS src
ON tgt.order_date = src.order_date
WHEN MATCHED THEN
    UPDATE SET
        tgt.total_orders = src.total_orders, tgt.confirmed_orders = src.confirmed_orders,
        tgt.shipped_orders = src.shipped_orders, tgt.delivered_orders = src.delivered_orders,
        tgt.cancelled_orders = src.cancelled_orders, tgt.total_revenue = src.total_revenue,
        tgt.avg_order_value = src.avg_order_value, tgt.mart_updated_at = CURRENT_TIMESTAMP()
WHEN NOT MATCHED THEN
    INSERT (order_date, total_orders, confirmed_orders, shipped_orders,
            delivered_orders, cancelled_orders, total_revenue, avg_order_value, mart_updated_at)
    VALUES (src.order_date, src.total_orders, src.confirmed_orders, src.shipped_orders,
            src.delivered_orders, src.cancelled_orders, src.total_revenue, src.avg_order_value,
            CURRENT_TIMESTAMP());


-- Resume: child first, root last
ALTER TASK ECOMM_DB.STAGING.STG_TO_MART_TASK RESUME;
ALTER TASK ECOMM_DB.RAW_DATA.RAW_TO_STG_TASK RESUME;
```

---

## Step 5 — Verification Queries

Use these after every test to confirm all three layers are consistent.

```sql
-- RAW layer
SELECT order_id, customer_name, product, status, total_amount, updated_at
FROM ECOMM_DB.RAW_DATA.ORDERS_RAW
ORDER BY order_id;

-- STAGING layer (soft deletes visible)
SELECT order_id, customer_name, product, status, total_amount, is_deleted, stg_updated_at
FROM ECOMM_DB.STAGING.STG_ORDERS
ORDER BY order_id;

-- MART layer (only active, aggregated by date)
SELECT * FROM ECOMM_DB.MARTS.ORDERS_MART ORDER BY order_date;

-- Stream status (should be 0 after task runs)
SELECT SYSTEM$STREAM_HAS_DATA('ECOMM_DB.RAW_DATA.ORDERS_RAW_STREAM') AS raw_stream_has_data;

-- Task run history
SELECT name, state, scheduled_time, completed_time, error_message
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    SCHEDULED_TIME_RANGE_START => DATEADD('minutes', -10, CURRENT_TIMESTAMP())
))
ORDER BY scheduled_time DESC;
```

---

## Summary

| Layer | Table | How it gets data |
|---|---|---|
| RAW | `ORDERS_RAW` | Direct INSERT / UPDATE / DELETE |
| STAGING | `STG_ORDERS` | MERGE from stream on RAW (Task 1) |
| MART | `ORDERS_MART` | MERGE from full STG_ORDERS scan (Task 2) |

**Next:** Day 5 covers semi-structured data — VARIANT, loading JSON, colon syntax, and LATERAL FLATTEN.
