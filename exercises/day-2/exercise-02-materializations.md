# Exercise 2 — Materializations

**Time: ~2 hours**

## Goal
Understand how each materialization behaves in practice by running the same model with different configurations and measuring the impact on TPC-H's large tables.

---

## Part A — View vs Table

> *Guided — follow along, full solution shown.*

Your bronze models are currently configured as `view` in `dbt_project.yml`. Let's compare.

**Step 1:** Run `bronze_tpch_lineitem` (6M rows) and nota el tiempo que aparece en el log:
```bash
dbt run --select bronze_tpch_lineitem
```

**Step 2:** Override the materialization to `table` at the top of `bronze_tpch_lineitem.sql`:
```sql
{{ config(materialized='table') }}

SELECT ...
```

Run it again and compare. Also query downstream from both in Snowflake — which is faster for a `GROUP BY` query? Why?

**Step 3:** Remove the config block (revert to `view`).

---

## Part B — Ephemeral Model

> *Guided — follow along, full solution shown.*

**Step 1 — Build a market segment summary.**

Create `models/gold/gold_market_summary.sql`. It should show, per market segment, total orders, total revenue, and average order value:

```sql
WITH line_totals AS (
    SELECT
        order_id,
        SUM(net_price) AS net_revenue
    FROM {{ ref('bronze_tpch_lineitem') }}
    GROUP BY 1
),
orders AS (
    SELECT * FROM {{ ref('bronze_tpch_orders') }}
),
customers AS (
    SELECT * FROM {{ ref('bronze_tpch_customers') }}
)

SELECT
    c.market_segment,
    COUNT(DISTINCT o.order_id)                AS total_orders,
    SUM(l.net_revenue)                        AS total_revenue,
    SUM(l.net_revenue) /
        NULLIF(COUNT(DISTINCT o.order_id), 0) AS avg_order_value
FROM orders o
JOIN customers c   ON o.customer_id = c.customer_id
JOIN line_totals l ON o.order_id    = l.order_id
GROUP BY 1
```

Run it:
```bash
dbt run --select gold_market_summary
```

**Step 2 — Introduce the ephemeral materialization.**

The `line_totals` aggregation is an intermediate step — it doesn't belong in bronze, silver, or gold, and you wouldn't expose it to analysts. This is the use case for `ephemeral`: a model that exists in code but never creates an object in the warehouse.

Extract it to `models/silver/silver_lineitem_totals.sql`:

```sql
{{ config(materialized='ephemeral') }}

SELECT
    order_id,
    COUNT(*)            AS total_items,
    SUM(extended_price) AS gross_revenue,
    SUM(net_price)      AS net_revenue
FROM {{ ref('bronze_tpch_lineitem') }}
GROUP BY 1
```

Update `silver_orders_enriched.sql` and `gold_market_summary.sql` to replace their inline `line_totals` CTE with:

```sql
line_totals AS (
    SELECT * FROM {{ ref('silver_lineitem_totals') }}
),
```

Run both:
```bash
dbt run --select silver_orders_enriched gold_market_summary
```

**Questions:**
1. Does `silver_lineitem_totals` appear as a table or view in Snowflake? Why not?
2. Open `silver_orders_enriched.sql` and click **Compile**. What happened to the ephemeral model in the compiled SQL?
3. What is the trade-off between ephemeral and a regular view? When would you choose each?

---

## Part C — Table Materialization for Gold

> *Guided — follow along, full solution shown.*

`dbt_project.yml` already sets `gold: +materialized: table`. Verify it works:

```bash
dbt run --select gold_customers
```

Check in Snowflake:
```sql
SHOW TABLES IN SCHEMA ANALYTICS.GOLD;
```

Run it a second time and observe the **Query History** in Snowflake. What SQL does dbt execute on the second run?

---

## Part D — Per-Model Config

> *Practice — write this yourself.*

`dbt_project.yml` sets `gold: +materialized: table` for the entire folder. But materializing as a table means dbt scans and stores the data physically on every `dbt run` — that cost is only justified for large or expensive models. `gold_nations_summary` aggregates 25 countries: querying it as a view is essentially free, so there's no reason to pay the scan cost on every run.

Create `models/gold/gold_nations_summary.sql` that shows total revenue and order count by country:

- Join `silver_orders_enriched` with `bronze_tpch_nations`
- Group by `nation_name`
- Include: `nation_name`, `total_orders`, `total_revenue`, `avg_order_value`

Configure it as a **view** using two different approaches — implement it both ways and observe the result:

**Option 1 — inline config block** (takes precedence over `dbt_project.yml`):
```sql
{{ config(materialized='view') }}
```

**Option 2 — `dbt_project.yml`** (useful when you want to override many models at once without touching each file):
```yaml
models:
  my_new_project:
    gold:
      +materialized: table
      gold_nations_summary:  # per-model override
        +materialized: view
```

Run and verify:
```bash
dbt run --select gold_nations_summary
```

```sql
-- Should appear under VIEWS, not TABLES
SHOW VIEWS IN SCHEMA ANALYTICS.GOLD;
```

**Question:** Between the inline `{{ config() }}` block and `dbt_project.yml`, which approach would you use in a team setting where multiple people work on the same project? Why?

---

## ✅ Success Criteria

- [ ] You measured the speed difference between view and table on `bronze_tpch_lineitem`
- [ ] `gold_market_summary` produces 5 rows (one per market segment) with correct totals
- [ ] `silver_lineitem_totals` is ephemeral and does not appear in Snowflake
- [ ] Both `silver_orders_enriched` and `gold_market_summary` use `{{ ref('silver_lineitem_totals') }}` — no duplicated CTEs
- [ ] `gold_customers` is materialized as a table
- [ ] `gold_nations_summary` is overridden to `view`
