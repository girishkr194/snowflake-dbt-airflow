# Day 5 — Practice Exercises: Stages, COPY INTO, Snowpipe, and Zero-Copy Cloning

> Run everything in Snowsight in order. No external cloud storage required — uses internal stages only.
> Estimated total time: 60–75 minutes.

---

## Setup — Run This Once

```sql
USE ROLE SYSADMIN;
USE WAREHOUSE COMPUTE_WH;
CREATE DATABASE IF NOT EXISTS ECOMM_DB;
CREATE SCHEMA  IF NOT EXISTS ECOMM_DB.RAW_DATA;
USE SCHEMA ECOMM_DB.RAW_DATA;
```

---

## Exercise 1 — Create a Stage and Inspect It

**What we are testing:** Stage creation, file format definition, LIST command.

```sql
-- Create a named internal stage with CSV format
CREATE OR REPLACE STAGE ECOMM_DB.RAW_DATA.ORDERS_STAGE
    FILE_FORMAT = (
        TYPE                         = 'CSV'
        SKIP_HEADER                  = 1
        FIELD_OPTIONALLY_ENCLOSED_BY = '"'
        NULL_IF                      = ('NULL', 'null', '')
    )
    COMMENT = 'Internal stage for order CSV files';

-- Verify it exists
SHOW STAGES IN SCHEMA ECOMM_DB.RAW_DATA;

-- See the stage definition
DESC STAGE ECOMM_DB.RAW_DATA.ORDERS_STAGE;

-- Check the stage is currently empty
LIST @ECOMM_DB.RAW_DATA.ORDERS_STAGE;
-- Expected: 0 rows
```

**Question to answer:** What is the difference between an internal and external stage?

---

## Exercise 2 — Upload a File and Use COPY INTO

**What we are testing:** File upload to stage, COPY INTO basic load, dedup, FORCE.

### Step 1: Upload the CSV file

Create a local file called `orders_batch1.csv` with the content below, then upload it to `ORDERS_STAGE`.

**File content:**
```
order_id,customer_name,product,quantity,unit_price,total_amount,status,order_date
1001,Alice Johnson,Wireless Headphones,1,149.99,149.99,CONFIRMED,2024-01-10
1002,Bob Smith,Running Shoes,2,89.99,179.98,CONFIRMED,2024-01-10
1003,Priya Sharma,Coffee Maker,1,129.99,129.99,SHIPPED,2024-01-11
1004,Carlos Mendez,Python Book,3,49.99,149.97,PENDING,2024-01-11
1005,Yuki Tanaka,Yoga Mat,2,39.99,79.98,DELIVERED,2024-01-12
```

**How to upload:**
- **Snowsight UI:** Data → Databases → ECOMM_DB → RAW_DATA → Stages → ORDERS_STAGE → Upload button
- **SnowSQL CLI:** `PUT file:///path/to/orders_batch1.csv @ECOMM_DB.RAW_DATA.ORDERS_STAGE;`

Confirm the file is there:
```sql
LIST @ECOMM_DB.RAW_DATA.ORDERS_STAGE;
-- Expected: orders_batch1.csv.gz  (Snowflake auto-compresses on upload)
```

### Step 2: Create the target table

```sql
CREATE OR REPLACE TABLE ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD (
    order_id      NUMBER,
    customer_name VARCHAR(200),
    product       VARCHAR(200),
    quantity      NUMBER,
    unit_price    NUMBER(10,2),
    total_amount  NUMBER(10,2),
    status        VARCHAR(50),
    order_date    DATE
);
```

### Step 3: Load the file with COPY INTO

```sql
COPY INTO ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD
FROM @ECOMM_DB.RAW_DATA.ORDERS_STAGE
FILE_FORMAT = (
    TYPE                         = 'CSV'
    SKIP_HEADER                  = 1
    FIELD_OPTIONALLY_ENCLOSED_BY = '"'
);
```

Output columns to check: `file`, `status` (should be `LOADED`), `rows_loaded` (5), `errors_seen` (0).

### Step 4: Verify

