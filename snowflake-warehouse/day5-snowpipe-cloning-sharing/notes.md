# Day 5 — Stages, COPY INTO, Snowpipe, and Zero-Copy Cloning

## What You Will Learn

1. **Stages** — Where Snowflake stores files before loading. Internal (Snowflake-managed) and External (your cloud storage).
2. **COPY INTO** — Bulk-load files from a stage into a table. Manual, batch, uses your warehouse.
3. **Snowpipe** — Automatic, continuous file ingestion. Serverless, event-driven, no scheduler needed.
4. **Zero-Copy Cloning** — Instant copy of a table/schema/database with zero storage cost.

---

## Concept 1 — Stages

### What is a Stage?

A **Stage** is a Snowflake object that points to a file storage location. Files must land in a stage before Snowflake can load them into a table.

Think of a stage as a **landing zone** — your files arrive here first, then COPY INTO or Snowpipe moves the data into a table.

```
Your files  ──►  Stage (landing zone)  ──►  Snowflake Table
```

Stages do **not** store Snowflake table data. They store raw files — CSV, JSON, Parquet, Avro, etc.

### Two Types of Stage

#### Internal Stage (Snowflake-managed storage)

Snowflake hosts the storage for you. No cloud bucket, no credentials to manage.

Three sub-types:

| Sub-type | Scope | Auto-created? | Use |
|---|---|---|---|
| User stage | Per user (`@~`) | Yes | Quick personal uploads |
| Table stage | Per table (`@%table_name`) | Yes | Load files directly into one table |
| Named stage | You create it | No | Shared, reusable, recommended for pipelines |

**Named stage** is what you will use in almost every real scenario.

#### External Stage (cloud storage you own)

Snowflake reads from your own S3 bucket, Azure Blob / ADLS, or GCS container. You provide the credentials. The files stay in your cloud storage — Snowflake never copies them in.

```sql
-- External stage example (definition only — requires your cloud credentials)
CREATE STAGE my_s3_stage
    URL                 = 's3://my-bucket/orders/'
    STORAGE_INTEGRATION = my_s3_integration;
```

> External stages require a **Storage Integration** (an OAuth trust between Snowflake and your cloud) — covered when you set up cloud-specific pipelines. For today's hands-on, we use **internal named stages** — no cloud credentials needed.

---

### Hands-On: Internal Named Stage

#### Step 1: Create a named internal stage

```sql
USE ROLE SYSADMIN;
USE WAREHOUSE COMPUTE_WH;
USE DATABASE ECOMM_DB;
USE SCHEMA RAW_DATA;

-- Create a named internal stage with CSV file format
CREATE OR REPLACE STAGE ORDERS_STAGE
    FILE_FORMAT = (
        TYPE                         = 'CSV'
        SKIP_HEADER                  = 1
        FIELD_OPTIONALLY_ENCLOSED_BY = '"'
        NULL_IF                      = ('NULL', 'null', '')
    )
    COMMENT = 'Internal stage for order CSV files';
```

#### Step 2: Inspect the stage

```sql
-- List all stages in this schema
SHOW STAGES IN SCHEMA ECOMM_DB.RAW_DATA;

-- Check the stage definition
DESC STAGE ECOMM_DB.RAW_DATA.ORDERS_STAGE;
```

#### Step 3: Upload a file to the stage

> In Snowsight: go to **Data → Databases → ECOMM_DB → RAW_DATA → Stages → ORDERS_STAGE** and use the upload button.
>
> With SnowSQL CLI: `PUT file:///path/to/orders.csv @ECOMM_DB.RAW_DATA.ORDERS_STAGE;`

Create a local file called `orders_batch1.csv` with this content (copy-paste into a text file):

```
order_id,customer_name,product,quantity,unit_price,total_amount,status,order_date
1001,Alice Johnson,Wireless Headphones,1,149.99,149.99,CONFIRMED,2024-01-10
1002,Bob Smith,Running Shoes,2,89.99,179.98,CONFIRMED,2024-01-10
1003,Priya Sharma,Coffee Maker,1,129.99,129.99,SHIPPED,2024-01-11
1004,Carlos Mendez,Python Book,3,49.99,149.97,PENDING,2024-01-11
1005,Yuki Tanaka,Yoga Mat,2,39.99,79.98,DELIVERED,2024-01-12
```

