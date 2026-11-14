# Day 3 — Practice Exercises: Loading Data, Stages, and COPY INTO

> All exercises are done in Snowsight SQL worksheets unless noted otherwise.
> Complete them in order — each builds on the previous one.
> Estimated total time: 45–60 minutes.

---

## Exercise 1 — Create Stages and File Formats

**Concept:** Stage types, file format creation, LIST command

**Tasks:**

**1a. Set up your session context**
```sql
USE ROLE SYSADMIN;
USE WAREHOUSE LEARNING_WH;
USE DATABASE ECOMM_DB;
USE SCHEMA RAW_DATA;
```

**1b. Create a named internal stage**
```sql
CREATE STAGE IF NOT EXISTS ECOMM_INTERNAL_STAGE
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')
  DIRECTORY  = (ENABLE = TRUE)
  COMMENT    = 'Internal stage for Day 3 exercises';
```

**1c. Verify the stage was created**
```sql
SHOW STAGES IN SCHEMA ECOMM_DB.RAW_DATA;
-- You should see ECOMM_INTERNAL_STAGE in the results

LIST @ECOMM_DB.RAW_DATA.ECOMM_INTERNAL_STAGE;
-- Empty for now — no files uploaded yet
```

**1d. Create named file formats for all three file types**

```sql
-- CSV format
CREATE FILE FORMAT IF NOT EXISTS CSV_FORMAT
  TYPE                        = 'CSV'
  FIELD_DELIMITER             = ','
  RECORD_DELIMITER            = '\n'
  SKIP_HEADER                 = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  NULL_IF                     = ('NULL', 'null', 'N/A', '')
  EMPTY_FIELD_AS_NULL         = TRUE
  TRIM_SPACE                  = TRUE
  DATE_FORMAT                 = 'YYYY-MM-DD'
  COMPRESSION                 = 'AUTO'
  COMMENT                     = 'Standard CSV format';

-- JSON format
CREATE FILE FORMAT IF NOT EXISTS JSON_FORMAT
  TYPE              = 'JSON'
  STRIP_OUTER_ARRAY = TRUE
  STRIP_NULL_VALUES = FALSE
  COMPRESSION       = 'AUTO'
  COMMENT           = 'JSON format for event data';

-- Parquet format
CREATE FILE FORMAT IF NOT EXISTS PARQUET_FORMAT
  TYPE              = 'PARQUET'
  SNAPPY_COMPRESSION = TRUE
  COMMENT           = 'Parquet format for columnar files';
```

**1e. Verify all three file formats**
```sql
SHOW FILE FORMATS IN SCHEMA ECOMM_DB.RAW_DATA;
-- You should see CSV_FORMAT, JSON_FORMAT, PARQUET_FORMAT
```

**Expected outcome:** One internal stage and three file format objects created.

**Discussion:** Why create a JSON format with `STRIP_OUTER_ARRAY = TRUE`? What would happen if your JSON file had one object per line (NDJSON format) and you used that setting?

---

## Exercise 2 — Explore a Public Stage (No File Upload Required)

**Concept:** External stages, LIST, COPY INTO from public data

Snowflake provides a public S3 bucket with sample files. This lets us practice loading without needing to set up Azure credentials.

**Tasks:**

**2a. Create a stage pointing to Snowflake's public sample data bucket**
```sql
USE DATABASE ECOMM_DB;
USE SCHEMA RAW_DATA;

-- Public sample data hosted by Snowflake (no credentials needed — public bucket)
CREATE STAGE IF NOT EXISTS CITIBIKE_STAGE
  URL         = 's3://snowflake-workshop-lab/citibike-trips-csv/'
  FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 0 FIELD_OPTIONALLY_ENCLOSED_BY = '"');
```

**2b. List files in the public stage**
```sql
LIST @CITIBIKE_STAGE;
-- You should see many .csv.gz files
-- Note the file names and sizes
```

**2c. Preview data before loading using VALIDATION_MODE**
```sql
-- Create a simple target table for the citibike data
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

-- Preview first 10 rows WITHOUT loading any data
COPY INTO ECOMM_DB.RAW_DATA.CITIBIKE_TRIPS_RAW
FROM @CITIBIKE_STAGE
VALIDATION_MODE = 'RETURN_10_ROWS';
```

**Expected outcome:** 10 rows returned as a preview — no data inserted into the table yet.

**Discussion:** `SELECT COUNT(*) FROM CITIBIKE_TRIPS_RAW` should return 0 after validation mode. Why is this useful before a production load? What would you check in those 10 preview rows?

---

## Exercise 3 — Load Data with COPY INTO

**Concept:** COPY INTO, load history, idempotency, ON_ERROR

**Tasks:**

**3a. Load data from the public stage**
```sql
-- Load from the public Citibike stage — column mapping with METADATA$FILENAME
COPY INTO ECOMM_DB.RAW_DATA.CITIBIKE_TRIPS_RAW
  (tripduration, starttime, stoptime, start_station_id, start_station_name,
   start_lat, start_lon, end_station_id, end_station_name, end_lat, end_lon,
   bikeid, membership_type, birth_year, gender, _source_file)
FROM (
  SELECT
    $1::INTEGER, $2::TIMESTAMP_NTZ, $3::TIMESTAMP_NTZ,
    $4::INTEGER, $5::VARCHAR,
    $6::FLOAT,   $7::FLOAT,
    $8::INTEGER, $9::VARCHAR,
    $10::FLOAT,  $11::FLOAT,
    $12::INTEGER,$13::VARCHAR,
    $14::INTEGER,$15::INTEGER,
    METADATA$FILENAME
  FROM @CITIBIKE_STAGE
)
FILE_FORMAT = (TYPE='CSV' SKIP_HEADER=0 FIELD_OPTIONALLY_ENCLOSED_BY='"')
ON_ERROR    = 'CONTINUE'
PATTERN     = '.*trips_2013.*\.csv\.gz';   -- load just 2013 files to keep it small
```

**3b. Check how many rows loaded**
```sql
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.CITIBIKE_TRIPS_RAW;
```

**3c. Check which files were loaded and their row counts**
```sql
SELECT *
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'CITIBIKE_TRIPS_RAW',
    START_TIME => DATEADD('hours', -1, CURRENT_TIMESTAMP())
))
ORDER BY last_load_time DESC;
```
- How many files were loaded?
- Were any rows skipped (`rows_parsed` ≠ `rows_loaded`)?

**3d. Test idempotency — run the same COPY INTO again**
```sql
-- Run the exact same COPY INTO from 3a again
-- How many rows are loaded this time?
```

**3e. Check source file distribution**
```sql
-- Which file did each row come from?
SELECT _source_file, COUNT(*) AS row_count
FROM ECOMM_DB.RAW_DATA.CITIBIKE_TRIPS_RAW
GROUP BY _source_file
ORDER BY row_count DESC;
```

**Expected outcome:** Data loaded from 2013 files. Second COPY INTO loads 0 rows. Source file column shows file names.

**Discussion:** You used `ON_ERROR = 'CONTINUE'` for the Citibike data. Would this be appropriate for a financial transactions table? What setting would you use instead and why?
