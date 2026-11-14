# Day 3 — Interview Solutions: COPY INTO, Streams, and Tasks

> Complete answers to all 23 questions. Read these after attempting the questions yourself.

---

## Concept 1: COPY INTO — Bulk Loading

**Q1 — What does COPY INTO do, and why can't you INSERT a CSV directly?**

`COPY INTO` reads files from a stage and bulk-loads them into a Snowflake table. It is the primary mechanism for getting file-based data into Snowflake in production.

**Why not INSERT a CSV directly:** Snowflake tables store data in an internal compressed columnar format (micro-partitions). There is no concept of "uploading a file" into a table directly. Files must land in a **stage** (a storage pointer) first. `COPY INTO` then reads those files from the stage, parses them according to a file format, and writes the rows into the table's internal format in parallel.

The separation of staging from loading also gives you validation, error handling, and idempotency that a plain INSERT cannot provide.

---

**Q2 — What does "idempotent" mean for COPY INTO? How long is load history kept?**

**Idempotent** means running the same COPY INTO command multiple times produces the same result as running it once — no duplicate rows.

Snowflake achieves this by tracking every loaded file in an internal **load history**. When COPY INTO runs, it checks the history — files already loaded successfully are silently skipped.

**Load history is kept for 64 days.**

```
First run:  10 files → 100,000 rows loaded
Second run: same 10 files → 0 rows loaded (all skipped)
```

To override: `FORCE = TRUE` bypasses history for that run only. Use it carefully — it can create duplicate rows if you don't truncate the target table first.

---

**Q3 — What is VALIDATION_MODE? Two values, and does it load data?**

`VALIDATION_MODE` puts COPY INTO into a **dry-run mode**. It parses the files and checks every row but **loads no data** into the table.

| Value | What it returns |
|---|---|
| `RETURN_ERRORS` | A result set of every row that would fail — error message, file name, row number, bad value |
| `RETURN_10_ROWS` | The first 10 rows that would be successfully loaded (preview) |
| `RETURN_ALL_ERRORS` | Like RETURN_ERRORS but includes errors from files that would be fully skipped |

```sql
-- Dry run — see errors
COPY INTO my_table FROM @my_stage
FILE_FORMAT     = (FORMAT_NAME = 'CSV_FORMAT')
VALIDATION_MODE = 'RETURN_ERRORS';
-- 0 rows in my_table after this. Only a result set of errors returned.

-- Preview 10 rows
COPY INTO my_table FROM @my_stage
VALIDATION_MODE = 'RETURN_10_ROWS';
```

Use this before every first production load.

---

**Q4 — `ABORT_STATEMENT` vs `CONTINUE`. Which for financial data?**

| Setting | Behaviour |
|---|---|
| `ABORT_STATEMENT` | Stop on the first bad row. No rows from any file are committed. All-or-nothing. |
| `CONTINUE` | Skip bad rows and log them. All valid rows from all files are committed. |

**For financial data: `ABORT_STATEMENT`**

Financial data requires 100% completeness. If a row fails (e.g., an amount field has corrupted data), you want the load to stop completely so the issue is immediately visible. Silently committing partial financial data with `CONTINUE` could cause:
- Incorrect account balances
- Missing transactions that balance sheets rely on
- Compliance violations (SOX, PCI-DSS require complete, auditable data)

Use `CONTINUE` only when missing a small percentage of rows is acceptable — e.g., log data or analytics events.

---

**Q5 — 300 files in a stage, load only January 2024.**

```sql
COPY INTO RAW_DATA.ORDERS_RAW
FROM @RAW_DATA.ORDERS_STAGE
FILE_FORMAT = (FORMAT_NAME = 'RAW_DATA.CSV_FORMAT')
PATTERN     = '.*orders_2024_01.*\.csv\.gz'
ON_ERROR    = 'ABORT_STATEMENT';
```

`PATTERN` takes a **Java regular expression**:
- `.*` — match any prefix or suffix
- `\.` — literal dot (escaped because `.` means "any char" in regex)
- `_2024_01_` — literal string to match January 2024

Alternatively, use `FILES = ('file1.csv', 'file2.csv', ...)` to list specific files explicitly, but PATTERN is cleaner when files are numerous.

---

**Q6 — What are `$1`, `$2`, `$3`? When to use them?**

