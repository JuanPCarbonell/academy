# Exercise 3 — Incremental Models

**Time: ~2.5 hours**

## Goal
Build `gold_orders` as an incremental model. Understand when dbt applies the incremental filter and when it rebuilds from scratch.

---

## Part A — Add a loaded_at Timestamp

> *Guided — follow along, full solution shown.*

You already have `gold_orders` as a table from Exercise 1. Before converting it to incremental, add one column: `loaded_at`. This timestamp records when each row was last written — essential for auditing and for the incremental filter you'll add next.

Open `models/gold/gold_orders.sql` and add `CURRENT_TIMESTAMP() AS loaded_at` at the end of the SELECT:

```sql
{{ config(materialized='table') }}

SELECT
    *,
    gross_revenue - net_revenue                                     AS discount_amount,
    ROUND(
        (gross_revenue - net_revenue) / NULLIF(gross_revenue, 0),
        4
    )                                                               AS effective_discount_rate,
    net_revenue / NULLIF(total_items, 0)                            AS revenue_per_item,
    CASE
        WHEN total_items >= 5 THEN 'Large'
        WHEN total_items >= 3 THEN 'Medium'
        ELSE 'Small'
    END                                                             AS order_size_band,
    CURRENT_TIMESTAMP()                                             AS loaded_at
FROM {{ ref('silver_orders_enriched') }}
```

Run:
```bash
dbt run --select gold_orders
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
    *,
    gross_revenue - net_revenue                                     AS discount_amount,
    ROUND(
        (gross_revenue - net_revenue) / NULLIF(gross_revenue, 0),
        4
    )                                                               AS effective_discount_rate,
    net_revenue / NULLIF(total_items, 0)                            AS revenue_per_item,
    CASE
        WHEN total_items >= 5 THEN 'Large'
        WHEN total_items >= 3 THEN 'Medium'
        ELSE 'Small'
    END                                                             AS order_size_band,
    CURRENT_TIMESTAMP()                                             AS loaded_at
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

Open `gold_orders.sql` in the Cloud IDE and click **Compile**. Look at the compiled output.

**Questions:**
1. The compiled SQL contains a `WHERE` clause even though you didn't write one. Where does it come from and when does dbt include it?
2. The log shows dbt ran a `MERGE` statement. What would happen if you removed `unique_key = 'order_id'` from the config and the same order appeared in two consecutive runs?

---

## Part D — Force a Full Refresh

> *Guided — follow along, full solution shown.*

```bash
dbt run --select gold_orders --full-refresh
```

**Questions:**
1. Your filter uses `WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})`. You change the `order_size_band` thresholds and run the model without `--full-refresh`. Which rows get the new band values?
2. A source system sent orders with backdated `order_date` values — 2 weeks in the past. Will the incremental filter pick them up? What do you do?

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
