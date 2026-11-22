# Day 2 — Interview Questions: Testing, Documentation, and Advanced Models

> 15 questions. Difficulty: (Warm-up) → (Conceptual) → (Scenario) → (Tricky)
> All scenarios use the ecomm_dbt project with ORDERS_RAW → stg_orders → orders_mart.

---

**Q1 (Warm-up)** — What is the purpose of a `schema.yml` file in dbt? What two things can you define inside it?

**Q2 (Warm-up)** — Name the four built-in dbt generic tests. For each one, describe in one sentence what it checks.

**Q3 (Conceptual)** — What SQL does dbt generate behind the scenes for a `not_null` test on `order_id` in `stg_orders`? Write it out.

**Q4 (Warm-up)** — What is the difference between `dbt test` and `dbt build`?

**Q5 (Scenario)** — You run `dbt test` and see this output:
```
FAIL 3  test not_null_stg_orders_customer_name
```
What does the `3` mean? How do you find which exact rows caused the failure?

**Q6 (Conceptual)** — What is `dbt source freshness` and how does dbt know if a source is stale? What two fields in `sources.yml` control this?

**Q7 (Warm-up)** — What is an incremental model in dbt? How is the first run different from all subsequent runs?

**Q8 (Scenario)** — Write the SQL for an incremental model `stg_orders_incremental` that:
- Uses `order_id` as the unique key
- On subsequent runs, only processes rows where `updated_at` is newer than the latest `updated_at` already in the table
Include the full Jinja block with `is_incremental()` and explain each line.

**Q9 (Tricky)** — What does `{{ this }}` resolve to inside an incremental model? Why can't you use `ref('stg_orders_incremental')` instead?

**Q10 (Conceptual)** — What is the difference between `{% if %}` and `{{ }}` in Jinja? Give an example of each from a dbt model.

**Q11 (Tricky)** — You have an incremental model that has been running daily for 6 months. You discover a bug in the transformation logic that affects all historical rows. How do you fix the data?

**Q12 (Scenario)** — Your `accepted_values` test on `status` currently allows: `CONFIRMED`, `SHIPPED`, `DELIVERED`, `CANCELLED`. The business adds a new status `RETURNED`. What breaks, what do you change, and in what order?

**Q13 (Conceptual)** — What is the difference between `sources.yml` and `schema.yml` in dbt? Can you put source declarations and model declarations in the same file?

**Q14 (Scenario)** — A new analyst joins the team and asks: "How do I know what `total_amount` means — is it pre-tax, post-tax, in USD or EUR?" What dbt feature answers this question without bothering the data engineer, and where do they find it?

**Q15 (Tricky)** — You set `materialized='incremental'` with `unique_key='order_id'`. An existing order changes its `status` from `CONFIRMED` to `SHIPPED` in the source. On the next `dbt run`, what happens to this row in your incremental table? What SQL does dbt generate?