`$1`, `$2`, `$3`... are **positional file column references** — they refer to the 1st, 2nd, 3rd column in the source file.

They are used inside a `SELECT` subquery in the `FROM` clause:

```sql
COPY INTO my_table (col_a, col_b, col_c)
FROM (
  SELECT
    $3,                -- take 3rd file column into col_a (reordering)
    UPPER($1),         -- transform 1st file column
    METADATA$FILENAME  -- audit column
  FROM @my_stage
)
FILE_FORMAT = (...);
```

**Use when:**
- File column order differs from table column order
- You want to skip certain file columns
- You need to transform values during load (cast, clean, compute)
- You want to capture `METADATA$FILENAME` as an audit column

Without a SELECT subquery, COPY INTO maps file columns to table columns positionally with no flexibility.

---

**Q7 — What is `METADATA$FILENAME`? Two other metadata columns?**

`METADATA$FILENAME` is a **virtual column** available during COPY INTO that contains the full stage path and name of the source file for every row being loaded.

```sql
COPY INTO RAW_DATA.CUSTOMERS_RAW (_source_file, customer_id, name)
FROM (
  SELECT METADATA$FILENAME, $1, $2
  FROM @my_stage
);
```

**Why capture it in every raw table:** When a data quality issue is found months later, `METADATA$FILENAME` lets you answer "which file contained this bad row?" and reload or investigate just that file. Without it, you have no lineage.

**Two other metadata columns:**

| Column | Contains |
|---|---|
| `METADATA$FILE_ROW_NUMBER` | Row number of this record within its source file |
| `METADATA$FILE_LAST_MODIFIED` | Last modified timestamp of the source file |
| `METADATA$FILE_CONTENT_KEY` | MD5 hash of the file — uniquely identifies the file version |
| `METADATA$START_SCAN_TIME` | Timestamp when this COPY INTO started scanning |

---

**Q8 — Run same COPY INTO twice. Second run loads how many rows? Force reload?**

**Second run loads 0 rows.**

After the first run, Snowflake recorded all 10 files as successfully loaded in its load history. The second COPY INTO checks history, sees all files are already loaded, and skips every one — completing immediately with "0 files processed."

**To force a reload:**

```sql
COPY INTO my_table FROM @my_stage
FILE_FORMAT = (FORMAT_NAME = 'CSV_FORMAT')
FORCE = TRUE;   -- bypass load history
```

`FORCE = TRUE` does not clear the history — it overrides it for this specific run. The rows are inserted again, potentially creating duplicates. Best practice: `TRUNCATE TABLE my_table` first if you want a clean reload.

---

## Concept 2: Streams — Change Data Capture

**Q9 — What is a Snowflake Stream? What problem does it solve?**

A **Stream** is a Snowflake object that sits on top of a table and records every DML change (INSERT, UPDATE, DELETE) made to that table since the stream was last consumed.

**Problem it solves:** Without CDC, your ETL pipeline has two bad options:
1. **Full reload** — truncate and reload the entire target table every run. Expensive and slow as the table grows.
2. **Timestamp-based delta** — query rows where `updated_at > last_run_time`. Fragile — requires every table to have an `updated_at` column, and that column must be reliably maintained.

A Stream solves both problems. Snowflake automatically tracks what changed, and your pipeline only processes the delta — regardless of table size.

---

**Q10 — Three stream metadata columns? Possible values for `METADATA$ACTION`?**

| Column | Possible values | Meaning |
|---|---|---|
| `METADATA$ACTION` | `'INSERT'` or `'DELETE'` | Type of change |
| `METADATA$ISUPDATE` | `TRUE` or `FALSE` | Was this row part of an UPDATE? |
| `METADATA$ROW_ID` | unique string | Identifies the specific row that changed |

`METADATA$ACTION` only ever has two values — `INSERT` and `DELETE`. There is no `UPDATE` action. See Q11 for why.

---

**Q11 — How does a stream represent an UPDATE? How many rows?**

A single UPDATE appears as **two rows** in the stream:

| Row | `METADATA$ACTION` | `METADATA$ISUPDATE` | Content |
|---|---|---|---|
| Row 1 | `DELETE` | `TRUE` | The old values (before update) |
| Row 2 | `INSERT` | `TRUE` | The new values (after update) |

This is called the **pre/post image** pattern. Snowflake stores the before and after states of the row, not a "diff."