#### Step 4: Verify the file is in the stage

```sql
-- List files currently in the stage
LIST @ECOMM_DB.RAW_DATA.ORDERS_STAGE;
-- You should see: orders_batch1.csv.gz  (Snowflake auto-compresses on upload)
```

---

## Concept 2 — COPY INTO

### What is COPY INTO?

`COPY INTO` loads files from a stage into a Snowflake table. It is a **manual, batch command** — you run it when you want to load. It uses your virtual warehouse.

```
Stage  ──►  COPY INTO  ──►  Table
```

### Key behaviors

- **Deduplication built-in:** Tracks which files it has already loaded (64-day metadata window). Re-running COPY INTO on the same file does NOT create duplicate rows.
- **Atomic per file:** Each file load is all-or-nothing. If a file fails halfway, no partial rows are inserted.
- **Transformation at load time:** You can cast, rename, or reorder columns inline using a SELECT in the COPY INTO.

### Hands-On: COPY INTO

#### Step 1: Create the target table

```sql
USE SCHEMA ECOMM_DB.RAW_DATA;

CREATE OR REPLACE TABLE ORDERS_RAW_LOAD (
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

#### Step 2: Load the file from the stage

```sql
COPY INTO ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD
FROM @ECOMM_DB.RAW_DATA.ORDERS_STAGE
FILE_FORMAT = (
    TYPE                         = 'CSV'
    SKIP_HEADER                  = 1
    FIELD_OPTIONALLY_ENCLOSED_BY = '"'
);
```

Output shows: file name, status (`LOADED`), rows loaded, rows with errors.

#### Step 3: Verify

```sql
SELECT * FROM ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD ORDER BY order_id;
-- Expected: 5 rows

SELECT COUNT(*) AS row_count FROM ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD;
```

#### Step 4: Try loading the same file again — observe deduplication

```sql
COPY INTO ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD
FROM @ECOMM_DB.RAW_DATA.ORDERS_STAGE
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1);

-- Status: LOAD_SKIPPED — file already loaded, dedup protection kicked in
-- Row count: still 5

SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD;
```

#### Step 5: Force reload (bypass dedup — useful for testing)

```sql
COPY INTO ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD
FROM @ECOMM_DB.RAW_DATA.ORDERS_STAGE
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1)
FORCE = TRUE;  -- ignores the dedup metadata, loads again

SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD;
-- Now 10 rows (5 original + 5 duplicated) — this is why FORCE = TRUE is for testing only
```

#### Step 6: Load specific files only

```sql
-- Load only one specific file from the stage
COPY INTO ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD
FROM @ECOMM_DB.RAW_DATA.ORDERS_STAGE
FILES = ('orders_batch1.csv.gz')
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1);
```

#### Step 7: Check load history

```sql
SELECT *
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME  => 'ORDERS_RAW_LOAD',
    START_TIME  => DATEADD('hours', -1, CURRENT_TIMESTAMP())
))
ORDER BY LAST_LOAD_TIME DESC;
```

#### Step 8: Load with inline transformation

```sql
-- Load only specific columns and add a loaded_at timestamp
CREATE OR REPLACE TABLE ORDERS_RAW_LOAD (
    order_id      NUMBER,
    customer_name VARCHAR(200),
    product       VARCHAR(200),
    total_amount  NUMBER(10,2),
    status        VARCHAR(50),
    loaded_at     TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

COPY INTO ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD (order_id, customer_name, product, total_amount, status)
FROM (
    SELECT $1, $2, $3, $6, $7   -- column positions from the CSV ($1 = first column)
    FROM @ECOMM_DB.RAW_DATA.ORDERS_STAGE
)
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1)
FORCE = TRUE;

