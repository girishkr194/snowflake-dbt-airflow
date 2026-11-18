# Day 5 — Interview Solutions: Stages, COPY INTO, Snowpipe, and Zero-Copy Cloning

> Complete answers. Read after attempting on your own.

---

**Q1 — What is a Stage? Why do you need one before loading?**

A **Stage** is a Snowflake object that points to a file storage location. It acts as a landing zone — files must arrive in a stage before Snowflake can load them into a table with `COPY INTO` or Snowpipe.

Snowflake cannot read files directly from your laptop or an arbitrary file path. Everything goes through a stage because:
- It gives Snowflake a controlled access point with defined file format settings.
- It enables deduplication — Snowflake tracks which files from a stage have already been loaded.
- It decouples the file location (cloud storage, internal storage) from the loading logic.

```
Your files  ──►  Stage  ──►  COPY INTO / Snowpipe  ──►  Table
```

---

**Q2 — Internal stage vs external stage.**

| | Internal Stage | External Stage |
|---|---|---|
| Storage | Snowflake-managed (you don't need a cloud bucket) | Your own S3 / Azure Blob / GCS |
| Credentials | None needed | Requires Storage Integration or access keys |
| File upload | `PUT` command or Snowsight UI | Drop files directly into your cloud bucket |
| Data ownership | Snowflake owns the storage | You own the storage |
| Best for | Testing, demos, pipelines without existing cloud storage | Production pipelines where files land in your cloud bucket |

---

**Q3 — Three types of internal stage: user, table, named. Which for production?**

| Type | Reference | Auto-created? | Scope |
|---|---|---|---|
| User | `@~` | Yes (one per user) | Private to the current user |
| Table | `@%table_name` | Yes (one per table) | Tied to one specific table |
| Named | `@schema.stage_name` | No — you create it | Shared, reusable across tables and users |

**Use named stage in production.** Reasons:
- Shared across multiple users and pipelines — one stage can feed multiple tables.
- File format is defined on the stage, not repeated in every COPY INTO command.
- Access controlled via Snowflake RBAC (you can GRANT USAGE to specific roles).
- User stages are private — no one else can use them. Table stages are locked to one table — you cannot load the same file into two different tables from a table stage.

---

**Q4 — What does COPY INTO do? What compute does it use?**

`COPY INTO` reads files from a stage and bulk-loads them into a Snowflake table. It is a **manual, batch command** — you run it when you want to load.

```sql
COPY INTO my_table FROM @my_stage FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1);
```

It uses your **virtual warehouse** — the XSMALL / SMALL / MEDIUM compute cluster you have running in your session. You are billed warehouse credits for the duration of the COPY INTO run.

---

**Q5 — Run COPY INTO on 10 already-loaded files. What happens?**

All 10 files are **skipped** — status shows `LOAD_SKIPPED` for each.

COPY INTO has a built-in **64-day deduplication window**. After loading a file, Snowflake records its metadata (file name, size, last-modified timestamp) in an internal tracking table. When you run COPY INTO again on the same stage, it checks this metadata first. Files already in the tracking table within the last 64 days are skipped — no rows are inserted, no error is raised.

This is exactly the same mechanism Snowpipe uses. It prevents accidental duplicates when you re-run COPY INTO after adding new files to a shared stage.

---

**Q6 — `FORCE = TRUE` in COPY INTO. Risk and when to use.**

`FORCE = TRUE` bypasses the deduplication metadata and loads the file regardless of whether it was already loaded.

**Risk:** If the file was already loaded, every row in that file is inserted **again** — creating exact duplicate rows in your target table. There is no automatic rollback. If you used `FORCE = TRUE` on 50 files that were already loaded, you now have double the data.

**When acceptable:**
- **Testing** — you need to load a file multiple times to test your pipeline logic.
- **Intentional reload** — a file had bad data, you fixed it, and you want to reload the corrected version. You must `DELETE` or `TRUNCATE` the previously loaded rows first, then use `FORCE = TRUE` to reload.
- Never use `FORCE = TRUE` in a scheduled production pipeline.

---

**Q7 — Load specific columns from CSV and add a `loaded_at` column.**

Use an inline SELECT in the COPY INTO `FROM` clause. Column positions in a CSV use `$1`, `$2`, `$3`, etc.:

```sql
CREATE OR REPLACE TABLE ORDERS_PARTIAL (
    order_id      NUMBER,
    customer_name VARCHAR(200),
    total_amount  NUMBER(10,2),
    status        VARCHAR(50),
    loaded_at     TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

COPY INTO ORDERS_PARTIAL (order_id, customer_name, total_amount, status)
FROM (
    SELECT
        $1,                      -- order_id      (column 1 in CSV)
        $2,                      -- customer_name (column 2)
        $6,                      -- total_amount  (column 6)
        $7                       -- status        (column 7)
    FROM @ECOMM_DB.RAW_DATA.ORDERS_STAGE
)
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1);
-- loaded_at gets its DEFAULT value (CURRENT_TIMESTAMP) automatically
```

You can also apply transformations: `CAST($6 AS NUMBER(12,2))`, `UPPER($7)`, `TRY_CAST(...)` for null-safe casting.

---

**Q8 — What is Snowpipe? How is it different from a 1-minute Task + COPY INTO?**

Snowpipe is Snowflake's **automatic, continuous file ingestion** service. You create a Pipe object with a COPY INTO statement, and Snowpipe runs it automatically whenever new files arrive — no scheduler needed.

| | Task + COPY INTO | Snowpipe |
|---|---|---|
| Trigger | Fixed schedule (min 1 min) | File arrival or REST API call |
| Compute | Your virtual warehouse | Snowflake-managed (serverless) |
| Latency | Up to 1 minute + warehouse startup | Seconds to ~1 minute |
| Idle cost | Warehouse may bill while idle | Zero — no files = no compute |
| Duplicate prevention | None built-in | Built-in (file metadata tracking) |
| Cost model | Warehouse credits | Per-second of serverless compute |

**Key point:** A Task polls every minute regardless of whether files exist. Snowpipe only runs when triggered and uses managed compute that you never provision or size.

---

**Q9 — Snowpipe is "serverless." What does that mean for compute and billing?**

Snowflake maintains a shared pool of pre-warmed compute nodes internally. When Snowpipe loads files, it uses nodes from this pool — you do not spin up a virtual warehouse. There is no XSMALL / SMALL / MEDIUM to choose.

**Billing:** You are charged **Snowpipe credits** (separate from warehouse credits) based on actual compute seconds used. If no files arrive for 8 hours, you pay zero — there is nothing running.

**Practical implication:** You cannot tune the warehouse size for Snowpipe. For extremely high-throughput loading (hundreds of thousands of rows per second), a dedicated virtual warehouse + COPY INTO may outperform Snowpipe because you can explicitly scale the warehouse. For typical file-by-file streaming (files every few seconds to minutes), Snowpipe is cheaper and simpler.

---

**Q10 — Same file is in the stage but Snowpipe does not load it again after REFRESH. Why?**

Snowpipe uses the same **deduplication metadata** as COPY INTO. After loading a file, it records the file's name, size, and last-modified timestamp internally. When `REFRESH` is triggered, Snowpipe scans the stage but skips any file already in its metadata.

This is intentional and correct — without it, every `REFRESH` call would reload all files in the stage, creating duplicates.

**To force a reload:**
- Drop and recreate the pipe (clears all metadata), then trigger `REFRESH`.
- Or, upload the file again with a **new filename** — Snowpipe treats it as a different file.

---

**Q11 — Pipe shows PAUSED and 20 files were not loaded. How to recover without data loss?**

```sql
-- Step 1: Resume the pipe
ALTER PIPE ECOMM_DB.RAW_DATA.ORDERS_PIPE SET PIPE_EXECUTION_PAUSED = FALSE;

-- Step 2: Force a scan to pick up the files that arrived while paused
-- REFRESH scans the stage and queues any files not yet loaded by this pipe
ALTER PIPE ECOMM_DB.RAW_DATA.ORDERS_PIPE REFRESH;
-- Returns: filesScanned = X, filesQueued = 20

-- Step 3: Wait ~20-30 seconds for serverless compute to process them
-- Then verify
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_PIPE_LOAD;

-- Check all 20 files loaded successfully
SELECT file_name, row_count, status
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME  => 'ORDERS_PIPE_LOAD',
    START_TIME  => DATEADD('hours', -1, CURRENT_TIMESTAMP())
))
ORDER BY LAST_LOAD_TIME DESC;
```

**Why no data loss:** The 20 files are still in the stage — they were not consumed or deleted when the pipe was paused. `REFRESH` re-scans the stage and picks up any file not yet in the pipe's metadata.

---

**Q12 — What is zero-copy cloning? What does "zero-copy" mean technically?**

Zero-copy cloning creates a new Snowflake object that **shares the same underlying micro-partitions** as the source. No bytes are copied to new storage at clone time. Snowflake just creates a new metadata entry pointing to the same physical data blocks.

**"Zero-copy"** = zero byte duplication at clone time. Both objects point to the same physical blocks.

**Copy-on-Write semantics:** When either the source OR the clone is modified (INSERT, UPDATE, DELETE), Snowflake creates new micro-partitions only for the changed data. The unmodified partitions remain shared.

```
After CREATE TABLE dev CLONE prod:
  prod ──► [Block A] [Block B] [Block C]   ← same blocks
  dev  ──► [Block A] [Block B] [Block C]   ← zero extra storage

After deleting old rows from dev:
  prod ──► [Block A] [Block B] [Block C]   ← untouched
  dev  ──► [Block A*] [Block B] [Block C]  ← only Block A* is new storage
```

---

**Q13 — 5 TB table cloned. Developer deletes 1 million rows from the clone. How much new storage?**

Only the **micro-partitions that contain the deleted rows** are rewritten — not the whole table.

Snowflake stores data in immutable micro-partitions (~16MB to ~512MB compressed). A `DELETE` does not modify partitions in place — it marks old partitions as deleted and writes new ones containing the surviving rows.

**What actually happens:**
- Micro-partitions with **only** deleted rows → marked deleted in clone's metadata. No new partition written, but the old partition is no longer shared — it is eventually removed when the clone's Time Travel window expires.
- Micro-partitions with a **mix** of deleted and surviving rows → rewritten. New partition = surviving rows only. This is new storage.
- Micro-partitions with **no deleted rows** → still shared between prod and clone. Zero new storage.

**In practice:** If the data is ordered by date (Snowflake's natural clustering), old-date rows tend to live in their own partitions. The delete creates minimal new partitions. The original prod table is **completely unaffected** — it still points to all its original blocks.

---

**Q14 — Clone a table that has a Stream. What happens to the stream?**

You can clone the table — it clones normally.

**The Stream is not cloned.** Streams are excluded when cloning a table, schema, or database. The original stream on the source table continues working normally — the clone operation does not consume or advance its offset.

The cloned table has no stream after cloning. If you need a stream on the clone, create one manually:

```sql
CREATE STREAM ECOMM_DB.STAGING_BACKUP.STG_ORDERS_STREAM
    ON TABLE ECOMM_DB.STAGING_BACKUP.STG_ORDERS;
```

**Why streams are not cloned:** A stream tracks changes from its current offset forward. Automatically cloning a stream would create ambiguity — should the clone replay from the same offset (potentially consuming the same changes twice) or start fresh? Snowflake avoids the ambiguity by simply not cloning streams.

---

**Q15 — Recover 3 million accidentally deleted rows using cloning + Time Travel.**

```sql
-- Step 1: Find the query ID of the bad DELETE
SELECT query_id, query_text, start_time
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY(
    DATE_RANGE_START => DATEADD('hours', -12, CURRENT_TIMESTAMP())
))
WHERE query_text ILIKE '%DELETE%STG_ORDERS%'
ORDER BY start_time DESC
LIMIT 1;
-- Copy the query_id from this result

-- Step 2: Clone the table as it was BEFORE that exact query
CREATE TABLE ECOMM_DB.STAGING.STG_ORDERS_RECOVERED
    CLONE ECOMM_DB.STAGING.STG_ORDERS
    BEFORE (STATEMENT => '<paste-query-id-here>');
-- This creates an instant snapshot of the table's state just before the DELETE ran

-- Step 3: Verify the recovered table has the full count
SELECT COUNT(*) FROM ECOMM_DB.STAGING.STG_ORDERS_RECOVERED;
-- Should be 3 million rows higher than the current STG_ORDERS

-- Step 4: Re-insert the missing rows
INSERT INTO ECOMM_DB.STAGING.STG_ORDERS
SELECT * FROM ECOMM_DB.STAGING.STG_ORDERS_RECOVERED r
WHERE NOT EXISTS (
    SELECT 1 FROM ECOMM_DB.STAGING.STG_ORDERS s
    WHERE s.order_id = r.order_id
);

-- Step 5: Confirm full count is restored
SELECT COUNT(*) FROM ECOMM_DB.STAGING.STG_ORDERS;

-- Step 6: Drop the recovery table
DROP TABLE ECOMM_DB.STAGING.STG_ORDERS_RECOVERED;
```

**Alternative — full table swap (if the live table has no good rows to preserve):**
```sql
ALTER TABLE ECOMM_DB.STAGING.STG_ORDERS          RENAME TO STG_ORDERS_BAD;
ALTER TABLE ECOMM_DB.STAGING.STG_ORDERS_RECOVERED RENAME TO STG_ORDERS;
DROP TABLE ECOMM_DB.STAGING.STG_ORDERS_BAD;
```
