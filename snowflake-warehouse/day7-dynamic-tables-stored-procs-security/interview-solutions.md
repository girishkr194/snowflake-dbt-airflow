# Day 7 — Interview Solutions: Dynamic Tables, Materialized Views, Stored Procedures & UDFs, Security Policies

> Complete answers. Read after attempting on your own.

---

**Q1 — What is a Dynamic Table? What is TARGET_LAG?**

A **Dynamic Table** is a table whose contents Snowflake automatically keeps in sync with a source query. You write a `SELECT` statement — Snowflake stores the result as a physical table and refreshes it on a schedule.

**TARGET_LAG** defines the maximum acceptable staleness — how far behind the Dynamic Table is allowed to be relative to its source data.

```sql
CREATE DYNAMIC TABLE STG_ORDERS_DT
    TARGET_LAG = '1 minute'   -- at most 1 minute out of date
    WAREHOUSE  = COMPUTE_WH
AS SELECT * FROM ORDERS_RAW;
```

Snowflake aims to refresh at most `TARGET_LAG` behind the source. If the source changes at 10:00:00, the DT is guaranteed to reflect that by 10:01:00. Setting a shorter lag (e.g., `'10 seconds'`) means more frequent refreshes and higher compute cost.

---

**Q2 — When to replace stream + task with a Dynamic Table? When to keep stream + task?**

**Replace with Dynamic Table when:**
- The pipeline is a straightforward transformation (SELECT, filter, aggregate, join)
- You want Snowflake to manage the refresh logic — no MERGE branches to write
- You need sub-minute lag (stream + task minimum is 1 minute)
- You are building a new pipeline from scratch

**Keep stream + task when:**
- You need **soft deletes** — marking `is_deleted = TRUE` instead of physically removing rows. Dynamic Tables reflect the source exactly; a hard DELETE from the source removes the row from the DT too.
- You need **custom MERGE logic** — e.g., the three-branch MERGE for INSERT/UPDATE/DELETE with different business rules per branch
- The pipeline already works and re-writing it has no clear benefit
- You need the WHEN condition (`SYSTEM$STREAM_HAS_DATA`) to avoid running when nothing changed

---

**Q3 — Chain: ORDERS_RAW → STG_ORDERS_DT (1 min) → ORDERS_MART_DT (5 min). Total lag?**

**Up to 6 minutes** in the worst case.

Lag is additive in a chain:
- A change hits `ORDERS_RAW` at T=0
- `STG_ORDERS_DT` (1-min lag) reflects it by T=1 minute
- `ORDERS_MART_DT` (5-min lag) reads from `STG_ORDERS_DT` — its 5-minute clock starts from when `STG_ORDERS_DT` was last refreshed, not from T=0

**Worst case:** The change lands in `STG_ORDERS_DT` at T=59 seconds (just before the 1-min mark). Then `ORDERS_MART_DT` refreshes at T=5 minutes after its last cycle — which may start only after `STG_ORDERS_DT` stabilizes. Total: up to ~6 minutes.

**Practical guidance:** Set the downstream DT's lag to be equal to or greater than the upstream DT's lag. Setting `ORDERS_MART_DT` to `'5 minutes'` with `STG_ORDERS_DT` at `'1 minute'` is reasonable — the mart can tolerate more staleness than staging.

---

**Q4 — What is a Materialized View? How is it different from a regular view?**

A **Materialized View** physically stores the query result as a table. When you query it, you read the pre-computed result — no recomputation at query time.

A **regular view** stores only the SQL definition. Every time you query it, Snowflake runs the full query against the source table.

| | Regular View | Materialized View |
|---|---|---|
| Stores data | No — just SQL definition | Yes — physical result set |
| Query time | Recomputes every time | Instant — reads pre-computed result |
| Automatic sync | No | Yes — Snowflake updates it in background |
| Cost | Query warehouse credits each run | Background refresh credits (serverless) |

Use an MV when: the same expensive query runs frequently and the source data changes infrequently enough that background refresh is acceptable.

---

**Q5 — Materialized View vs Dynamic Table. When to choose each?**

| | Materialized View | Dynamic Table |
|---|---|---|
| SQL complexity | Simple SELECT only — no subqueries, limited joins | Any SELECT — subqueries, CTEs, complex joins |
| Primary goal | Speed up a specific read-heavy query | Build a full pipeline layer |
| Incremental refresh | Yes | Yes |
| Chaining | No — cannot read from another MV | Yes — DT can read from another DT |
| Edition required | Enterprise | All editions |

**Choose Materialized View** when: you have one specific, frequently-run query on a large table that you want to accelerate, and that query is simple enough to fit MV's restrictions.

**Choose Dynamic Table** when: you are building a pipeline layer (RAW → STAGING → MART), the SQL is complex, or you need to chain transformations.

---

**Q6 — What is a Stored Procedure? How is it different from a UDF?**

A **Stored Procedure** is reusable, named SQL logic that runs multiple statements, uses variables, loops, and conditionals. You call it with `CALL`. It is for **automation and multi-step workflows**.

