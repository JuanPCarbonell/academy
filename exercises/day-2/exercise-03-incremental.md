# Exercise 3 — Incremental Models

**Time: ~2.5 hours**

## Goal
Build `gold_orders` as an incremental model. Understand when dbt applies the incremental filter and when it rebuilds from scratch.

---

## Part A — Create gold_orders as a Table First

> *Guided — follow along, full solution shown.*

Before going incremental, build `models/gold/gold_orders.sql` as a plain `table` to understand the base query:

```sql
{{ config(materialized='table') }}

SELECT
    order_id,
    order_date,
    order_status,
    order_priority,
    customer_id,
    customer_name,
    market_segment,
    nation_name,
    total_items,
    gross_revenue,
    net_revenue,
    net_revenue / NULLIF(total_items, 0) AS revenue_per_item,
    CURRENT_TIMESTAMP()                  AS loaded_at
FROM {{ ref('silver_orders_enriched') }}
```

Run and verify row count:
```bash
dbt run --select gold_orders
```

```sql
SELECT COUNT(*) FROM ANALYTICS.GOLD.GOLD_ORDERS;
-- Expected: 1,500,000
```

---

## Part B — Convert to Incremental

> *Guided — follow along, full solution shown.*

Change the config to `incremental` and add the filter:

```sql
{{ config(
    materialized = 'incremental',
    unique_key   = 'order_id'
) }}

SELECT
    order_id,
    order_date,
    order_status,
    order_priority,
    customer_id,
    customer_name,
    market_segment,
    nation_name,
    total_items,
    gross_revenue,
    net_revenue,
    net_revenue / NULLIF(total_items, 0) AS revenue_per_item,
    CURRENT_TIMESTAMP()                  AS loaded_at
FROM {{ ref('silver_orders_enriched') }}

{% if is_incremental() %}
    WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})
{% endif %}
```

Run it (first incremental run = full load, same as table):
```bash
dbt run --select gold_orders
```

Run it again immediately:
```bash
dbt run --select gold_orders
```

**Questions:**
1. Why were 0 rows processed on the second run?
2. What does `{{ this }}` resolve to?
3. What does `{% if is_incremental() %}` evaluate to on the very first run?

---

## Part C — Inspect the Compiled SQL

> *Guided — follow along, full solution shown.*

In the Cloud IDE, open `gold_orders.sql` and click **Compile**.

Find the `WHERE` clause in the compiled output. Then switch to Snowflake Query History and find the actual `MERGE` or `INSERT` statement dbt ran. What is the difference between `unique_key` with `MERGE` vs without it?

---

## Part D — Force a Full Refresh

> *Guided — follow along, full solution shown.*

```bash
dbt run --select gold_orders --full-refresh
```

**Questions:**
1. When would you need `--full-refresh` in production?
2. If a bug in `silver_orders_enriched` corrupted data for 3 months, what is the recovery process?

---

## Part E — Run the Full Pipeline

> *Guided — follow along, full solution shown.*

```bash
dbt build --select +gold_orders
```

This runs sources → bronze → silver → gold_orders in dependency order, and tests each step. Confirm all 5 nodes complete successfully.

---

## ✅ Success Criteria

- [ ] `gold_orders` starts as a `table` with 1.5M rows
- [ ] After converting to `incremental`, the second run processes 0 rows
- [ ] You can read the compiled SQL and identify the incremental filter
- [ ] `--full-refresh` rebuilds the full table
- [ ] You can explain what `is_incremental()` returns on first run vs subsequent runs
