# Day 5 — Interview Questions: Stages, COPY INTO, Snowpipe, and Zero-Copy Cloning

> 15 questions. Difficulty tags: (Warm-up) → (Conceptual) → (Scenario) → (Tricky)

---

**Q1 (Warm-up)** — What is a Stage in Snowflake? Why do you need one before loading data into a table?

**Q2 (Warm-up)** — What is the difference between an internal stage and an external stage?

**Q3 (Conceptual)** — Snowflake has three types of internal stage: user, table, and named. What is the difference between them? Which one should you use in a production pipeline and why?

**Q4 (Warm-up)** — What does `COPY INTO` do? What compute resource does it use?

**Q5 (Conceptual)** — You run `COPY INTO` on a stage that has 10 files, all already loaded previously. What happens? Why?

**Q6 (Tricky)** — A developer uses `FORCE = TRUE` in a `COPY INTO` command. What is the risk? When is it acceptable?

**Q7 (Scenario)** — You need to load only specific columns from a CSV file and add a `loaded_at` timestamp column. `COPY INTO` runs directly against a stage — it cannot join or filter. How do you do it?

**Q8 (Warm-up)** — What is Snowpipe? How is it different from running `COPY INTO` on a 1-minute scheduled Task?

**Q9 (Conceptual)** — Snowpipe is described as "serverless." What does that mean for compute and billing compared to a Task that runs COPY INTO?

**Q10 (Tricky)** — You set up a Pipe with `AUTO_INGEST = FALSE` and run `ALTER PIPE ... REFRESH` to trigger loading. The same file is in the stage but Snowpipe does not load it again. Why?

**Q11 (Scenario)** — Your Snowpipe loaded files for 2 weeks. This morning it shows `PIPE_EXECUTION_PAUSED = TRUE` and 20 files in the stage were not loaded. How do you recover without losing those 20 files?

**Q12 (Warm-up)** — What is zero-copy cloning in Snowflake? What does "zero-copy" mean technically?

**Q13 (Conceptual)** — You clone a 5 TB production table. A developer then deletes 1 million rows from the clone. How much new storage is created, and where?

**Q14 (Tricky)** — Can you clone a table that has a Stream on it? What happens to the stream in the clone?

**Q15 (Scenario)** — At 2 AM a developer ran `DELETE FROM STG_ORDERS WHERE order_date < '2024-01-01'` and deleted 3 million rows by mistake. The table has a 1-day Time Travel window. Write the exact SQL to recover the deleted rows using cloning.
