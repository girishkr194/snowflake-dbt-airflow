# Day 7 — Interview Questions: Dynamic Tables, Materialized Views, Stored Procedures & UDFs, Security Policies

> 15 questions. Difficulty: (Warm-up) → (Conceptual) → (Scenario) → (Tricky)
> All scenarios use the Day 4 tables: ORDERS_RAW, STG_ORDERS, ORDERS_MART.

---

**Q1 (Warm-up)** — What is a Dynamic Table in Snowflake? What is `TARGET_LAG` and what does it control?

**Q2 (Conceptual)** — Your team currently has a stream on `ORDERS_RAW` and a Task that runs a MERGE every minute to sync `STG_ORDERS`. When should you replace this with a Dynamic Table instead, and when should you keep the stream + task approach?

**Q3 (Tricky)** — You create two Dynamic Tables: `STG_ORDERS_DT` reads from `ORDERS_RAW`, and `ORDERS_MART_DT` reads from `STG_ORDERS_DT`. `STG_ORDERS_DT` has a 1-minute lag, `ORDERS_MART_DT` has a 5-minute lag. How long after a change to `ORDERS_RAW` does it take to appear in `ORDERS_MART_DT`?

**Q4 (Warm-up)** — What is a Materialized View? How is it different from a regular view?

**Q5 (Conceptual)** — What is the key difference between a Materialized View and a Dynamic Table? When would you choose one over the other?

**Q6 (Warm-up)** — What is a Stored Procedure? How is it different from a UDF?

**Q7 (Scenario)** — Write a stored procedure `REFRESH_MART()` that: (1) suspends both Day 4 tasks, (2) truncates `ORDERS_MART`, (3) rebuilds it from `STG_ORDERS`, then (4) resumes the tasks. The procedure should return how many rows were inserted into the mart.

**Q8 (Conceptual)** — What languages can you write Snowflake UDFs and Stored Procedures in?

**Q9 (Scenario)** — You need a UDF `REVENUE_TIER(total NUMBER)` that returns `'LOW'` for < 100, `'MID'` for 100–499, and `'HIGH'` for 500+. Write it and show how to use it in a SELECT on `ORDERS_RAW`.

**Q10 (Tricky)** — Can a UDF run a `CREATE TABLE` statement? Can a Stored Procedure? Explain why or why not.

**Q11 (Warm-up)** — What is a Row Access Policy? How is it different from just granting SELECT on a filtered view?

**Q12 (Scenario)** — Your company has two teams: `SALES` role and `FINANCE` role. `SALES` should only see orders with `status IN ('CONFIRMED','SHIPPED')`. `FINANCE` should see all orders. `SYSADMIN` sees everything. Write the row access policy and attach it to `ORDERS_RAW`.

**Q13 (Warm-up)** — What is a Column Masking Policy? Give a real-world example of when you would use one.

**Q14 (Tricky)** — A masking policy is applied to `customer_name` in `ORDERS_RAW`. An analyst queries a view that selects from `ORDERS_RAW` and includes `customer_name`. Does the masking policy apply to the view result too?

**Q15 (Scenario)** — Your security team says: "Analysts should be able to query order data but never see real customer names or emails." You have one `ORDERS_RAW` table and don't want to create a separate filtered copy of it. What is the cleanest Snowflake solution?
