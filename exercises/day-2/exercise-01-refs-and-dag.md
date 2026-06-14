# Exercise 1 — Refs & the DAG

**Time: ~2.5 hours**

## Goal
Build the silver and gold layers using `{{ ref() }}`, so dbt manages execution order automatically. You'll join TPC-H's five bronze tables into analytics-ready models.

---

## Part A — Add Silver and Gold to dbt_project.yml

> *Guided — follow along, full solution shown.*

You're about to create models in two new layers. Before you do, tell dbt where to put them. Open `dbt_project.yml` and add `silver` and `gold` under the existing `bronze` config:

```yaml
models:
  my_new_project:
    bronze:
      +materialized: view
      +schema: bronze
    silver:
      +materialized: view
      +schema: silver
    gold:
      +materialized: table
      +schema: gold
```

This will create two new schemas in Snowflake — `SILVER` and `GOLD` — the first time you run models in those folders.

---

## Part B — Create silver_orders_enriched

> *Guided — follow along, full solution shown.*

Create `models/silver/silver_orders_enriched.sql`.

This model joins orders, customers, line items, and nations into one enriched row per order.

**Requirements:**
- Use `{{ ref('bronze_tpch_orders') }}`, `{{ ref('bronze_tpch_customers') }}`, `{{ ref('bronze_tpch_lineitem') }}`, `{{ ref('bronze_tpch_nations') }}`
- One row per order
- Include: `order_id`, `order_date`, `order_status`, `order_priority`
- Include: `customer_id`, `customer_name`, `market_segment`, `nation_name`
- Include: `total_items` — count of line items per order
- Include: `gross_revenue` — sum of `extended_price` across line items
- Include: `net_revenue` — sum of `net_price` (after discount) across line items

**Hint:** Aggregate `bronze_tpch_lineitem` before joining:

```sql
WITH line_totals AS (
    SELECT
        order_id,
        COUNT(*)            AS total_items,
        SUM(extended_price) AS gross_revenue,
        SUM(net_price)      AS net_revenue
    FROM {{ ref('bronze_tpch_lineitem') }}
    GROUP BY 1
),
orders AS (
    SELECT * FROM {{ ref('bronze_tpch_orders') }}
),
customers AS (
    SELECT * FROM {{ ref('bronze_tpch_customers') }}
),
nations AS (
    SELECT * FROM {{ ref('bronze_tpch_nations') }}
)

SELECT
    o.order_id,
    o.order_date,
    o.order_status,
    o.order_priority,
    o.customer_id,
    c.customer_name,
    c.market_segment,
    n.nation_name,
    l.total_items,
    l.gross_revenue,
    l.net_revenue
FROM orders o
LEFT JOIN customers  c ON o.customer_id = c.customer_id
LEFT JOIN nations    n ON c.nation_id   = n.nation_id
LEFT JOIN line_totals l ON o.order_id   = l.order_id
```

Run it:
```bash
dbt run --select silver_orders_enriched
```

---

## Part C — Understand Execution Order

> *Guided — follow along, full solution shown.*

```bash
# Run silver_orders_enriched and all its parents
dbt run --select +silver_orders_enriched

# Run silver_orders_enriched and all its children
dbt run --select silver_orders_enriched+
```

**Questions:**
1. What models ran when you used `+silver_orders_enriched`?
2. Why does dbt know the correct execution order without you specifying it?
3. What happens if you try to run `silver_orders_enriched` before any of its bronze parents exist?

---

## Part D — Create gold_customers

> *Practice — write this yourself.*

Create `models/gold/gold_customers.sql`.

**Requirements:**
- Use `{{ ref('bronze_tpch_customers') }}`, `{{ ref('bronze_tpch_nations') }}`, `{{ ref('silver_orders_enriched') }}`
- One row per customer
- Include: `customer_id`, `customer_name`, `market_segment`, `nation_name`
- Add `total_orders`: count of orders
- Add `total_revenue`: sum of `net_revenue`
- Add `avg_order_value`: total_revenue / total_orders (handle divide-by-zero)
- Add `first_order_date` and `last_order_date`
- Add `customer_tier`:
  - `'Platinum'` if `total_revenue > 500000`
  - `'Gold'` if `total_revenue > 100000`
  - `'Silver'` if `total_revenue > 0`
  - `'No Orders'` otherwise

---

## Part E — Create gold_orders

> *Practice — write this yourself.*

Create `models/gold/gold_orders.sql`.

**Requirements:**
- Use `{{ ref('silver_orders_enriched') }}`
- One row per order, all fields from silver
- Add `revenue_per_item`: `net_revenue / total_items`
- Materialize as `table` using an in-file config:

```
# Your solution here
```

**Question:** Why does it make sense to materialize gold models as tables rather than views?

---

## Part F — Inspect the Full DAG

> *Guided — follow along, full solution shown.*

In the dbt Cloud IDE, open `gold_customers` and click the **lineage** tab.

Verify the upstream chain includes all 4 bronze models flowing through silver to gold:
```
tpch.orders    → bronze_tpch_orders    ↘
tpch.customer  → bronze_tpch_customers  → silver_orders_enriched → gold_customers
tpch.nation    → bronze_tpch_nations   ↗
tpch.lineitem  → bronze_tpch_lineitem  ↗
```

**Question:** If TPC-H renamed `c_nationkey` to `c_nation_id`, which models would break downstream? How would you identify them without running everything?

---

## Part G — Circular Dependency

> *Guided — follow along, full solution shown.*

Add `{{ ref('gold_customers') }}` inside `bronze_tpch_customers.sql` and click **Compile**. What error does dbt return? Remove it immediately after.

This is a **circular dependency** — dbt cannot build a DAG that contains a cycle. Unlike SQL, where you'd only discover this at runtime, dbt catches it at compile time before anything runs.

---

## ✅ Success Criteria

- [ ] `silver_orders_enriched` joins 4 bronze models and has correct columns
- [ ] `gold_customers` has correct aggregations and `customer_tier`
- [ ] `gold_orders` is materialized as a table
- [ ] `dbt run --select +gold_customers` runs the full chain without errors
- [ ] The DAG shows full lineage from TPC-H sources through to gold