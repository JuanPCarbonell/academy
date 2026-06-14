# Exercise 4 — Incremental Models

**Time: ~1 hour**

## Goal
Build an incremental model on top of TPC-H's `bronze_tpch_lineitem` (6M rows). Understand when incremental materializaton is appropriate and how dbt handles it.

---

## Part A — Why Incremental?

> *Guided — follow along, full solution shown.*

Run this and note how long it takes:

```bash
dbt run --select bronze_tpch_lineitem
```

`bronze_tpch_lineitem` is a view — every downstream query re-reads 6M rows from `SNOWFLAKE_SAMPLE_DATA`. For a real pipeline landing millions of rows daily, you don't want to reprocess old data on every run.

Incremental models solve this: on the first run they build the full table; on subsequent runs they only process new rows.

---

## Part B — Create an Incremental Silver Model

> *Guided — follow along, full solution shown.*

Create `models/silver/silver_lineitem_daily.sql`:

```sql
{{ config(materialized='incremental', unique_key='order_id || \'-\' || line_number') }}

SELECT
    order_id,
    line_number,
    part_id,
    supplier_id,
    quantity,
    extended_price,
    discount_rate,
    net_price,
    return_flag,
    ship_date,
    ship_mode
FROM {{ ref('bronze_tpch_lineitem') }}

{% if is_incremental() %}
    WHERE ship_date > (SELECT MAX(ship_date) FROM {{ this }})
{% endif %}
```

Run it for the first time (full build):
```bash
dbt run --select silver_lineitem_daily
```

Note how long it takes. Then run it again:
```bash
dbt run --select silver_lineitem_daily
```

**Questions:**
1. Why was the second run faster?
2. What does `{{ this }}` refer to in dbt?
3. What does `{% if is_incremental() %}` do on the very first run?

---

## Part C — Force a Full Refresh

> *Practice — write this yourself.*

```
# Your solution here
```

**Question:** When would you need `--full-refresh` in production? Give two scenarios.

---

## Part D — Aggregated Incremental Model

> *Guided — follow along, full solution shown.*

Create `models/gold/gold_daily_revenue.sql` — a daily revenue summary:

```sql
{{ config(materialized='incremental', unique_key='ship_date') }}

SELECT
    ship_date,
    COUNT(DISTINCT order_id)    AS order_count,
    SUM(net_price)              AS net_revenue,
    SUM(extended_price)         AS gross_revenue,
    AVG(discount_rate)          AS avg_discount_rate
FROM {{ ref('silver_lineitem_daily') }}
{% if is_incremental() %}
    WHERE ship_date > (SELECT MAX(ship_date) FROM {{ this }})
{% endif %}
GROUP BY 1
```

Run it:
```bash
dbt run --select gold_daily_revenue
```

Query the top 10 revenue days:
```sql
SELECT ship_date, net_revenue
FROM ANALYTICS.GOLD.GOLD_DAILY_REVENUE
ORDER BY net_revenue DESC
LIMIT 10;
```

---

## Part E — Compare Materializations

> *Guided — follow along, full solution shown.*

Run the same aggregation as a view and a table and compare:

```bash
# Set bronze_tpch_lineitem as view (default) and query it
dbt run --select bronze_tpch_lineitem

# Set silver_lineitem_daily as table and query it
dbt run --select silver_lineitem_daily --full-refresh
```

Open a Snowflake worksheet and run the gold_daily_revenue SELECT manually against each upstream option. Observe query times using Snowflake's **Query History**.

**Question:** Summarize when you'd use `view`, `table`, and `incremental` for a model that processes 10M new rows per day.

---

## ✅ Success Criteria

- [ ] `silver_lineitem_daily` builds as an incremental table
- [ ] Second run is faster than the first (or skips rows if ship_date filter applies)
- [ ] `--full-refresh` rebuilds the entire table
- [ ] `gold_daily_revenue` aggregates by ship_date correctly
- [ ] You can explain what `{{ this }}` and `is_incremental()` do