```sql
SELECT * FROM ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD ORDER BY order_id;
-- Expected: 5 rows

SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD;
-- Expected: 5
```

### Step 5: Run COPY INTO again — observe deduplication

```sql
COPY INTO ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD
FROM @ECOMM_DB.RAW_DATA.ORDERS_STAGE
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1);
-- Status: LOAD_SKIPPED — file already loaded

SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD;
-- Still 5 — no duplicates
```

**Question:** Why did COPY INTO skip the file?

### Step 6: Force reload and observe duplicates

```sql
COPY INTO ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD
FROM @ECOMM_DB.RAW_DATA.ORDERS_STAGE
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1)
FORCE = TRUE;

SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD;
-- Expected: 10 — duplicates created!

-- Clean up: remove the duplicates before the next exercise
TRUNCATE TABLE ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD;

-- Reload cleanly
COPY INTO ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD
FROM @ECOMM_DB.RAW_DATA.ORDERS_STAGE
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1)
FORCE = TRUE;

SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD;
-- Back to 5
```

### Step 7: Check load history

```sql
SELECT file_name, row_count, status, last_load_time
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME  => 'ORDERS_RAW_LOAD',
    START_TIME  => DATEADD('minutes', -30, CURRENT_TIMESTAMP())
))
ORDER BY LAST_LOAD_TIME DESC;
```

### Step 8: Load with column selection and transformation

```sql
-- Load only specific columns from the CSV
CREATE OR REPLACE TABLE ECOMM_DB.RAW_DATA.ORDERS_PARTIAL (
    order_id      NUMBER,
    customer_name VARCHAR(200),
    total_amount  NUMBER(10,2),
    status        VARCHAR(50),
    loaded_at     TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

COPY INTO ECOMM_DB.RAW_DATA.ORDERS_PARTIAL (order_id, customer_name, total_amount, status)
FROM (
    SELECT $1, $2, $6, $7   -- $1=order_id, $2=customer_name, $6=total_amount, $7=status
    FROM @ECOMM_DB.RAW_DATA.ORDERS_STAGE
)
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1)
FORCE = TRUE;

SELECT * FROM ECOMM_DB.RAW_DATA.ORDERS_PARTIAL;
-- order_id, customer_name, total_amount, status — no quantity, no unit_price
-- loaded_at filled automatically with DEFAULT
```

---

## Exercise 3 — Upload a Second Batch and Load

**What we are testing:** Loading multiple files, only new files load.

Create a second file `orders_batch2.csv`:
```
order_id,customer_name,product,quantity,unit_price,total_amount,status,order_date
1006,Sophie Dubois,Standing Desk,1,499.99,499.99,CONFIRMED,2024-01-12
1007,Marco Rossi,Mechanical Keyboard,1,189.99,189.99,CONFIRMED,2024-01-13
1008,Aisha Patel,Smart Watch,1,299.99,299.99,PENDING,2024-01-13
```

Upload it to `ORDERS_STAGE`, then:

```sql
-- Both files are in the stage now
LIST @ECOMM_DB.RAW_DATA.ORDERS_STAGE;
-- Should see both orders_batch1.csv.gz and orders_batch2.csv.gz

-- Run COPY INTO — only batch2 loads (batch1 already tracked)
COPY INTO ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD
FROM @ECOMM_DB.RAW_DATA.ORDERS_STAGE
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1);
-- batch1: LOAD_SKIPPED  |  batch2: LOADED (3 rows)

SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD;
-- Expected: 8  (5 from batch1 + 3 from batch2)

-- Load history shows both
SELECT file_name, row_count, status
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME  => 'ORDERS_RAW_LOAD',
    START_TIME  => DATEADD('hours', -1, CURRENT_TIMESTAMP())
))
ORDER BY LAST_LOAD_TIME DESC;
```

---

## Exercise 4 — Snowpipe

**What we are testing:** Pipe creation, REFRESH trigger, deduplication via pipe, pause/resume.

### Step 1: Create the target table for the pipe

