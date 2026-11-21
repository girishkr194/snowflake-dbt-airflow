# Day 1 — Interview Questions: dbt Introduction and Setup

> 15 questions. Difficulty: (Warm-up) → (Conceptual) → (Scenario) → (Tricky)

---

**Q1 (Warm-up)** — What is dbt and what problem does it solve in a modern data stack?

**Q2 (Warm-up)** — What does dbt actually do when you run `dbt run`? What SQL does it generate?

**Q3 (Conceptual)** — Where does dbt fit in the ELT pipeline? What does it NOT do?

**Q4 (Warm-up)** — What is the difference between dbt Core and dbt Cloud? Which would you use in a production pipeline managed by Airflow?

**Q5 (Conceptual)** — What is `profiles.yml`? Why is it kept outside the project directory and not committed to Git?

**Q6 (Warm-up)** — What is a dbt model? What does the `.sql` file contain?

**Q7 (Conceptual)** — What is `{{ ref() }}` in dbt? What two things does it do beyond just resolving a table name?

**Q8 (Conceptual)** — What is `{{ source() }}`? How is it different from `ref()`? When do you use each?

**Q9 (Scenario)** — You have a model `orders_mart.sql` that runs `SELECT ... FROM {{ ref('stg_orders') }}`. When you run `dbt run`, which model runs first and why?

**Q10 (Warm-up)** — What are the four dbt materializations? Briefly describe what each creates in the warehouse.

**Q11 (Scenario)** — Your staging model `stg_orders` reads from `ECOMM_DB.RAW_DATA.ORDERS_RAW`. Tomorrow, the raw schema is renamed from `RAW_DATA` to `RAW`. What is the minimum change you need to make in your dbt project?

**Q12 (Tricky)** — A colleague sets `materialized='table'` on a staging model and `materialized='view'` on the mart model. What is wrong with this and what are the consequences?

**Q13 (Conceptual)** — What does `dbt debug` check? List at least four things it verifies.

**Q14 (Scenario)** — You run `dbt run` and one model fails. The models downstream of it are marked as `skipped`, not `failed`. Why does dbt skip them instead of trying to run them?

**Q15 (Tricky)** — What is the difference between `dbt run` and `dbt run --select stg_orders+`? When would you use the `+` selector?
