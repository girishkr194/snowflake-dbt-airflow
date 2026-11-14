# Day 3 — COPY INTO, Streams, and Tasks

## What You Will Learn

By the end of Day 3 you will understand how data gets into Snowflake (bulk loading), how Snowflake tracks row-level changes (CDC with Streams), and how to automate pipelines on a schedule (Tasks).

**Overview — 3 concepts, 6 steps:**
1. Load data in bulk using `COPY INTO` from a stage
2. Control error handling and validate files before loading
3. Create a Stream on a table to capture change data
4. Query a Stream to see INSERT / UPDATE / DELETE metadata
5. Create a Task to run SQL on a schedule
6. Chain Tasks to build an automated mini-pipeline

---

## Concept 1 — COPY INTO: Bulk Loading Data

### Why COPY INTO exists

Snowflake tables store data in an internal compressed columnar format. You cannot `INSERT` a CSV or JSON file directly — files must land in a **stage** first, then `COPY INTO` reads them from the stage and loads them into tables.

`COPY INTO` is optimized for bulk loading — it runs in parallel across all nodes of your warehouse and processes thousands of files at once.

### Stage recap (from Day 2)

A stage is a pointer to a file storage location:

| Stage type | Reference | When to use |
|---|---|---|
| User stage | `@~` | Personal scratch work |
| Table stage | `@%TABLE_NAME` | Quick single-table loads |
| Named internal stage | `@STAGE_NAME` | Production pipelines |
| External stage | `@STAGE_NAME` pointing to Azure/S3/GCS | Files already in cloud storage |

### Approach 1 — Simplest COPY INTO

```sql
-- Load all files in the stage into the table
COPY INTO my_schema.my_table
FROM @my_schema.my_stage
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1);
```

### Approach 2 — COPY INTO with a named file format

```sql
-- Create the file format once
CREATE FILE FORMAT IF NOT EXISTS CSV_FORMAT
  TYPE                        = 'CSV'
  FIELD_DELIMITER             = ','
  SKIP_HEADER                 = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  NULL_IF                     = ('NULL', 'null', '')
  EMPTY_FIELD_AS_NULL         = TRUE
  DATE_FORMAT                 = 'YYYY-MM-DD'
  COMPRESSION                 = 'AUTO';

-- Reference it by name in every COPY INTO
COPY INTO my_schema.my_table
FROM @my_schema.my_stage
FILE_FORMAT = (FORMAT_NAME = 'my_schema.CSV_FORMAT');
```

Named formats are reusable — change the format in one place and all COPY INTOs that reference it pick up the change automatically.

### Approach 3 — COPY INTO with column mapping and METADATA

Use a `SELECT` subquery inside the `FROM` clause to reorder columns, transform values, or capture audit metadata:

```sql
COPY INTO RAW_DATA.CUSTOMERS_RAW
  (customer_id, full_name, email, created_date, _source_file, _loaded_at)
FROM (
  SELECT
    $1::NUMBER,                    -- column 1 from file → customer_id
    $3::VARCHAR,                   -- column 3 → full_name (skipping column 2)
    LOWER($4::VARCHAR),            -- transform to lowercase
    $5::DATE,
    METADATA$FILENAME,             -- audit: which file this row came from
    CURRENT_TIMESTAMP()
  FROM @RAW_DATA.INTERNAL_STAGE
)
FILE_FORMAT = (FORMAT_NAME = 'RAW_DATA.CSV_FORMAT')
ON_ERROR    = 'ABORT_STATEMENT';
```

`$1`, `$2`, `$3`... refer to file columns by position (1-indexed).

### Approach 4 — COPY INTO with PATTERN to filter files

```sql
-- Load only January 2024 files from a stage that has many months
COPY INTO RAW_DATA.ORDERS_RAW
FROM @RAW_DATA.ORDERS_STAGE
FILE_FORMAT = (FORMAT_NAME = 'RAW_DATA.CSV_FORMAT')
PATTERN     = '.*orders_2024_01.*\.csv\.gz'   -- Java regex
ON_ERROR    = 'ABORT_STATEMENT';
```

### Approach 5 — Dry run before loading (VALIDATION_MODE)

```sql
-- Preview errors without loading any data
COPY INTO my_table FROM @my_stage
FILE_FORMAT     = (FORMAT_NAME = 'CSV_FORMAT')
VALIDATION_MODE = 'RETURN_ERRORS';

-- Preview first 10 rows that would be loaded
COPY INTO my_table FROM @my_stage
FILE_FORMAT     = (FORMAT_NAME = 'CSV_FORMAT')
VALIDATION_MODE = 'RETURN_10_ROWS';
```