SELECT * FROM ECOMM_DB.RAW_DATA.ORDERS_RAW_LOAD;
```

---

## Concept 3 — Snowpipe

### What is Snowpipe?

Snowpipe is Snowflake's **automatic, continuous file ingestion** service. Instead of you running `COPY INTO` manually or on a schedule, Snowpipe monitors a stage and loads files **as they arrive**.

```
File lands in stage  ──►  Snowpipe detects it  ──►  COPY INTO runs automatically
```

You define a **Pipe** object — which stage to watch and what COPY INTO to run. Snowpipe handles the rest.

### Key Properties — Snowpipe vs COPY INTO vs Task+COPY INTO

| Property | COPY INTO (manual) | Task + COPY INTO | Snowpipe |
|---|---|---|---|
| **Trigger** | You run it manually | Fixed schedule (min 1 min) | File arrival or REST API call |
| **Compute** | Your virtual warehouse | Your virtual warehouse | Snowflake-managed (serverless) |
| **Latency** | On demand — whenever you run | Up to 1 minute + startup | Seconds to ~1 minute |
| **Idle cost** | Zero (you only run when needed) | Warehouse may bill while idle | Zero — no files = no compute |
| **Duplicate prevention** | Built-in (64-day window) | You must handle it | Built-in (tracks file metadata) |
| **Cost model** | Warehouse credits | Warehouse credits | Per-second of serverless compute |
| **Best for** | One-time / ad-hoc loads | Scheduled micro-batches | Continuous file streaming |

### Two Trigger Modes

| Mode | How it works | When to use |
|---|---|---|
| **REST API** | Your app calls the Snowpipe REST endpoint to say "new files arrived" | When your app knows about the files |
| **Auto-Ingest** | Cloud storage fires an event when a file lands (S3 SNS / Azure Event Grid / GCS Pub/Sub) → Snowpipe reads it automatically | Zero-code, fully automated pipelines |

---

### Hands-On: Snowpipe (REST / REFRESH mode)

> We use `ALTER PIPE ... REFRESH` — the Snowsight-compatible way to trigger Snowpipe without needing cloud event infrastructure. This is the same as calling the REST API manually.

#### Step 1: Create the target table

```sql
USE SCHEMA ECOMM_DB.RAW_DATA;

CREATE OR REPLACE TABLE ORDERS_PIPE_LOAD (
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

#### Step 2: Create the Pipe

```sql
CREATE OR REPLACE PIPE ECOMM_DB.RAW_DATA.ORDERS_PIPE
    AUTO_INGEST = FALSE   -- we trigger manually; set TRUE for event-driven
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

The `AS` clause is a standard COPY INTO. Snowpipe runs this automatically every time it is triggered.

#### Step 3: Check pipe status

```sql
SHOW PIPES IN DATABASE ECOMM_DB;

-- Detailed status
SELECT SYSTEM$PIPE_STATUS('ECOMM_DB.RAW_DATA.ORDERS_PIPE');
-- Look for: "executionState": "RUNNING"
```

#### Step 4: Confirm file is in the stage

```sql
LIST @ECOMM_DB.RAW_DATA.ORDERS_STAGE;
-- orders_batch1.csv.gz should be there from earlier
```

#### Step 5: Trigger Snowpipe to scan the stage and load new files

```sql
-- Tells Snowpipe: scan the stage and load any files not yet loaded by this pipe
ALTER PIPE ECOMM_DB.RAW_DATA.ORDERS_PIPE REFRESH;
-- Returns: filesScanned, filesQueued

-- Wait 15–20 seconds (serverless compute spins up and loads the file)
```

#### Step 6: Verify data loaded

```sql
SELECT * FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD ORDER BY order_id;
-- Expected: 5 rows

SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD;
```

#### Step 7: Check load history

```sql
SELECT file_name, row_count, status, last_load_time
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME  => 'ORDERS_PIPE_LOAD',
    START_TIME  => DATEADD('minutes', -10, CURRENT_TIMESTAMP())
))
ORDER BY LAST_LOAD_TIME DESC;
```

#### Step 8: Upload a second batch and load it

Create `orders_batch2.csv`:
```
order_id,customer_name,product,quantity,unit_price,total_amount,status,order_date
1006,Sophie Dubois,Standing Desk,1,499.99,499.99,CONFIRMED,2024-01-12
1007,Marco Rossi,Mechanical Keyboard,1,189.99,189.99,CONFIRMED,2024-01-13
1008,Aisha Patel,Smart Watch,1,299.99,299.99,PENDING,2024-01-13
```

Upload it to `ORDERS_STAGE`, then:

```sql
ALTER PIPE ECOMM_DB.RAW_DATA.ORDERS_PIPE REFRESH;

