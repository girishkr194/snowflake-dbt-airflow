# Day 1 — Interview Solutions: dbt Introduction and Setup

> Complete answers. Read after attempting on your own.

---

**Q1 — What is dbt and what problem does it solve in a modern data stack?**

**dbt (data build tool)** is an open-source transformation framework that lets data teams write SQL `SELECT` statements as versioned, testable, documented files and run them inside the data warehouse.

**The problem it solves:** In a traditional setup, analysts write ad-hoc SQL queries with no version control, no dependency tracking, no testing, and no documentation. When a table breaks, nobody knows which queries depend on it. dbt treats SQL transformations as software — each model is a file in Git, models declare dependencies on each other with `ref()`, and dbt runs them in the correct order automatically.

It fills the **T** in **ELT**: Extract and Load are handled by tools like Fivetran or Snowpipe; dbt handles the Transform step inside the warehouse.

---

**Q2 — What does dbt actually do when you run `dbt run`?**

For each `.sql` model file, dbt:
1. Reads the SELECT statement
2. Resolves `{{ ref() }}` and `{{ source() }}` calls to fully-qualified table names
3. Wraps the SELECT in a `CREATE OR REPLACE VIEW/TABLE AS` based on the materialization config
4. Executes the generated SQL in the warehouse
5. Reports success or failure per model

Example — you write:
```sql
SELECT order_id, customer_name FROM {{ ref('stg_orders') }}
```

dbt executes in Snowflake:
```sql
CREATE OR REPLACE TABLE ECOMM_DB.DBT_DEV.orders_mart AS (
    SELECT order_id, customer_name FROM ECOMM_DB.DBT_DEV.stg_orders
)
```

You write plain SELECT. dbt handles the DDL.

---

**Q3 — Where does dbt fit in the ELT pipeline? What does it NOT do?**

```
Source Systems  →  [Extract & Load]  →  Warehouse (raw)  →  [dbt Transform]  →  BI Tool
                   Fivetran/Airbyte                          dbt models
                   Snowpipe/COPY INTO
```

dbt handles only the **Transform** step — it reads raw tables already in the warehouse and produces clean, structured output tables and views.

**dbt does NOT:**
- Extract data from source systems
- Load data into the warehouse (no INSERT from external files)
- Schedule itself (dbt Core has no scheduler — use Airflow, cron, or dbt Cloud)
- Move data between warehouses

---

**Q4 — Difference between dbt Core and dbt Cloud? Which for Airflow-managed production?**

| | dbt Core | dbt Cloud |
|---|---|---|
| Type | Open-source CLI | Managed SaaS on top of dbt Core |
| Cost | Free | Free tier + paid plans |
| Scheduling | External — Airflow, cron | Built-in scheduler |
| Interface | Terminal + your IDE | Web IDE + UI |

**For a production pipeline managed by Airflow: dbt Core.** Airflow calls `dbt run` as a BashOperator or uses the `DbtRunOperator` from the Astronomer Cosmos package. You get full control over scheduling, retries, and dependency ordering through Airflow — dbt Cloud's built-in scheduler would be redundant.

---

**Q5 — What is `profiles.yml`? Why is it outside the project and not committed to Git?**

`profiles.yml` is the connection configuration file. It maps a profile name to Snowflake (or other warehouse) connection details: account identifier, username, password, role, database, warehouse, schema, and thread count.

