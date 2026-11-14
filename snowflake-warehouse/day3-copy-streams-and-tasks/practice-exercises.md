# Day 3 — Practice Exercises: COPY INTO, Streams, and Tasks

> All exercises run in Snowsight SQL worksheets — no local tools or file uploads needed.
> Complete them in order — each builds on the previous one.
> Estimated total time: 60–80 minutes.

---

## Exercise 1 — COPY INTO from a Public Stage

**Concept:** Bulk loading, file formats, VALIDATION_MODE, load history

Snowflake hosts a public S3 bucket with sample data. We use it so you can practice COPY INTO without setting up any cloud credentials.

**1a. Set up your session**
```sql
USE ROLE SYSADMIN;
USE WAREHOUSE COMPUTE_WH;
USE DATABASE ECOMM_DB;
USE SCHEMA RAW_DATA;
```

**1b. Create a stage pointing to Snowflake's public Citibike dataset**
```sql
CREATE STAGE IF NOT EXISTS CITIBIKE_STAGE
  URL         = 's3://snowflake-workshop-lab/citibike-trips-csv/'
  FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 0 FIELD_OPTIONALLY_ENCLOSED_BY = '"');
```

**1c. See what files are in the stage**
```sql
LIST @CITIBIKE_STAGE;
-- Notice: many .csv.gz files. These are real Citibike NYC trip records.
```

**1d. Create the target table**
```sql
CREATE TABLE IF NOT EXISTS CITIBIKE_TRIPS_RAW (
    tripduration        INTEGER,
    starttime           TIMESTAMP_NTZ,
    stoptime            TIMESTAMP_NTZ,
    start_station_id    INTEGER,
    start_station_name  VARCHAR(255),
    start_lat           FLOAT,
    start_lon           FLOAT,
    end_station_id      INTEGER,
    end_station_name    VARCHAR(255),
    end_lat             FLOAT,
    end_lon             FLOAT,
    bikeid              INTEGER,
    membership_type     VARCHAR(50),
    birth_year          INTEGER,
    gender              INTEGER,
    _source_file        VARCHAR(500)
);
```

**1e. Dry run first — preview without loading**
```sql
COPY INTO ECOMM_DB.RAW_DATA.CITIBIKE_TRIPS_RAW
FROM @CITIBIKE_STAGE
VALIDATION_MODE = 'RETURN_10_ROWS';
-- 10 rows returned as a preview. Check the data looks right.
-- Run this after: SELECT COUNT(*) FROM CITIBIKE_TRIPS_RAW;
-- It should return 0 — VALIDATION_MODE loads nothing.
```

**1f. Load data from 2013 files only**
```sql
COPY INTO ECOMM_DB.RAW_DATA.CITIBIKE_TRIPS_RAW
  (tripduration, starttime, stoptime,
   start_station_id, start_station_name, start_lat, start_lon,
   end_station_id, end_station_name, end_lat, end_lon,
   bikeid, membership_type, birth_year, gender, _source_file)
FROM (
  SELECT
    $1::INTEGER, $2::TIMESTAMP_NTZ, $3::TIMESTAMP_NTZ,
    $4::INTEGER, $5::VARCHAR, $6::FLOAT, $7::FLOAT,
    $8::INTEGER, $9::VARCHAR, $10::FLOAT, $11::FLOAT,
    $12::INTEGER, $13::VARCHAR, $14::INTEGER, $15::INTEGER,
    METADATA$FILENAME
  FROM @CITIBIKE_STAGE
)
FILE_FORMAT = (TYPE='CSV' SKIP_HEADER=0 FIELD_OPTIONALLY_ENCLOSED_BY='"')
ON_ERROR    = 'CONTINUE'
PATTERN     = '.*trips_2013.*\.csv\.gz';
```