-- Wait 15–20 seconds
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD;
-- Expected: 8

-- Load history shows 2 files now
SELECT file_name, row_count, status
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME  => 'ORDERS_PIPE_LOAD',
    START_TIME  => DATEADD('minutes', -30, CURRENT_TIMESTAMP())
))
ORDER BY LAST_LOAD_TIME DESC;
```

#### Step 9: Test deduplication — trigger REFRESH again without new files

```sql
ALTER PIPE ECOMM_DB.RAW_DATA.ORDERS_PIPE REFRESH;

-- Wait 20 seconds
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD;
-- Still 8 — Snowpipe tracks loaded files, does not reload them
```

#### Step 10: Pause and resume the pipe

```sql
-- Pause: stops loading new files
ALTER PIPE ECOMM_DB.RAW_DATA.ORDERS_PIPE SET PIPE_EXECUTION_PAUSED = TRUE;

SELECT SYSTEM$PIPE_STATUS('ECOMM_DB.RAW_DATA.ORDERS_PIPE');
-- "executionState": "PAUSED"

-- Resume
ALTER PIPE ECOMM_DB.RAW_DATA.ORDERS_PIPE SET PIPE_EXECUTION_PAUSED = FALSE;

SELECT SYSTEM$PIPE_STATUS('ECOMM_DB.RAW_DATA.ORDERS_PIPE');
-- "executionState": "RUNNING"
```

---

## Concept 4 — Zero-Copy Cloning

### What is Zero-Copy Cloning?

Cloning creates an **instant copy** of a table, schema, or database that **costs zero extra storage** at clone time. The clone and the source share the same underlying data blocks — no bytes are duplicated.

```
CREATE DATABASE DEV CLONE PROD;
-- Runs in milliseconds regardless of PROD's size.
-- Storage cost at clone time: ZERO
```

This uses **Copy-on-Write (CoW)**: the clone is a new metadata pointer to the same physical blocks. New storage is only allocated when either the source or the clone is **modified** — and only for the changed blocks.

```
After cloning:
  PROD ──► [Block A] [Block B] [Block C]
  DEV  ──► [Block A] [Block B] [Block C]   ← same blocks, zero extra storage

After deleting old rows from DEV:
  PROD ──► [Block A] [Block B] [Block C]   ← untouched
  DEV  ──► [Block A*] [Block B] [Block C]  ← only the modified partition is new
```

### What can be cloned?

| Object | Example |
|---|---|
| Table | `CREATE TABLE orders_backup CLONE orders_raw` |
| Schema | `CREATE SCHEMA staging_backup CLONE staging` |
| Database | `CREATE DATABASE ecomm_dev CLONE ecomm_db` |
| Stage | `CREATE STAGE stg2 CLONE stg1` |
| Streams / Tasks | Cannot be cloned |

### Real-world use cases

| Scenario | Why cloning |
|---|---|
| Dev/test environment | Clone prod → developers get real data, zero storage cost |
| Pre-migration backup | Clone before a risky `ALTER TABLE` or bulk `UPDATE` |
| Disaster recovery | Accidentally deleted rows? Clone at a past point in time and restore |
| Data science sandbox | Each analyst gets their own clone — modify freely, no impact on prod |

---

### Hands-On: Zero-Copy Cloning

#### Step 1: Clone a single table

```sql
USE SCHEMA ECOMM_DB.RAW_DATA;

-- Make sure we have data in the source table
SELECT COUNT(*) FROM ORDERS_PIPE_LOAD;

-- Clone it
CREATE TABLE ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD_BACKUP
    CLONE ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD;

-- Verify both have the same row count
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD;
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD_BACKUP;
-- Both should return the same number
```

#### Step 2: Verify clone isolation — changes to source do not affect clone

```sql
-- Insert a new row into the SOURCE table
INSERT INTO ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD
VALUES (9999, 'Test User', 'Test Product', 1, 50.00, 50.00, 'PENDING', '2024-01-15');

-- Source: has the new row
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD;        -- +1

-- Clone: does NOT have it — it was cloned before this insert
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD_BACKUP; -- same as before
```

#### Step 3: Verify clone isolation — changes to clone do not affect source

```sql
-- Delete from the CLONE
DELETE FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD_BACKUP
WHERE order_id <= 1003;