To get only the new values (what you usually want):
```sql
SELECT * FROM my_stream
WHERE METADATA$ACTION = 'INSERT' AND METADATA$ISUPDATE = TRUE;
```

To get the old values (useful for auditing what changed):
```sql
SELECT * FROM my_stream
WHERE METADATA$ACTION = 'DELETE' AND METADATA$ISUPDATE = TRUE;
```

---

**Q12 — Standard stream vs APPEND ONLY stream. When to use each?**

| Type | Tracks | Performance | Use when |
|---|---|---|---|
| Standard | INSERT, UPDATE, DELETE | Slightly more overhead | Tables where rows can be updated or deleted |
| Append-only | INSERT only | More efficient | Tables where rows are only ever added |

```sql
-- Standard — full CDC
CREATE STREAM customers_stream ON TABLE customers_raw;

-- Append-only — inserts only, more efficient
CREATE STREAM orders_stream ON TABLE orders_raw APPEND ONLY = TRUE;
```

Append-only is ideal for raw ingest tables (event logs, IoT data, audit trails) where rows are written once and never modified. It is more efficient because Snowflake doesn't need to track the update/delete offset.

---

**Q13 — You SELECT from a stream. Does the offset advance?**

**No. A plain `SELECT` does not advance the stream offset.**

The offset only advances when the stream is consumed inside a **DML transaction**:
- `INSERT INTO ... SELECT FROM stream`
- `MERGE INTO ... USING stream`
- `UPDATE ... FROM stream`
- `DELETE FROM ... USING stream`

This means you can query the stream as many times as you want to inspect the changes without affecting it. The changes only "disappear" from the stream once you use them in a DML statement that commits successfully.

If the DML fails and rolls back, the stream offset rolls back too — the changes remain in the stream for the next attempt.

---

**Q14 — Write a MERGE that uses a CUSTOMERS_STREAM to sync STG_CUSTOMERS.**

```sql
MERGE INTO STAGING.STG_CUSTOMERS AS target
USING (
  SELECT * FROM RAW_DATA.CUSTOMERS_STREAM
  WHERE METADATA$ACTION = 'INSERT'   -- both new inserts and the new-value row of updates
) AS src
ON target.customer_id = src.customer_id
WHEN MATCHED THEN
  UPDATE SET
    target.full_name  = src.full_name,
    target.email      = src.email,
    target.phone      = src.phone,
    target.updated_at = CURRENT_TIMESTAMP()
WHEN NOT MATCHED THEN
  INSERT (customer_id, full_name, email, phone, updated_at)
  VALUES (src.customer_id, src.full_name, src.email, src.phone, CURRENT_TIMESTAMP());
```

**Why filter to `METADATA$ACTION = 'INSERT'`:** For new rows, action is `INSERT`. For updates, the new-value row also has action `INSERT` (with `METADATA$ISUPDATE = TRUE`). Both should be applied to the target. The `DELETE` rows (old values of updates) are ignored — we only want the latest state.

This pattern is **SCD Type 1** — overwrite with the latest values.

---

**Q15 — What does it mean for a stream to become "stale"? How to fix it?**

A stream becomes **stale** when the changes it was tracking have aged out of the source table's **Time Travel** history.

Streams internally rely on Time Travel to store the before-image of changed rows. If you don't consume the stream within the source table's data retention period (default: 1 day on Standard, up to 90 days on Enterprise), Snowflake can no longer reconstruct those change records.

```sql
SHOW STREAMS;
-- If the "stale" column is TRUE, the stream can no longer be used
```

**How to fix:** You cannot "un-stale" a stream. You must:
1. Drop the stale stream
2. Recreate it — the new stream starts tracking from the current moment
3. Perform a full reload of the target table (since you lost the incremental changes)

**Prevention:** Ensure your Tasks run frequently enough to consume the stream before its data ages out. If your table has 1-day retention, your Task must run at least once per day.

---

## Concept 3: Tasks — Scheduling and Automation

**Q16 — What is a Snowflake Task? Two scheduling approaches?**

A **Task** is a Snowflake object that runs a single SQL statement (or stored procedure call) on a schedule. It is Snowflake's native job scheduler — no Airflow, cron jobs, or external tools needed for Snowflake-internal pipelines.

**Two scheduling approaches:**