A **UDF (User-Defined Function)** is a reusable function called inline in a SELECT — just like `SUM()` or `UPPER()`. It applies a transformation per row and returns a single value. It cannot run DDL or DML.

| | Stored Procedure | UDF |
|---|---|---|
| Called with | `CALL proc()` | `SELECT func(col) FROM ...` |
| Multi-statement | Yes | No — single expression |
| Can run DDL/DML | Yes | No |
| Returns | Scalar value or table | One value per row |
| Use | Pipeline automation, admin | Per-row transformation |

---

**Q7 — Write REFRESH_MART() procedure: suspend tasks, truncate, rebuild, resume, return count.**

```sql
CREATE OR REPLACE PROCEDURE ECOMM_DB.RAW_DATA.REFRESH_MART()
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
DECLARE
    row_count INTEGER;
BEGIN
    -- Step 1: Suspend tasks to avoid interference during rebuild
    ALTER TASK ECOMM_DB.RAW_DATA.STG_TO_MART_TASK SUSPEND;
    ALTER TASK ECOMM_DB.RAW_DATA.RAW_TO_STG_TASK  SUSPEND;

    -- Step 2: Wipe the mart
    TRUNCATE TABLE ECOMM_DB.MARTS.ORDERS_MART;

    -- Step 3: Rebuild mart from staging
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

    -- Step 4: Get row count
    SELECT COUNT(*) INTO :row_count FROM ECOMM_DB.MARTS.ORDERS_MART;

    -- Step 5: Resume tasks (child first, root last)
    ALTER TASK ECOMM_DB.RAW_DATA.STG_TO_MART_TASK RESUME;
    ALTER TASK ECOMM_DB.RAW_DATA.RAW_TO_STG_TASK  RESUME;

    RETURN 'Mart rebuilt. Rows inserted: ' || row_count;
END;
$$;

-- Call it
CALL ECOMM_DB.RAW_DATA.REFRESH_MART();
```

---

**Q8 — What languages can you write Snowflake UDFs and Stored Procedures in?**

**UDFs:**
- SQL
- JavaScript
- Python (via Snowpark)
- Java (via Snowpark)
- Scala (via Snowpark)

**Stored Procedures:**
- SQL (Snowflake Scripting)
- JavaScript
- Python (via Snowpark)
- Java (via Snowpark)
- Scala (via Snowpark)

**Most common in interviews:** SQL procedures (simple, no extra setup) and Python/JavaScript UDFs (for custom string manipulation, complex math, regex). Python stored procedures (Snowpark) are increasingly common for ML and data engineering workflows.

---

**Q9 — Write REVENUE_TIER UDF and use it in a SELECT.**

```sql
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

-- Use it in a SELECT
SELECT
    order_id,
    customer_name,
    total_amount,
    ECOMM_DB.RAW_DATA.REVENUE_TIER(total_amount) AS revenue_tier
FROM ECOMM_DB.RAW_DATA.ORDERS_RAW
ORDER BY total_amount DESC;

-- Aggregate using the UDF
SELECT
    ECOMM_DB.RAW_DATA.REVENUE_TIER(total_amount) AS tier,
    COUNT(*)                                       AS order_count,
    SUM(total_amount)                              AS tier_revenue
FROM ECOMM_DB.RAW_DATA.ORDERS_RAW
GROUP BY tier
ORDER BY tier_revenue DESC;
```

---

**Q10 — Can a UDF run CREATE TABLE? Can a Stored Procedure?**

**UDF: No.** UDFs are pure functions — they take input, return a value, and have no side effects. They cannot run DDL (CREATE, ALTER, DROP) or DML (INSERT, UPDATE, DELETE). Snowflake enforces this at the function-creation level. The reason: UDFs are called per-row inside a SELECT — running a CREATE TABLE once per row would be meaningless and destructive.

**Stored Procedure: Yes.** Procedures are designed for multi-step automation. They can run DDL, DML, and control flow. This is one of the main reasons to use a procedure instead of a UDF — when your logic needs to modify the database state, not just transform a value.

```sql
-- This is valid in a stored procedure:
BEGIN
    CREATE TABLE IF NOT EXISTS backup_table CLONE source_table;
    INSERT INTO audit_log VALUES (CURRENT_TIMESTAMP(), 'backup created');
END;

-- This is NOT valid in a UDF
```

---

**Q11 — What is a Row Access Policy? How is it different from a filtered view?**

A **Row Access Policy** is a function attached directly to a table that transparently filters rows based on the calling user's role or attributes. No matter how the table is queried — directly, through a view, through a join — the policy always applies.

A **filtered view** is a view with a WHERE clause. It only protects data if everyone is forced to query through that view. If a user has direct table access, they bypass it. If you create a second view from the same table without the filter, the data is exposed again.

| | Filtered View | Row Access Policy |
|---|---|---|
| Enforcement | Only if users query the specific view | Always — table, view, join, any path |
| Maintenance | One view per access pattern needed | One policy on the table |
| Bypass risk | Yes — if user has direct table grants | No — policy is on the table itself |
| Dynamic (role-based) | Needs multiple views or CURRENT_ROLE() logic | Native — CURRENT_ROLE() in the policy |