```sql
CREATE OR REPLACE TABLE ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD (
    order_id      NUMBER,
    customer_name VARCHAR(200),
    product       VARCHAR(200),
    quantity      NUMBER,
    unit_price    NUMBER(10,2),
    total_amount  NUMBER(10,2),
    status        VARCHAR(50),
    order_date    DATE
);
```

### Step 2: Create the pipe

```sql
CREATE OR REPLACE PIPE ECOMM_DB.RAW_DATA.ORDERS_PIPE
    AUTO_INGEST = FALSE
    COMMENT     = 'Loads CSV files from ORDERS_STAGE into ORDERS_PIPE_LOAD'
AS
COPY INTO ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD
FROM @ECOMM_DB.RAW_DATA.ORDERS_STAGE
FILE_FORMAT = (
    TYPE                         = 'CSV'
    SKIP_HEADER                  = 1
    FIELD_OPTIONALLY_ENCLOSED_BY = '"'
);
```

### Step 3: Verify pipe was created and is running

```sql
SHOW PIPES IN DATABASE ECOMM_DB;

SELECT SYSTEM$PIPE_STATUS('ECOMM_DB.RAW_DATA.ORDERS_PIPE');
-- Look for: "executionState": "RUNNING"
```

### Step 4: Trigger Snowpipe to scan the stage

```sql
-- Both batch1 and batch2 are still in the stage from earlier
-- The PIPE has its own metadata — it has NOT yet loaded them
ALTER PIPE ECOMM_DB.RAW_DATA.ORDERS_PIPE REFRESH;
-- Returns: filesScanned (2), filesQueued (2)

-- Wait 15–20 seconds for serverless compute to process
```

### Step 5: Verify data loaded

```sql
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD;
-- Expected: 8 (5 from batch1 + 3 from batch2)

SELECT * FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD ORDER BY order_id;

SELECT file_name, row_count, status, last_load_time
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME  => 'ORDERS_PIPE_LOAD',
    START_TIME  => DATEADD('minutes', -10, CURRENT_TIMESTAMP())
))
ORDER BY LAST_LOAD_TIME DESC;
```

### Step 6: Trigger REFRESH again — verify deduplication

```sql
ALTER PIPE ECOMM_DB.RAW_DATA.ORDERS_PIPE REFRESH;

-- Wait 20 seconds
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD;
-- Still 8 — Snowpipe tracks loaded files, skips them on re-trigger

SELECT SYSTEM$PIPE_STATUS('ECOMM_DB.RAW_DATA.ORDERS_PIPE');
-- pendingFileCount should be 0
```

### Step 7: Pause the pipe and test

```sql
ALTER PIPE ECOMM_DB.RAW_DATA.ORDERS_PIPE SET PIPE_EXECUTION_PAUSED = TRUE;

SELECT SYSTEM$PIPE_STATUS('ECOMM_DB.RAW_DATA.ORDERS_PIPE');
-- "executionState": "PAUSED"

-- Create and upload orders_batch3.csv:
-- 1009,Lena Müller,Noise Cancelling Mic,1,79.99,79.99,CONFIRMED,2024-01-13

-- Trigger REFRESH — pipe is paused, nothing loads
ALTER PIPE ECOMM_DB.RAW_DATA.ORDERS_PIPE REFRESH;

SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD;
-- Still 8 — paused pipe ignores the refresh
```

### Step 8: Resume and load the missed file

```sql
ALTER PIPE ECOMM_DB.RAW_DATA.ORDERS_PIPE SET PIPE_EXECUTION_PAUSED = FALSE;

-- REFRESH again now that the pipe is running
ALTER PIPE ECOMM_DB.RAW_DATA.ORDERS_PIPE REFRESH;

-- Wait 20 seconds
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD;
-- Expected: 9 (batch3 loaded)
```

---

## Exercise 5 — Zero-Copy Cloning

**What we are testing:** Clone table, clone isolation, schema clone, Time Travel BEFORE recovery.

### Step 1: Clone a single table

```sql
-- Confirm source has data
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD;

-- Clone it instantly
CREATE TABLE ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD_BACKUP
    CLONE ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD;

-- Both have the same count — zero storage cost so far
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD;
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD_BACKUP;
```