-- Clone: fewer rows
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD_BACKUP;

-- Source: completely unaffected
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD;
```

#### Step 4: Clone a schema

```sql
-- Clone the entire RAW_DATA schema — all tables inside are cloned
CREATE SCHEMA ECOMM_DB.RAW_DATA_BACKUP
    CLONE ECOMM_DB.RAW_DATA;

-- See all tables in the cloned schema
SHOW TABLES IN SCHEMA ECOMM_DB.RAW_DATA_BACKUP;
-- You should see ORDERS_PIPE_LOAD, ORDERS_RAW_LOAD, etc. — all cloned instantly
```

#### Step 5: Clone at a point in time (Time Travel clone) — disaster recovery

```sql
USE SCHEMA ECOMM_DB.RAW_DATA;

-- Step 1: Note the current row count
SELECT COUNT(*) FROM ORDERS_PIPE_LOAD;

-- Step 2: Simulate an accidental delete
DELETE FROM ORDERS_PIPE_LOAD WHERE order_id > 1005;

SELECT COUNT(*) FROM ORDERS_PIPE_LOAD;
-- Rows are gone — disaster!

-- Step 3: Find the query ID of the bad delete
SELECT query_id, query_text, start_time
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY(
    DATE_RANGE_START => DATEADD('minutes', -5, CURRENT_TIMESTAMP())
))
WHERE query_text ILIKE '%DELETE%ORDERS_PIPE_LOAD%'
ORDER BY start_time DESC
LIMIT 1;
-- Copy the query_id value from the result

-- Step 4: Clone the table as it was BEFORE that exact query
CREATE TABLE ECOMM_DB.RAW_DATA.ORDERS_RECOVERED
    CLONE ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD
    BEFORE (STATEMENT => '<paste-your-query-id-here>');

-- Step 5: Verify the recovered table has all the rows
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_RECOVERED;
-- Should match the count before the accidental delete

-- Step 6: Restore the missing rows back into the live table
INSERT INTO ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD
SELECT * FROM ECOMM_DB.RAW_DATA.ORDERS_RECOVERED r
WHERE NOT EXISTS (
    SELECT 1 FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD s
    WHERE s.order_id = r.order_id
);

-- Step 7: Verify full count is restored
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD;

-- Step 8: Drop the recovery table
DROP TABLE ECOMM_DB.RAW_DATA.ORDERS_RECOVERED;
```

#### Step 6: Clone with a time offset (alternative to BEFORE STATEMENT)

```sql
-- Clone as the table was 10 minutes ago (offset in seconds)
CREATE TABLE ECOMM_DB.RAW_DATA.ORDERS_10MIN_AGO
    CLONE ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD
    AT (OFFSET => -600);  -- 600 seconds = 10 minutes

SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_10MIN_AGO;

DROP TABLE ECOMM_DB.RAW_DATA.ORDERS_10MIN_AGO;
```

---

## Teardown

```sql
-- Drop the pipe (always clean up pipes — they consume background resources)
DROP PIPE ECOMM_DB.RAW_DATA.ORDERS_PIPE;

-- Drop backup objects
DROP SCHEMA ECOMM_DB.RAW_DATA_BACKUP;
DROP TABLE  ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD_BACKUP;

-- Confirm
SHOW PIPES  IN DATABASE ECOMM_DB;
SHOW STAGES IN SCHEMA   ECOMM_DB.RAW_DATA;
```

---

## Summary

| Concept | Key command | Use |
|---|---|---|
| Internal Stage | `CREATE STAGE name FILE_FORMAT = (...)` | Landing zone for files |
| External Stage | `CREATE STAGE name URL='s3://...'` | Point to your own cloud storage |
| COPY INTO | `COPY INTO table FROM @stage` | Manual bulk load; your warehouse; dedup built-in |
| Snowpipe | `CREATE PIPE ... AS COPY INTO ...` | Automatic load on file arrival; serverless |
| Zero-Copy Clone | `CREATE TABLE t2 CLONE t1` | Instant copy; zero storage cost; CoW on writes |
| Time Travel Clone | `CLONE ... BEFORE (STATEMENT => 'id')` | Recover from accidental deletes/updates |

**Next:** Day 6 — Time Travel, Fail-Safe, and Query Optimization.