No rows are inserted. Use this before every first production load to validate the file format assumptions.

### Approach 6 — Force reload (bypass load history)

```sql
COPY INTO my_table FROM @my_stage
FILE_FORMAT = (FORMAT_NAME = 'CSV_FORMAT')
FORCE       = TRUE;   -- ignore load history, reload everything
```

### ON_ERROR modes

| Mode | Behaviour | Use when |
|---|---|---|
| `ABORT_STATEMENT` | Stop on first bad row, load nothing | Financial data, must-be-complete loads |
| `CONTINUE` | Skip bad rows, load everything else | Log data, analytics events |
| `SKIP_FILE` | Skip the entire file if any row fails | When partial files are unacceptable |
| `SKIP_FILE_N%` | Skip a file only if more than N% of rows fail | Mixed quality tolerance |

### Idempotency — why COPY INTO never double-loads

Snowflake tracks every file loaded in an internal **load history** (kept for 64 days). When COPY INTO runs, it checks the history — files already loaded are silently skipped.

```
First run:  10 files → 100,000 rows loaded
Second run: same 10 files → 0 rows loaded (all skipped, already in history)
```

This is idempotent by design. Use `FORCE = TRUE` to override.

### METADATA columns available during COPY INTO

| Column | What it contains |
|---|---|
| `METADATA$FILENAME` | Full path and name of the source file |
| `METADATA$FILE_ROW_NUMBER` | Row number within the source file |
| `METADATA$FILE_CONTENT_KEY` | MD5 hash of the file |
| `METADATA$FILE_LAST_MODIFIED` | Last modified timestamp of the source file |
| `METADATA$START_SCAN_TIME` | Timestamp when this COPY INTO started |

Always capture `METADATA$FILENAME` in raw tables — it is the foundation of data lineage and debugging.

### Check load history after COPY INTO

```sql
SELECT
    file_name,
    status,
    rows_parsed,
    rows_loaded,
    rows_parsed - rows_loaded AS rows_failed,
    first_error_message,
    last_load_time
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'CUSTOMERS_RAW',
    START_TIME => DATEADD('hours', -1, CURRENT_TIMESTAMP())
))
ORDER BY last_load_time DESC;
```

---

## Concept 2 — Streams: Change Data Capture (CDC)

### What is CDC?

**Change Data Capture** is the technique of detecting and capturing row-level changes (INSERT, UPDATE, DELETE) as they happen in a source table, rather than re-scanning the whole table every time.

Without CDC, your ETL must either reload everything or use a "last updated" timestamp to find changed rows — both are fragile. With CDC, Snowflake automatically records what changed and when.

### What is a Snowflake Stream?

A **Stream** is an object that sits on top of a table and records every DML change (INSERT, UPDATE, DELETE) made to that table since the stream was last consumed.

```
Source table ──► [Stream] ──► Consumer (Task / ETL query)
```

The stream does not store a copy of the data — it stores **change records**: what row changed, what the new value is, and what type of change occurred.

### Creating a Stream

```sql
-- Standard stream: captures INSERT, UPDATE, DELETE
CREATE STREAM IF NOT EXISTS RAW_DATA.CUSTOMERS_STREAM
  ON TABLE RAW_DATA.CUSTOMERS_RAW
  COMMENT = 'CDC stream on customers raw table';

-- Append-only stream: captures only INSERTs (more efficient for log/event tables)
CREATE STREAM IF NOT EXISTS RAW_DATA.ORDERS_STREAM
  ON TABLE RAW_DATA.ORDERS_RAW
  APPEND ONLY = TRUE;
```

Use `APPEND ONLY = TRUE` for tables where rows are only ever inserted (logs, events, raw ingest tables) — it is more efficient because Snowflake does not need to track UPDATE and DELETE offsets.

### Stream metadata columns

Every row in a stream has three extra metadata columns:

| Column | Type | Values | Meaning |
|---|---|---|---|
| `METADATA$ACTION` | VARCHAR | `INSERT` or `DELETE` | Type of change |
| `METADATA$ISUPDATE` | BOOLEAN | `TRUE` or `FALSE` | Was this change part of an UPDATE? |
| `METADATA$ROW_ID` | VARCHAR | unique ID | Identifies the specific row that changed |

**How UPDATE is represented:**
An UPDATE in Snowflake streams appears as TWO rows:
- One `DELETE` row (the old values, `METADATA$ISUPDATE = TRUE`)
- One `INSERT` row (the new values, `METADATA$ISUPDATE = TRUE`)

This is called a "pre/post image" pattern.

### Querying a Stream