1. **Interval-based** — run every N minutes:
   ```sql
   SCHEDULE = '5 MINUTE'    -- every 5 minutes
   SCHEDULE = '60 MINUTE'   -- every hour
   ```

2. **CRON-based** — run at specific times:
   ```sql
   SCHEDULE = 'USING CRON 0 2 * * * UTC'    -- 2 AM every day
   SCHEDULE = 'USING CRON 0 9 * * 1 UTC'    -- 9 AM every Monday
   SCHEDULE = 'USING CRON */15 * * * * UTC'  -- every 15 minutes
   ```

---

**Q17 — Why are tasks created SUSPENDED? How do you start one?**

Tasks are created suspended by default to prevent accidentally running a task before it has been fully configured and tested. Creating a task immediately triggers its schedule — if you're still building the pipeline, you don't want it firing.

The pattern is:
1. Create the task (suspended)
2. Test it manually with `EXECUTE TASK`
3. Verify results
4. Resume it to activate the schedule

```sql
-- Activate a task
ALTER TASK STAGING.LOAD_CUSTOMERS_TASK RESUME;

-- Pause without dropping
ALTER TASK STAGING.LOAD_CUSTOMERS_TASK SUSPEND;

-- Check current state
SHOW TASKS IN SCHEMA STAGING;
-- Look at the "state" column: "started" = running on schedule, "suspended" = paused
```

---

**Q18 — Write a Task that runs at 2 AM UTC daily and runs a COPY INTO.**

```sql
CREATE TASK IF NOT EXISTS RAW_DATA.DAILY_CUSTOMER_LOAD_TASK
  WAREHOUSE = LOADING_WH
  SCHEDULE  = 'USING CRON 0 2 * * * UTC'
AS
COPY INTO RAW_DATA.CUSTOMERS_RAW
  (customer_id, full_name, email, phone, city, country, _source_file)
FROM (
  SELECT $1::NUMBER, $2::VARCHAR, $3::VARCHAR, $4::VARCHAR, $5::VARCHAR, $6::VARCHAR,
         METADATA$FILENAME
  FROM @RAW_DATA.CUSTOMERS_STAGE
)
FILE_FORMAT = (FORMAT_NAME = 'RAW_DATA.CSV_FORMAT')
ON_ERROR    = 'ABORT_STATEMENT';

-- Activate after creation
ALTER TASK RAW_DATA.DAILY_CUSTOMER_LOAD_TASK RESUME;
```

---

**Q19 — What does `WHEN SYSTEM$STREAM_HAS_DATA(...)` do? Why is it important for cost?**

`SYSTEM$STREAM_HAS_DATA('schema.stream_name')` is a system function that returns `TRUE` if the stream has unconsumed change records, and `FALSE` if it is empty.

When used in a Task's `WHEN` clause, the task only executes its SQL if the condition is `TRUE`:

```sql
CREATE TASK process_changes_task
  WAREHOUSE = TRANSFORM_WH
  SCHEDULE  = '1 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('RAW_DATA.CUSTOMERS_STREAM')
AS
MERGE INTO STAGING.STG_CUSTOMERS ...;
```

**Why it matters for cost:** Tasks that run on short intervals (every 1 or 5 minutes) would wake up and spin the warehouse whether or not there is any data to process. Each warehouse startup costs compute credits.

With `WHEN SYSTEM$STREAM_HAS_DATA(...)`, if the stream is empty, the task **skips execution** — no warehouse is started, no credits are consumed. This can reduce warehouse costs dramatically for pipelines that only occasionally receive data.

---

**Q20 — How does Task chaining work? Root vs child tasks? Resume order?**

Tasks can depend on other tasks using the `AFTER` keyword, forming a **DAG (directed acyclic graph)**:

```sql
-- Root task: has a schedule, starts the chain
CREATE TASK root_task
  WAREHOUSE = WH
  SCHEDULE  = 'USING CRON 0 3 * * * UTC'
AS <sql>;

-- Child task: no schedule, runs after root
CREATE TASK child_task_1
  WAREHOUSE = WH
  AFTER root_task
AS <sql>;

-- Grandchild task: runs after child_task_1
CREATE TASK child_task_2
  WAREHOUSE = WH
  AFTER child_task_1
AS <sql>;
```