---

**Q12 — Row access policy: SALES sees CONFIRMED/SHIPPED, FINANCE sees all, SYSADMIN sees all.**

```sql
USE ROLE SYSADMIN;
USE SCHEMA ECOMM_DB.RAW_DATA;

-- Create the policy
CREATE OR REPLACE ROW ACCESS POLICY ECOMM_DB.RAW_DATA.ORDER_ROLE_FILTER
    AS (status VARCHAR) RETURNS BOOLEAN ->
        CASE
            WHEN CURRENT_ROLE() IN ('SYSADMIN', 'FINANCE') THEN TRUE
            WHEN CURRENT_ROLE() = 'SALES'
                 THEN status IN ('CONFIRMED', 'SHIPPED')
            ELSE FALSE
        END;

-- Attach to ORDERS_RAW on the status column
ALTER TABLE ECOMM_DB.RAW_DATA.ORDERS_RAW
    ADD ROW ACCESS POLICY ECOMM_DB.RAW_DATA.ORDER_ROLE_FILTER
    ON (status);

-- As SYSADMIN: all rows visible
SELECT status, COUNT(*) FROM ORDERS_RAW GROUP BY status;

-- As FINANCE role: all rows visible (same as SYSADMIN)
-- As SALES role: only CONFIRMED and SHIPPED rows visible

-- Remove when done
ALTER TABLE ECOMM_DB.RAW_DATA.ORDERS_RAW
    DROP ROW ACCESS POLICY ECOMM_DB.RAW_DATA.ORDER_ROLE_FILTER;
```

---

**Q13 — What is a Column Masking Policy? Real-world example.**

A **Column Masking Policy** is a function attached to a specific column that replaces its value with a masked version for certain roles — without changing the stored data.

```sql
CREATE MASKING POLICY mask_email AS (val VARCHAR) RETURNS VARCHAR ->
    CASE
        WHEN CURRENT_ROLE() = 'SYSADMIN' THEN val
        ELSE REGEXP_REPLACE(val, '.+@', '****@')
    END;
```

**Real-world examples:**
- **Customer PII** — analysts query order data but see `****@gmail.com` instead of `alice@gmail.com`
- **Credit card numbers** — finance team sees `****-****-****-1234` instead of full number
- **Salary data** — HR sees real values, managers see `*****`
- **Phone numbers** — customer service sees last 4 digits only: `****3210`

The data is never changed in storage — only the value returned to non-privileged roles is masked. SYSADMIN always sees the real value.

---

**Q14 — Masking policy on ORDERS_RAW.customer_name. Does it apply when queried through a view?**

**Yes.** Column masking policies are enforced at the **table level**, not the view level. Any query that touches `customer_name` from `ORDERS_RAW` — whether directly, through a view, through a join, or through a Dynamic Table — has the masking applied.

```sql
-- Even this view will show masked names for non-SYSADMIN roles:
CREATE VIEW customer_orders AS
SELECT order_id, customer_name, total_amount FROM ORDERS_RAW;

SELECT * FROM customer_orders;
-- customer_name column is still masked for non-SYSADMIN
```

This is the key advantage over manually masking inside a view — you cannot forget to apply it, and it cannot be bypassed by going directly to the table or creating a different view.

---

**Q15 — Analysts query order data but must never see real customer names or emails. Cleanest solution.**

Apply **Column Masking Policies** directly on the sensitive columns in `ORDERS_RAW`. No duplicate tables needed.

```sql
USE ROLE SYSADMIN;
USE SCHEMA ECOMM_DB.RAW_DATA;

-- Policy for customer_name
CREATE OR REPLACE MASKING POLICY ECOMM_DB.RAW_DATA.MASK_NAME
    AS (val VARCHAR) RETURNS VARCHAR ->
        CASE
            WHEN CURRENT_ROLE() IN ('SYSADMIN', 'DATA_ENGINEER') THEN val
            ELSE '***MASKED***'
        END;

-- Apply to customer_name in ORDERS_RAW
ALTER TABLE ECOMM_DB.RAW_DATA.ORDERS_RAW
    MODIFY COLUMN customer_name
    SET MASKING POLICY ECOMM_DB.RAW_DATA.MASK_NAME;

-- Grant ANALYST role SELECT on ORDERS_RAW normally
GRANT SELECT ON TABLE ECOMM_DB.RAW_DATA.ORDERS_RAW TO ROLE ANALYST;

-- ANALYST queries the full table — sees all rows, all columns
-- But customer_name shows '***MASKED***' instead of real names
-- No second table, no restricted view, no duplicated data
```

**Why this is cleaner than alternatives:**
- One table, one source of truth — no sync issues
- Masking is automatic for any new view or query on that column
- SYSADMIN/DATA_ENGINEER still see real data for debugging
- Audit trail: `INFORMATION_SCHEMA.POLICY_REFERENCES` shows exactly which columns are masked and for whom
