# Exercise 2 — Materializations

**Time: ~2 hours**

## Goal
Understand how each materialization behaves in practice by running the same model with different configurations and measuring the impact on TPC-H's large tables.

---

## Part A — View vs Table

> *Guided — follow along, full solution shown.*

Your bronze models are currently configured as `view` in `dbt_project.yml`. Let's compare.

**Step 1:** Run `bronze_tpch_lineitem` (6M rows) and time it:
```bash
time dbt run --select bronze_tpch_lineitem
```

**Step 2:** Override the materialization to `table` at the top of `bronze_tpch_lineitem.sql`:
```sql
{{ config(materialized='table') }}

SELECT ...
```

Run it again and compare. Also query downstream from both in Snowflake — which is faster for a `GROUP BY` query? Why?

**Step 3:** Remove the config block (revert to `view`).

**Question:** `bronze_tpch_orders` has 1.5M rows. Would you ever materialize a bronze model as a table? What would be the trade-off?

---

## Part B — Ephemeral Model

> *Guided — follow along, full solution shown.*

Create `models/silver/silver_lineitem_totals.sql` as an ephemeral model:

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

Update `silver_orders_enriched.sql` to use `{{ ref('silver_lineitem_totals') }}` instead of the inline CTE `line_totals` you wrote earlier.

Run:
```bash
dbt run --select silver_orders_enriched
```

**Questions:**
1. Does `silver_lineitem_totals` appear as a table or view in Snowflake? Why not?
2. Open `silver_orders_enriched.sql` in the Cloud IDE and click **Compile**. What did dbt do with the ephemeral model?
3. What is the trade-off between ephemeral and a regular view?

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

Add a new model `models/gold/gold_nations_summary.sql`:

```
# Your solution here
```

Configure it as a **view** even though the rest of `gold/` uses `table`. Do this two ways:

1. Via a `{{ config() }}` block inside the model file
2. Via `dbt_project.yml` with a per-model config under `+config`

Which approach do you prefer and why?

---

## ✅ Success Criteria

- [ ] You measured the speed difference between view and table on 6M-row lineitem
- [ ] `silver_lineitem_totals` is ephemeral and does not appear in Snowflake
- [ ] `silver_orders_enriched` uses the ephemeral model and produces correct results
- [ ] `gold_customers` is materialized as a table
- [ ] `gold_nations_summary` is overridden to `view`