**1g. Check the load**
```sql
-- How many rows loaded?
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.CITIBIKE_TRIPS_RAW;

-- Which files were loaded?
SELECT _source_file, COUNT(*) AS row_count
FROM ECOMM_DB.RAW_DATA.CITIBIKE_TRIPS_RAW
GROUP BY _source_file ORDER BY row_count DESC;

-- Full load history with status
SELECT file_name, status, rows_parsed, rows_loaded,
       rows_parsed - rows_loaded AS rows_skipped,
       first_error_message
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'CITIBIKE_TRIPS_RAW',
    START_TIME => DATEADD('hours', -1, CURRENT_TIMESTAMP())
))
ORDER BY last_load_time DESC;
```

**1h. Test idempotency**
```sql
-- Run the exact same COPY INTO from 1f again.
-- How many rows are loaded the second time?
-- Check COPY_HISTORY again — what does the status say?
```

**Expected outcome:** Data loaded from 2013 files. Second run loads 0 rows. Load history shows status `LOADED` for first run and `LOAD_SKIPPED` for second.

**Discussion:** You used `ON_ERROR = 'CONTINUE'` for Citibike. If this were a table of financial transactions, what setting would you use instead and why?

---

## Exercise 2 — Streams: Capturing Changes (CDC)

**Concept:** Creating a stream, making changes, reading CDC metadata

**2a. Create a simple customers table to work with**
```sql
USE DATABASE ECOMM_DB;
USE SCHEMA RAW_DATA;

CREATE TABLE IF NOT EXISTS CUSTOMERS_RAW (
    customer_id   NUMBER       PRIMARY KEY,
    full_name     VARCHAR(200),
    email         VARCHAR(200),
    city          VARCHAR(100),
    status        VARCHAR(50)  DEFAULT 'ACTIVE'
);
```

**2b. Create a stream on the table**
```sql
CREATE STREAM IF NOT EXISTS CUSTOMERS_STREAM
  ON TABLE ECOMM_DB.RAW_DATA.CUSTOMERS_RAW
  COMMENT = 'CDC stream — tracks all changes to CUSTOMERS_RAW';
```

**2c. Insert some initial rows**
```sql
INSERT INTO CUSTOMERS_RAW (customer_id, full_name, email, city, status) VALUES
  (1, 'Alice Johnson',  'alice@example.com',  'New York',   'ACTIVE'),
  (2, 'Bob Smith',      'bob@example.com',    'Chicago',    'ACTIVE'),
  (3, 'Priya Sharma',   'priya@example.com',  'Austin',     'ACTIVE'),
  (4, 'Carlos Mendez',  'carlos@example.com', 'Miami',      'ACTIVE'),
  (5, 'Yuki Tanaka',    'yuki@example.com',   'Seattle',    'ACTIVE');
```

**2d. See what the stream captured**
```sql
SELECT
    customer_id,
    full_name,
    email,
    city,
    METADATA$ACTION,
    METADATA$ISUPDATE,
    METADATA$ROW_ID
FROM ECOMM_DB.RAW_DATA.CUSTOMERS_STREAM;
-- 5 rows, all METADATA$ACTION = 'INSERT', METADATA$ISUPDATE = FALSE
```

**2e. Now make some changes — update and delete**
```sql
-- Update Bob's city
UPDATE CUSTOMERS_RAW SET city = 'Los Angeles' WHERE customer_id = 2;

-- Delete Carlos
DELETE FROM CUSTOMERS_RAW WHERE customer_id = 4;

-- Insert a new customer
INSERT INTO CUSTOMERS_RAW VALUES (6, 'Sophie Dubois', 'sophie@example.com', 'Denver', 'ACTIVE');
```

**2f. Query the stream — observe how UPDATE is represented**
```sql
SELECT
    customer_id,
    full_name,
    city,
    METADATA$ACTION,
    METADATA$ISUPDATE
FROM ECOMM_DB.RAW_DATA.CUSTOMERS_STREAM
ORDER BY customer_id, METADATA$ACTION;
```

Questions to answer from the results:
- How many rows does the UPDATE to Bob (customer_id=2) produce? What are their METADATA$ACTION values?
- How is the DELETE of Carlos (customer_id=4) represented?
- How is the new INSERT of Sophie (customer_id=6) represented?