```sql
-- See all pending changes
SELECT * FROM RAW_DATA.CUSTOMERS_STREAM;

-- See only new inserts
SELECT * FROM RAW_DATA.CUSTOMERS_STREAM
WHERE METADATA$ACTION = 'INSERT' AND METADATA$ISUPDATE = FALSE;

-- See only updates (the new value rows)
SELECT * FROM RAW_DATA.CUSTOMERS_STREAM
WHERE METADATA$ACTION = 'INSERT' AND METADATA$ISUPDATE = TRUE;

-- See only deletes
SELECT * FROM RAW_DATA.CUSTOMERS_STREAM
WHERE METADATA$ACTION = 'DELETE' AND METADATA$ISUPDATE = FALSE;
```

### How the stream offset works

The stream has an internal **offset** (a timestamp pointer). Every time you consume the stream inside a DML transaction (INSERT INTO ... SELECT FROM stream, MERGE using stream), the offset advances to the current time and those change records disappear from the stream.

```
Make changes → query stream → see changes → consume in DML → stream is empty
```

If you just `SELECT` from the stream without using it in a DML, the offset does not advance — the changes remain visible.

### Practical CDC pattern — MERGE from stream

The most common use: use a stream to drive a MERGE that keeps a staging table in sync with the raw table:

```sql
MERGE INTO STAGING.STG_CUSTOMERS AS target
USING (
  SELECT * FROM RAW_DATA.CUSTOMERS_STREAM
  WHERE METADATA$ACTION = 'INSERT'
) AS src
ON target.customer_id = src.customer_id
WHEN MATCHED THEN
  UPDATE SET
    target.full_name   = src.full_name,
    target.email       = src.email,
    target.updated_at  = CURRENT_TIMESTAMP()
WHEN NOT MATCHED THEN
  INSERT (customer_id, full_name, email, updated_at)
  VALUES (src.customer_id, src.full_name, src.email, CURRENT_TIMESTAMP());
```

This is the foundation of **SCD Type 1** (overwrite with latest values) in Snowflake.

### Check stream status

```sql
SHOW STREAMS IN SCHEMA RAW_DATA;
-- Look at: stale (TRUE means changes were lost), mode, source_table, offset
```

A stream becomes **stale** if changes are not consumed within the **data retention period** of the source table (default 1 day on Standard, up to 90 days on Enterprise). Stale streams must be dropped and recreated.

---

## Concept 3 — Tasks: Scheduling and Automation

### What is a Task?

A **Task** is a Snowflake object that runs a single SQL statement (or a stored procedure call) on a schedule. Tasks are how you automate your pipeline — instead of manually running COPY INTO or the MERGE from your stream, a Task runs it for you on a timer.

Tasks replace the need for external schedulers (Airflow, cron jobs) for simple Snowflake-internal pipelines.

### Creating a Task

```sql
-- Task that runs every 5 minutes
CREATE TASK IF NOT EXISTS STAGING.LOAD_CUSTOMERS_TASK
  WAREHOUSE = LOADING_WH
  SCHEDULE  = '5 MINUTE'
AS
COPY INTO RAW_DATA.CUSTOMERS_RAW
FROM @RAW_DATA.CUSTOMERS_STAGE
FILE_FORMAT = (FORMAT_NAME = 'RAW_DATA.CSV_FORMAT')
ON_ERROR    = 'CONTINUE';
```

```sql
-- Task with a CRON schedule (run at 2 AM UTC every day)
CREATE TASK IF NOT EXISTS STAGING.DAILY_LOAD_TASK
  WAREHOUSE = LOADING_WH
  SCHEDULE  = 'USING CRON 0 2 * * * UTC'
AS
COPY INTO RAW_DATA.ORDERS_RAW
FROM @RAW_DATA.ORDERS_STAGE
FILE_FORMAT = (FORMAT_NAME = 'RAW_DATA.CSV_FORMAT');
```

### CRON syntax

```
USING CRON <min> <hour> <day-of-month> <month> <day-of-week> <timezone>

Examples:
  0 2 * * *      → 2 AM every day
  0 */6 * * *    → every 6 hours
  */5 * * * *    → every 5 minutes
  0 9 * * 1      → 9 AM every Monday
  0 0 1 * *      → midnight on the 1st of every month
```

### Tasks start SUSPENDED — you must resume them

```sql
-- Tasks are created in SUSPENDED state by default
ALTER TASK STAGING.LOAD_CUSTOMERS_TASK RESUME;

-- Check task status
SHOW TASKS IN SCHEMA STAGING;

-- Suspend a task (pause it without dropping)
ALTER TASK STAGING.LOAD_CUSTOMERS_TASK SUSPEND;
```

### Conditional execution — only run if the stream has data

A Task can include a `WHEN` clause so it only runs (and spends warehouse credits) when there is actually something to process:

