# Day 3 — Interview Questions: COPY INTO, Streams, and Tasks

> 21 questions across all three concepts. Attempt your own answer first, then check the solutions file.
> Difficulty tags: (Warm-up) → (Conceptual) → (Scenario) → (Tricky) → (Deep dive) → (System design)

---

## Concept 1: COPY INTO — Bulk Loading

**Q1 (Warm-up)** — What does `COPY INTO` do, and why can't you just `INSERT` a CSV file directly into a Snowflake table?

**Q2 (Warm-up)** — What does it mean that COPY INTO is "idempotent"? How long does Snowflake remember which files have already been loaded?

**Q3 (Conceptual)** — What is `VALIDATION_MODE`? Name two values you can pass to it and what each does. Does it load any data?

**Q4 (Conceptual)** — What is `ON_ERROR = 'ABORT_STATEMENT'` vs `ON_ERROR = 'CONTINUE'`? Which would you use for a financial transactions load and why?

**Q5 (Scenario)** — You have a stage with 300 files from multiple months. You only want to load files matching the pattern `orders_2024_01_*.csv.gz`. How do you do it in COPY INTO?

**Q6 (Conceptual)** — What are `$1`, `$2`, `$3` in a COPY INTO statement? When would you use them instead of a direct column-to-column load?

**Q7 (Tricky)** — What is `METADATA$FILENAME` and why should you capture it in every raw table? Name two other metadata columns available during COPY INTO.

**Q8 (Deep dive)** — You run COPY INTO and load 10,000 rows. You immediately run the same COPY INTO again. How many rows are loaded the second time and why? How do you force a reload?

---

## Concept 2: Streams — Change Data Capture

**Q9 (Warm-up)** — What is a Snowflake Stream? What problem does it solve compared to reloading an entire table every time?

**Q10 (Warm-up)** — What are the three metadata columns every Stream adds to its rows? What are the possible values for `METADATA$ACTION`?

**Q11 (Conceptual)** — How does a Snowflake Stream represent an UPDATE? How many rows appear in the stream for a single UPDATE, and what are their `METADATA$ACTION` and `METADATA$ISUPDATE` values?

**Q12 (Conceptual)** — What is the difference between a standard stream and an `APPEND ONLY` stream? When would you choose append-only?

**Q13 (Tricky)** — You query a stream and see 500 change records. You run a `SELECT *` from the stream. Does the offset advance? What action actually advances the stream offset?

**Q14 (Scenario)** — You have a `CUSTOMERS_RAW` table that receives inserts and updates throughout the day. You want to keep a `STG_CUSTOMERS` staging table in sync. Write the MERGE statement that uses the stream to apply only the new inserts and updates.

**Q15 (Deep dive)** — What does it mean for a stream to become "stale"? What causes it, and what do you have to do to fix it?

---

## Concept 3: Tasks — Scheduling and Automation

**Q16 (Warm-up)** — What is a Snowflake Task? What two scheduling approaches does it support?

**Q17 (Conceptual)** — Tasks are created in SUSPENDED state by default. Why do you think Snowflake does this? What command do you run to start a task?

**Q18 (Scenario)** — Write a Task that runs every day at 2 AM UTC and executes a COPY INTO from a named stage into a raw table.

**Q19 (Tricky)** — What does `WHEN SYSTEM$STREAM_HAS_DATA('schema.stream_name')` do in a Task definition? Why is it important for cost management?

**Q20 (Deep dive)** — Explain how Task chaining works. What is the difference between a root task and a child task? In what order do you resume them and why?

**Q21 (System design)** — Design a fully automated Snowflake-native pipeline for an e-commerce company. Data arrives as CSV files in an Azure Blob container every hour. You need to: load raw data, apply upserts to a staging table, and refresh a mart table. Use stages, file formats, COPY INTO, a stream, and chained tasks. Specify schedules, ON_ERROR settings, and the WHEN condition.

---

## Mixed / Senior-Level Questions

**Q22 (Scenario)** — Your nightly Task fails because the warehouse was suspended. The stream now has 24 hours of changes piled up. What do you check first, and how do you recover without losing changes?

**Q23 (Tricky)** — What is the difference between `EXECUTE TASK` and waiting for the scheduled run? When would you use `EXECUTE TASK` in production?