**2g. Filter stream by change type**
```sql
-- Only new inserts (not update-related)
SELECT customer_id, full_name, city, 'NEW' AS change_type
FROM ECOMM_DB.RAW_DATA.CUSTOMERS_STREAM
WHERE METADATA$ACTION = 'INSERT' AND METADATA$ISUPDATE = FALSE;

-- Only updates — the new values
SELECT customer_id, full_name, city, 'UPDATED' AS change_type
FROM ECOMM_DB.RAW_DATA.CUSTOMERS_STREAM
WHERE METADATA$ACTION = 'INSERT' AND METADATA$ISUPDATE = TRUE;

-- Only deletes
SELECT customer_id, full_name, city, 'DELETED' AS change_type
FROM ECOMM_DB.RAW_DATA.CUSTOMERS_STREAM
WHERE METADATA$ACTION = 'DELETE' AND METADATA$ISUPDATE = FALSE;
```

**2h. Consume the stream — apply changes to a staging table**
```sql
USE SCHEMA ECOMM_DB.STAGING;

CREATE TABLE IF NOT EXISTS STG_CUSTOMERS (
    customer_id   NUMBER       PRIMARY KEY,
    full_name     VARCHAR(200),
    email         VARCHAR(200),
    city          VARCHAR(100),
    status        VARCHAR(50),
    updated_at    TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Apply stream changes using MERGE
MERGE INTO ECOMM_DB.STAGING.STG_CUSTOMERS AS target
USING (
  SELECT * FROM ECOMM_DB.RAW_DATA.CUSTOMERS_STREAM
  WHERE METADATA$ACTION = 'INSERT'
) AS src
ON target.customer_id = src.customer_id
WHEN MATCHED THEN
  UPDATE SET
    target.full_name  = src.full_name,
    target.email      = src.email,
    target.city       = src.city,
    target.status     = src.status,
    target.updated_at = CURRENT_TIMESTAMP()
WHEN NOT MATCHED THEN
  INSERT (customer_id, full_name, email, city, status, updated_at)
  VALUES (src.customer_id, src.full_name, src.email, src.city, src.status, CURRENT_TIMESTAMP());
```

**2i. Check the stream after consuming**
```sql
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.CUSTOMERS_STREAM;
-- Should be 0 — the MERGE consumed the stream, offset advanced
```

**2j. Verify the staging table**
```sql
SELECT * FROM ECOMM_DB.STAGING.STG_CUSTOMERS ORDER BY customer_id;
-- Alice, Bob (with Los Angeles), Priya, Sophie should be here
-- Carlos should NOT be here (he was deleted from raw, and the MERGE only processed INSERTs)
```

**Discussion:** We only applied `METADATA$ACTION = 'INSERT'` rows in the MERGE. How would you also handle deletes — i.e., remove deleted customers from the staging table? Add a `WHEN MATCHED AND src.METADATA$ACTION = 'DELETE' THEN DELETE` clause.

---

## Exercise 3 — Tasks: Automating the Pipeline

**Concept:** Creating tasks, scheduling, WHEN condition, chaining

**3a. Create a task that automatically processes stream changes**
```sql
USE ROLE SYSADMIN;
USE DATABASE ECOMM_DB;
USE SCHEMA STAGING;

CREATE TASK IF NOT EXISTS ECOMM_DB.STAGING.PROCESS_CUSTOMER_CHANGES_TASK
  WAREHOUSE = COMPUTE_WH
  SCHEDULE  = '1 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('ECOMM_DB.RAW_DATA.CUSTOMERS_STREAM')
AS
MERGE INTO ECOMM_DB.STAGING.STG_CUSTOMERS AS target
USING (
  SELECT * FROM ECOMM_DB.RAW_DATA.CUSTOMERS_STREAM
  WHERE METADATA$ACTION = 'INSERT'
) AS src
ON target.customer_id = src.customer_id
WHEN MATCHED THEN
  UPDATE SET
    target.full_name  = src.full_name,
    target.email      = src.email,
    target.city       = src.city,
    target.status     = src.status,
    target.updated_at = CURRENT_TIMESTAMP()
WHEN NOT MATCHED THEN
  INSERT (customer_id, full_name, email, city, status, updated_at)
  VALUES (src.customer_id, src.full_name, src.email, src.city, src.status, CURRENT_TIMESTAMP());
```