```sql
CREATE TASK IF NOT EXISTS STAGING.PROCESS_CUSTOMER_CHANGES_TASK
  WAREHOUSE = LOADING_WH
  SCHEDULE  = '1 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('RAW_DATA.CUSTOMERS_STREAM')   -- only run if stream is not empty
AS
MERGE INTO STAGING.STG_CUSTOMERS AS target
USING RAW_DATA.CUSTOMERS_STREAM AS src
ON target.customer_id = src.customer_id
WHEN MATCHED AND src.METADATA$ACTION = 'INSERT' THEN
  UPDATE SET target.full_name = src.full_name, target.email = src.email
WHEN NOT MATCHED AND src.METADATA$ACTION = 'INSERT' THEN
  INSERT (customer_id, full_name, email) VALUES (src.customer_id, src.full_name, src.email);
```

`SYSTEM$STREAM_HAS_DATA()` returns TRUE only when there are unconsumed changes in the stream. The task skips execution (no warehouse credit used) when it returns FALSE.

### Chaining Tasks — building a DAG

Tasks can depend on other tasks using `AFTER`. This creates a **directed acyclic graph (DAG)** of task dependencies:

```sql
-- Root task: runs on a schedule (the trigger)
CREATE TASK PIPELINE.ROOT_TASK
  WAREHOUSE = LOADING_WH
  SCHEDULE  = 'USING CRON 0 3 * * * UTC'   -- 3 AM daily
AS
COPY INTO RAW_DATA.CUSTOMERS_RAW
FROM @RAW_DATA.CUSTOMERS_STAGE
FILE_FORMAT = (FORMAT_NAME = 'RAW_DATA.CSV_FORMAT');

-- Child task 1: runs after ROOT_TASK completes
CREATE TASK PIPELINE.TRANSFORM_TASK
  WAREHOUSE = TRANSFORM_WH
  AFTER PIPELINE.ROOT_TASK              -- dependency
AS
INSERT INTO STAGING.STG_CUSTOMERS
SELECT customer_id, full_name, email, CURRENT_TIMESTAMP()
FROM RAW_DATA.CUSTOMERS_RAW
WHERE _loaded_at >= DATEADD('hours', -4, CURRENT_TIMESTAMP());

-- Child task 2: runs after TRANSFORM_TASK completes
CREATE TASK PIPELINE.MART_TASK
  WAREHOUSE = TRANSFORM_WH
  AFTER PIPELINE.TRANSFORM_TASK
AS
INSERT INTO MARTS.CUSTOMER_MART
SELECT * FROM STAGING.STG_CUSTOMERS
WHERE updated_at >= DATEADD('hours', -4, CURRENT_TIMESTAMP());
```

```sql
-- Resume all tasks in the right order: children first, root last
ALTER TASK PIPELINE.MART_TASK RESUME;
ALTER TASK PIPELINE.TRANSFORM_TASK RESUME;
ALTER TASK PIPELINE.ROOT_TASK RESUME;   -- root task triggers the chain
```

Only the root task has a `SCHEDULE`. Child tasks have no schedule — they run only when their parent completes successfully.

### Check task run history

```sql
-- See recent task runs — status, duration, errors
SELECT
    name,
    state,
    scheduled_time,
    completed_time,
    error_message,
    DATEDIFF('seconds', scheduled_time, completed_time) AS duration_seconds
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    SCHEDULED_TIME_RANGE_START => DATEADD('hours', -24, CURRENT_TIMESTAMP()),
    TASK_NAME => 'LOAD_CUSTOMERS_TASK'
))
ORDER BY scheduled_time DESC;
```

### Manually trigger a task (without waiting for schedule)

```sql
EXECUTE TASK STAGING.LOAD_CUSTOMERS_TASK;
```

Useful for testing a task immediately after creation without waiting for the next scheduled run.

---

## Summary

| Concept | Key object | Key command | What it solves |
|---|---|---|---|
| Bulk loading | Stage + File Format | `COPY INTO` | Getting files into tables efficiently |
| Change tracking | Stream | `CREATE STREAM ON TABLE` | Knowing what changed since last run |
| Automation | Task | `CREATE TASK ... SCHEDULE` | Running SQL on a timer without external tools |

**The full pipeline pattern:**
```
Files in stage
     │
     ▼
COPY INTO → RAW table
     │
     ▼ (Stream captures changes)
Stream → MERGE INTO → STAGING table
     │
     ▼ (Task runs this on schedule)
Task chain → MARTS table
```

**Next:** Day 4 covers semi-structured data — VARIANT, loading JSON files, colon syntax, LATERAL FLATTEN, and querying nested structures.