**Root task:** Has a `SCHEDULE`. It is the trigger for the whole chain. Only root tasks fire on a schedule.

**Child task:** Has `AFTER <parent>` instead of a schedule. It fires when its parent completes successfully. If the parent fails, the child does not run.

**Resume order — children first, root last:**

```sql
ALTER TASK child_task_2 RESUME;    -- resume leaf first
ALTER TASK child_task_1 RESUME;    -- then intermediate
ALTER TASK root_task RESUME;       -- root last — this activates the schedule
```

You resume children before the root because Snowflake validates the DAG when the root is resumed. If a child is still suspended, the root resume will fail.

---

**Q21 (System design) — Automated e-commerce pipeline: hourly CSV → raw → staging → mart.**

```sql
-- ── STAGE AND FILE FORMAT ─────────────────────────────────────────────────
CREATE STAGE IF NOT EXISTS RAW_DATA.ORDERS_STAGE
  URL         = 'azure://account.blob.core.windows.net/orders/'
  CREDENTIALS = (AZURE_SAS_TOKEN = '<token>');

CREATE FILE FORMAT IF NOT EXISTS RAW_DATA.CSV_FORMAT
  TYPE = 'CSV' SKIP_HEADER = 1 FIELD_DELIMITER = ','
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  NULL_IF = ('NULL', '') EMPTY_FIELD_AS_NULL = TRUE
  DATE_FORMAT = 'YYYY-MM-DD' COMPRESSION = 'AUTO';

-- ── RAW TABLE ─────────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS RAW_DATA.ORDERS_RAW (
    order_id      NUMBER,
    customer_id   NUMBER,
    order_date    DATE,
    total_amount  NUMBER(12,2),
    status        VARCHAR,
    _source_file  VARCHAR,
    _loaded_at    TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- ── STREAM ON RAW TABLE ───────────────────────────────────────────────────
CREATE STREAM IF NOT EXISTS RAW_DATA.ORDERS_STREAM
  ON TABLE RAW_DATA.ORDERS_RAW
  APPEND ONLY = TRUE;   -- orders are only ever inserted, never updated

-- ── STAGING TABLE ─────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS STAGING.STG_ORDERS (
    order_id      NUMBER,
    customer_id   NUMBER,
    order_date    DATE,
    total_amount  NUMBER(12,2),
    status        VARCHAR,
    _loaded_at    TIMESTAMP_NTZ
);

-- ── MART TABLE ────────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS MARTS.DAILY_ORDER_SUMMARY (
    order_date        DATE,
    total_orders      NUMBER,
    total_revenue     NUMBER(14,2),
    avg_order_value   NUMBER(12,2),
    refreshed_at      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- ── ROOT TASK: load raw from stage every hour ─────────────────────────────
CREATE TASK IF NOT EXISTS PIPELINE.LOAD_ORDERS_TASK
  WAREHOUSE = LOADING_WH
  SCHEDULE  = 'USING CRON 0 * * * * UTC'    -- top of every hour
AS
COPY INTO RAW_DATA.ORDERS_RAW
  (order_id, customer_id, order_date, total_amount, status, _source_file)
FROM (
  SELECT $1::NUMBER, $2::NUMBER, $3::DATE, $4::NUMBER(12,2), $5::VARCHAR,
         METADATA$FILENAME
  FROM @RAW_DATA.ORDERS_STAGE
)
FILE_FORMAT = (FORMAT_NAME = 'RAW_DATA.CSV_FORMAT')
ON_ERROR    = 'SKIP_FILE_5%';   -- skip file if more than 5% of rows fail

-- ── CHILD TASK 1: apply stream changes to staging ─────────────────────────
CREATE TASK IF NOT EXISTS PIPELINE.STAGE_ORDERS_TASK
  WAREHOUSE = TRANSFORM_WH
  AFTER     PIPELINE.LOAD_ORDERS_TASK
  WHEN      SYSTEM$STREAM_HAS_DATA('RAW_DATA.ORDERS_STREAM')   -- only if changes exist
AS
INSERT INTO STAGING.STG_ORDERS
SELECT order_id, customer_id, order_date, total_amount, status, _loaded_at
FROM RAW_DATA.ORDERS_STREAM
WHERE METADATA$ACTION = 'INSERT';

-- ── CHILD TASK 2: refresh mart from staging ───────────────────────────────
CREATE TASK IF NOT EXISTS PIPELINE.REFRESH_MART_TASK
  WAREHOUSE = TRANSFORM_WH
  AFTER     PIPELINE.STAGE_ORDERS_TASK
AS
INSERT INTO MARTS.DAILY_ORDER_SUMMARY (order_date, total_orders, total_revenue, avg_order_value)
SELECT
    order_date,
    COUNT(*)               AS total_orders,
    SUM(total_amount)      AS total_revenue,
    AVG(total_amount)      AS avg_order_value
FROM STAGING.STG_ORDERS
GROUP BY order_date
ON CONFLICT (order_date) DO UPDATE
  SET total_orders    = EXCLUDED.total_orders,
      total_revenue   = EXCLUDED.total_revenue,
      avg_order_value = EXCLUDED.avg_order_value,
      refreshed_at    = CURRENT_TIMESTAMP();

-- ── RESUME IN ORDER: children first, root last ────────────────────────────
ALTER TASK PIPELINE.REFRESH_MART_TASK RESUME;
ALTER TASK PIPELINE.STAGE_ORDERS_TASK RESUME;
ALTER TASK PIPELINE.LOAD_ORDERS_TASK RESUME;
```