It lives at `~/.dbt/profiles.yml` (the user's home directory, outside the project) for two reasons:
1. **Security** — it contains credentials (passwords, private keys). Committing it to Git would expose them to everyone with repo access.
2. **Portability** — different developers connect to different environments (dev schema, different accounts). The project code is the same; only the profile differs per person or environment.

---

**Q6 — What is a dbt model?**

A **dbt model** is a `.sql` file containing a single `SELECT` statement. The filename (without `.sql`) becomes the name of the view or table dbt creates in the warehouse.

```sql
-- models/staging/stg_orders.sql
SELECT
    order_id,
    customer_name,
    total_amount
FROM {{ source('raw_data', 'orders_raw') }}
```

Running `dbt run` creates `DBT_DEV.STG_ORDERS` as a view (or table, depending on materialization config). You write only the SELECT logic — dbt generates and runs the `CREATE OR REPLACE` statement.

---

**Q7 — What is `{{ ref() }}`? What two things does it do beyond resolving a table name?**

`{{ ref('model_name') }}` does three things:

1. **Resolves the name** — returns the fully-qualified table/view name for the referenced model in the current target schema (e.g., `ECOMM_DB.DBT_DEV.STG_ORDERS`). This means you never hardcode schema names.

2. **Builds the dependency graph** — dbt registers a dependency between the current model and the referenced model. This ensures dbt runs `stg_orders` before any model that calls `ref('stg_orders')`.

3. **Enables lineage tracking** — dbt's docs and lineage graph are built from these `ref()` calls, so you can visualize which models depend on which.

---

**Q8 — What is `{{ source() }}`? How is it different from `ref()`?**

`{{ source('source_name', 'table_name') }}` references a **raw table** declared in a `sources.yml` file — a table that was loaded by an EL tool, not built by dbt.

`{{ ref('model_name') }}` references another **dbt model** — a `.sql` file in your project.

| | `source()` | `ref()` |
|---|---|---|
| Points to | Raw table loaded externally | Another dbt model |
| Declared in | `sources.yml` | Implicitly — the .sql file is the declaration |
| Enables | Source freshness checks, lineage to raw data | Model-to-model dependency ordering |

**When to use each:**
- Use `source()` for the first layer — staging models that read from raw tables
- Use `ref()` for every downstream model that reads from another model

---

**Q9 — `orders_mart.sql` references `ref('stg_orders')`. Which runs first?**

**`stg_orders` runs first.** dbt builds a Directed Acyclic Graph (DAG) from all `ref()` calls in the project. Because `orders_mart` calls `ref('stg_orders')`, dbt registers `stg_orders` as a dependency of `orders_mart`. dbt always runs dependencies before the models that depend on them.

If you try to run `orders_mart` before `stg_orders` exists, dbt would fail — but dbt's DAG execution prevents this automatically.

---

**Q10 — The four dbt materializations?**

| Materialization | What dbt creates | Use |
|---|---|---|
| `view` | `CREATE OR REPLACE VIEW` — no data stored, query executes at read time | Staging/lightweight models |
| `table` | `CREATE OR REPLACE TABLE AS SELECT` — full rebuild every run | Marts, frequently queried aggregations |
| `incremental` | `INSERT` new rows / `MERGE` changes — only processes delta | Large tables where full rebuild is too slow |
| `ephemeral` | No object in warehouse — compiled as a CTE into the referencing model | Intermediate logic used by one downstream model |

---

**Q11 — Raw schema renamed from `RAW_DATA` to `RAW`. Minimum change needed?**

Update **one line** in `sources.yml`:

```yaml
# Before
sources:
  - name: raw_data
    database: ECOMM_DB
    schema: RAW_DATA   # ← change this

# After
sources:
  - name: raw_data
    database: ECOMM_DB
    schema: RAW        # ← updated here only
```

Every model that uses `{{ source('raw_data', 'orders_raw') }}` automatically resolves to the new schema on the next `dbt run`. You do not change any model SQL files. This is exactly why `source()` exists — a single change propagates everywhere.

---

**Q12 — `materialized='table'` on staging, `materialized='view'` on mart. What is wrong?**

**Staging as table:** Every `dbt run` does a full `CREATE TABLE AS SELECT` on the staging model, which costs compute and time — but staging models are usually just light renaming/casting, so a view is cheaper and equally correct.

**Mart as view:** The mart aggregation query runs every time a BI tool or analyst queries it. For large aggregations, this is expensive. If 100 dashboards query `orders_mart` simultaneously, 100 full aggregation queries hit Snowflake. As a table, the aggregation runs once per `dbt run` and all 100 reads hit the pre-computed result.

**Convention to follow:** staging → `view` (cheap, no data stored), marts → `table` (pre-computed, fast to query).

---

**Q13 — What does `dbt debug` check?**

`dbt debug` runs a pre-flight checklist:

1. **dbt version** — reports the installed version of dbt Core and the adapter
2. **profiles.yml location and syntax** — confirms the file exists and is valid YAML
3. **Profile and target** — verifies the profile name in `dbt_project.yml` matches a key in `profiles.yml`
4. **Warehouse connection** — attempts a live connection to Snowflake (or whichever adapter)
5. **Required permissions** — verifies the configured role can connect and use the specified warehouse and database
6. **dbt_project.yml syntax** — checks the project config file is valid

Output: each check is listed as `OK` or `ERROR` with the specific failure reason.

---

**Q14 — One model fails, downstream models are `skipped`. Why?**

dbt builds a DAG (Directed Acyclic Graph) of all models and their dependencies. When a model fails, its output table or view either does not exist or is stale. Any model downstream of it would be querying invalid data — so dbt marks them `skipped` rather than attempting to run them on bad input.

This is safer than running them: a downstream model that runs against a failed upstream might produce incorrect results silently, which is worse than an explicit skip.

The failed model and all its `skipped` dependents are shown in the run summary so you know exactly what to fix and re-run.

---

**Q15 — `dbt run` vs `dbt run --select stg_orders+`. When use the `+` selector?**

`dbt run` runs **all models** in the project.

`dbt run --select stg_orders+` runs `stg_orders` **and all models downstream of it** (the `+` after the name means "and descendants").

Other selector variants:
```bash
dbt run --select stg_orders          # only stg_orders
dbt run --select stg_orders+         # stg_orders + all downstream
dbt run --select +stg_orders         # stg_orders + all upstream
dbt run --select +stg_orders+        # full subtree: upstream + stg_orders + downstream
dbt run --select staging.*           # all models in the staging folder
dbt run --select tag:daily           # all models tagged 'daily'
```

**When to use `+`:** When you've changed a model and want to re-run it plus everything that depends on it — to propagate the change end-to-end without re-running unrelated models.