**3b. Test the task manually before activating the schedule**
```sql
-- First make some new changes to generate stream data
INSERT INTO ECOMM_DB.RAW_DATA.CUSTOMERS_RAW VALUES
  (7, 'Marco Rossi',   'marco@example.com',  'Boston',  'ACTIVE'),
  (8, 'Aisha Patel',   'aisha@example.com',  'Phoenix', 'ACTIVE');

UPDATE ECOMM_DB.RAW_DATA.CUSTOMERS_RAW
SET status = 'VIP' WHERE customer_id = 1;

-- Check the stream has data
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.CUSTOMERS_STREAM;

-- Execute the task immediately (without waiting for the 1-minute schedule)
EXECUTE TASK ECOMM_DB.STAGING.PROCESS_CUSTOMER_CHANGES_TASK;
```

**3c. Check the task run result**
```sql
SELECT
    name,
    state,
    scheduled_time,
    completed_time,
    error_message,
    DATEDIFF('seconds', scheduled_time, completed_time) AS duration_seconds
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    SCHEDULED_TIME_RANGE_START => DATEADD('minutes', -5, CURRENT_TIMESTAMP()),
    TASK_NAME => 'PROCESS_CUSTOMER_CHANGES_TASK'
))
ORDER BY scheduled_time DESC;
-- state should be "SUCCEEDED"
```

**3d. Verify the staging table was updated**
```sql
SELECT * FROM ECOMM_DB.STAGING.STG_CUSTOMERS ORDER BY customer_id;
-- Marco and Aisha should now appear
-- Alice should show status = 'VIP'
```

**3e. Activate the schedule**
```sql
ALTER TASK ECOMM_DB.STAGING.PROCESS_CUSTOMER_CHANGES_TASK RESUME;

-- Verify it's running
SHOW TASKS IN SCHEMA ECOMM_DB.STAGING;
-- Look at the "state" column — it should say "started"
```

**3f. Test the WHEN condition — insert data, then wait**
```sql
-- Add one more customer
INSERT INTO ECOMM_DB.RAW_DATA.CUSTOMERS_RAW VALUES
  (9, 'Lena Müller', 'lena@example.com', 'Berlin', 'ACTIVE');

-- Wait about 1–2 minutes, then check task history
SELECT name, state, scheduled_time, completed_time
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    SCHEDULED_TIME_RANGE_START => DATEADD('minutes', -5, CURRENT_TIMESTAMP()),
    TASK_NAME => 'PROCESS_CUSTOMER_CHANGES_TASK'
))
ORDER BY scheduled_time DESC;

-- And verify Lena appears in staging
SELECT * FROM ECOMM_DB.STAGING.STG_CUSTOMERS WHERE customer_id = 9;
```

**3g. Suspend the task when done**
```sql
ALTER TASK ECOMM_DB.STAGING.PROCESS_CUSTOMER_CHANGES_TASK SUSPEND;
-- Always suspend tasks after exercises to avoid unexpected credit usage
```

**Expected outcome:** Task runs automatically on schedule, picks up new changes from the stream, and applies them to staging. When no changes exist, the WHEN condition prevents execution.

**Discussion:** This task runs every minute. In a real production pipeline where data arrives once an hour, running every minute wastes warehouse credits 59 out of 60 minutes. What two changes would you make to optimize this?

---

## Bonus — Build a Chained Task Pipeline

**Concept:** Task DAGs, root + child tasks, ordered dependencies

Build a 3-step pipeline entirely in Snowflake:

```sql
-- Step 1: Create a mart table
CREATE TABLE IF NOT EXISTS ECOMM_DB.MARTS.CUSTOMER_CITY_SUMMARY (
    city          VARCHAR(100),
    customer_count NUMBER,
    vip_count     NUMBER,
    refreshed_at  TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Step 2: Root task — simulates a load (we insert directly for the exercise)
CREATE TASK IF NOT EXISTS ECOMM_DB.RAW_DATA.SIMULATE_LOAD_TASK
  WAREHOUSE = COMPUTE_WH
  SCHEDULE  = 'USING CRON */2 * * * * UTC'   -- every 2 minutes
AS
INSERT INTO ECOMM_DB.RAW_DATA.CUSTOMERS_RAW (customer_id, full_name, email, city, status)
VALUES (
    (SELECT COALESCE(MAX(customer_id), 0) + 1 FROM ECOMM_DB.RAW_DATA.CUSTOMERS_RAW),
    'Test User ' || TO_VARCHAR(CURRENT_TIMESTAMP()),
    'test@example.com',
    CASE MOD(UNIFORM(1,4,RANDOM()), 4)
      WHEN 0 THEN 'New York'
      WHEN 1 THEN 'Chicago'
      WHEN 2 THEN 'Austin'
      ELSE 'Denver'
    END,
    'ACTIVE'
);

-- Step 3: Child task — consume stream into staging
CREATE TASK IF NOT EXISTS ECOMM_DB.STAGING.SYNC_STAGING_TASK
  WAREHOUSE = COMPUTE_WH
  AFTER ECOMM_DB.RAW_DATA.SIMULATE_LOAD_TASK
  WHEN SYSTEM$STREAM_HAS_DATA('ECOMM_DB.RAW_DATA.CUSTOMERS_STREAM')
AS
MERGE INTO ECOMM_DB.STAGING.STG_CUSTOMERS AS target
USING (SELECT * FROM ECOMM_DB.RAW_DATA.CUSTOMERS_STREAM WHERE METADATA$ACTION = 'INSERT') AS src
ON target.customer_id = src.customer_id
WHEN NOT MATCHED THEN
  INSERT (customer_id, full_name, email, city, status, updated_at)
  VALUES (src.customer_id, src.full_name, src.email, src.city, src.status, CURRENT_TIMESTAMP());

-- Step 4: Grandchild task — refresh the mart
CREATE TASK IF NOT EXISTS ECOMM_DB.MARTS.REFRESH_CITY_SUMMARY_TASK
  WAREHOUSE = COMPUTE_WH
  AFTER ECOMM_DB.STAGING.SYNC_STAGING_TASK
AS
INSERT INTO ECOMM_DB.MARTS.CUSTOMER_CITY_SUMMARY (city, customer_count, vip_count)
SELECT city, COUNT(*) AS customer_count, SUM(CASE WHEN status = 'VIP' THEN 1 ELSE 0 END) AS vip_count
FROM ECOMM_DB.STAGING.STG_CUSTOMERS
GROUP BY city;

-- Resume in correct order: children first, root last
ALTER TASK ECOMM_DB.MARTS.REFRESH_CITY_SUMMARY_TASK RESUME;
ALTER TASK ECOMM_DB.STAGING.SYNC_STAGING_TASK RESUME;
ALTER TASK ECOMM_DB.RAW_DATA.SIMULATE_LOAD_TASK RESUME;

-- Wait 2–3 minutes, then check
SELECT * FROM ECOMM_DB.MARTS.CUSTOMER_CITY_SUMMARY ORDER BY customer_count DESC;

-- Check the pipeline ran end-to-end
SELECT name, state, scheduled_time, completed_time
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    SCHEDULED_TIME_RANGE_START => DATEADD('minutes', -10, CURRENT_TIMESTAMP())
))
ORDER BY scheduled_time DESC;

-- Suspend everything when done
ALTER TASK ECOMM_DB.RAW_DATA.SIMULATE_LOAD_TASK SUSPEND;
ALTER TASK ECOMM_DB.STAGING.SYNC_STAGING_TASK SUSPEND;
ALTER TASK ECOMM_DB.MARTS.REFRESH_CITY_SUMMARY_TASK SUSPEND;
```

**Design question:** The mart refresh uses `INSERT INTO` every run, which will keep adding duplicate date summaries over time. What Snowflake feature would you use to make the mart table always show the latest counts without duplicates?