**Design decisions explained:**
- `APPEND ONLY = TRUE` on the stream because raw order rows are never updated — append-only is more efficient
- `WHEN SYSTEM$STREAM_HAS_DATA(...)` on the staging task prevents unnecessary warehouse startups when no new data arrived
- `ON_ERROR = 'SKIP_FILE_5%'` for orders — tolerates minor data issues without failing the whole hour's load
- Mart task has no `WHEN` condition — it always runs (even if staging had nothing new, the mart refresh is cheap)

---

## Mixed / Senior-Level Questions

**Q22 — Task failed, warehouse was suspended, stream has 24 hours of changes piled up. What do you do?**

**Step 1: Check if the stream is still valid (not stale).**
```sql
SHOW STREAMS IN SCHEMA RAW_DATA;
-- Check the "stale" column — if TRUE, you've lost the changes and need a full reload
```

**Step 2: If not stale, check what's in the stream.**
```sql
SELECT COUNT(*) FROM RAW_DATA.CUSTOMERS_STREAM;
-- How many changes are pending?
```

**Step 3: Resume the warehouse and manually execute the task.**
```sql
ALTER WAREHOUSE LOADING_WH RESUME;
EXECUTE TASK STAGING.PROCESS_CUSTOMER_CHANGES_TASK;
-- This runs the MERGE/INSERT immediately without waiting for the next schedule
```

**Step 4: Verify the task completed and the stream is consumed.**
```sql
SELECT * FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    TASK_NAME => 'PROCESS_CUSTOMER_CHANGES_TASK',
    SCHEDULED_TIME_RANGE_START => DATEADD('hours', -1, CURRENT_TIMESTAMP())
));
-- Check "state" column is "SUCCEEDED"

SELECT COUNT(*) FROM RAW_DATA.CUSTOMERS_STREAM;
-- Should be 0 if the task consumed the stream
```

**Step 5: Investigate root cause** — Why was the warehouse suspended? Add a Task failure notification (Snowflake Alerts or Airflow monitoring) so the team is alerted before the stream goes stale.

---

**Q23 — What is the difference between `EXECUTE TASK` and waiting for the scheduled run?**

| Approach | When it fires | Use case |
|---|---|---|
| `EXECUTE TASK task_name` | Immediately, on demand | Testing, manual recovery, backfill |
| Scheduled run | At the next scheduled time (interval or CRON) | Normal automated production runs |

```sql
EXECUTE TASK STAGING.LOAD_CUSTOMERS_TASK;
```

**When to use `EXECUTE TASK` in production:**
1. **After creating a new task** — test it immediately instead of waiting for the first scheduled run
2. **After a failure** — manually trigger a re-run after fixing the underlying issue
3. **Backfill** — if you need to process historical data that was not covered by scheduled runs
4. **On-demand trigger** — when an upstream event (a file arrival notification from Azure Event Grid) triggers your Snowflake pipeline outside the normal schedule

`EXECUTE TASK` respects the same `WHEN` condition as the scheduled run — if `SYSTEM$STREAM_HAS_DATA()` returns FALSE, the manual execution also skips.