### Step 2: Modify source — clone is unaffected

```sql
-- Insert into source
INSERT INTO ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD
VALUES (9999, 'Test User', 'Test Product', 1, 50.00, 50.00, 'PENDING', '2024-01-15');

-- Source: has the new row
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD;

-- Clone: does NOT have it
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD_BACKUP;
-- Proves clone is an independent snapshot
```

### Step 3: Modify clone — source is unaffected

```sql
-- Delete from the CLONE
DELETE FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD_BACKUP
WHERE order_id <= 1003;

-- Clone: fewer rows
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD_BACKUP;

-- Source: completely unchanged
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD;
```

### Step 4: Clone a schema

```sql
-- Clone the entire RAW_DATA schema — all tables cloned at once, instantly
CREATE SCHEMA ECOMM_DB.RAW_DATA_BACKUP
    CLONE ECOMM_DB.RAW_DATA;

-- See all tables that were cloned
SHOW TABLES IN SCHEMA ECOMM_DB.RAW_DATA_BACKUP;
-- You should see ORDERS_PIPE_LOAD, ORDERS_RAW_LOAD, ORDERS_PARTIAL, etc.

SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA_BACKUP.ORDERS_PIPE_LOAD;
-- Same count as the source at clone time
```

### Step 5: Time Travel recovery with BEFORE (STATEMENT)

```sql
USE SCHEMA ECOMM_DB.RAW_DATA;

-- Step 1: Note the current count
SELECT COUNT(*) FROM ORDERS_PIPE_LOAD;

-- Step 2: Accidental delete
DELETE FROM ORDERS_PIPE_LOAD WHERE order_id > 1005;
SELECT COUNT(*) FROM ORDERS_PIPE_LOAD;
-- Rows are gone!

-- Step 3: Find the bad query ID
SELECT query_id, query_text, start_time
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY(
    DATE_RANGE_START => DATEADD('minutes', -5, CURRENT_TIMESTAMP())
))
WHERE query_text ILIKE '%DELETE%ORDERS_PIPE_LOAD%'
ORDER BY start_time DESC
LIMIT 1;
-- Copy the query_id

-- Step 4: Clone table BEFORE the bad delete
CREATE TABLE ECOMM_DB.RAW_DATA.ORDERS_RECOVERED
    CLONE ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD
    BEFORE (STATEMENT => '<paste-query-id-here>');

-- Step 5: Verify recovered data
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_RECOVERED;
-- Should match the count before the delete

-- Step 6: Restore missing rows
INSERT INTO ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD
SELECT * FROM ECOMM_DB.RAW_DATA.ORDERS_RECOVERED r
WHERE NOT EXISTS (
    SELECT 1 FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD s
    WHERE s.order_id = r.order_id
);

-- Step 7: Verify full count is back
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD;

-- Step 8: Drop the recovery table
DROP TABLE ECOMM_DB.RAW_DATA.ORDERS_RECOVERED;
```

---

## Teardown

```sql
USE ROLE SYSADMIN;

-- Drop the pipe
DROP PIPE ECOMM_DB.RAW_DATA.ORDERS_PIPE;

-- Drop backup objects
DROP SCHEMA ECOMM_DB.RAW_DATA_BACKUP;
DROP TABLE  ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD_BACKUP;
DROP TABLE  ECOMM_DB.RAW_DATA.ORDERS_PARTIAL;

-- Confirm
SHOW PIPES  IN DATABASE ECOMM_DB;
SHOW STAGES IN SCHEMA   ECOMM_DB.RAW_DATA;
```

---

## Summary

| Exercise | What you verified |
|---|---|
| 1 | Stage creation, inspection, file listing |
| 2 | COPY INTO basic load, deduplication, FORCE = TRUE, column transformation |
| 3 | Multiple files in stage — only new files loaded |
| 4 | Pipe creation, REFRESH trigger, deduplication via pipe, pause/resume recovery |
| 5 | Clone isolation, schema clone, Time Travel BEFORE(STATEMENT) recovery |

**Next:** Day 6 — Time Travel, Fail-Safe, and Query Optimization.
