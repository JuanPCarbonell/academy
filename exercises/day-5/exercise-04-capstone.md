# Exercise 4 — Capstone: Supplier & Parts Analytics

**Time: ~2.5 hours**

---

## Context

The Procurement team currently runs ad-hoc SQL queries to understand which suppliers and parts drive the most value. They need a proper analytics layer they can connect to a dashboard — consistent, tested, and documented.

Your job is to build that layer from scratch using the TPC-H `SUPPLIER`, `PART`, and `LINEITEM` tables. You already have `LINEITEM`, `ORDERS`, `SUPPLIER`, and `NATION` in bronze. The only table missing is `PART`.

This exercise applies every practice from the week: layered architecture, macros, packages, tests, and documentation. There are no step-by-step instructions — only requirements, the business questions to answer, and criteria for what good looks like.

---

## Business Questions to Answer

The three gold models you build must collectively answer:

1. Which suppliers generate the most net revenue, and how do they rank globally?
2. Which parts sell the most units, and what is their average discount rate?
3. How does each supplier's monthly revenue evolve over time — is it growing or shrinking?

---

## Step 1 — Add the Missing Bronze Model

`PART` is the product catalog — 200,000 parts with name, type, manufacturer, and retail price. Without it you cannot answer the parts question.

Add `models/bronze/bronze_tpch_parts.sql` following the same conventions as every other bronze model in the project: rename raw columns to readable names, use `{{ source() }}`, no transformations.

Then register the source in `tpch_sources.yml`:
```yaml
- name: part
  description: "TPC-H parts catalog — 200K rows, one row per part."
```

**Why:** Bronze models are the single point of truth for raw data. If you skip this and join directly from source in a silver or gold model, you break the lineage and duplicate the renaming logic.

---

## Step 2 — Build a Silver Intermediate Model

Create `models/silver/silver_supplier_sales.sql`.

This model joins `bronze_tpch_lineitem` with `bronze_tpch_suppliers` and `bronze_tpch_nations` to produce a clean, enriched fact table at line-item grain with readable supplier and nation names resolved. It should also carry `ship_date`, `ship_year`, and `ship_month` so the gold monthly model doesn't need to extract dates itself.

Use `net_price` and `extended_price` from `bronze_tpch_lineitem` (already computed in bronze). Use CTEs — no nested subqueries.

**Why:** Gold models should aggregate, not join. If both `gold_supplier_revenue` and `gold_supplier_monthly_revenue` join to suppliers and nations independently, you're duplicating join logic. Silver is the right place to resolve it once.

---

## Step 3 — Build the Gold Models

### `models/gold/gold_supplier_revenue.sql`
One row per supplier. Aggregates total revenue, line item count, average discount, and ranks suppliers globally by net revenue. Use `{{ revenue_tier() }}` to classify each supplier — override the macro defaults since supplier revenue is on a much larger scale than the order-level defaults (`low=1000000, high=10000000`).

### `models/gold/gold_part_performance.sql`
One row per part. Joins `bronze_tpch_lineitem` with `bronze_tpch_parts` to aggregate total units sold, distinct orders, total revenue, and average discount per part. Use `{{ dbt_utils.generate_surrogate_key(['part_id']) }}` as the surrogate key — `part_id` is a natural key from TPC-H but the pattern of using surrogate keys consistently applies here too.

### `models/gold/gold_supplier_monthly_revenue.sql`
One row per `(supplier_id, ship_year, ship_month)`. Built from `silver_supplier_sales`. Aggregates monthly net revenue and line item count per supplier, and computes month-over-month revenue change using a LAG window function partitioned by `supplier_id` and ordered chronologically.

---

## Step 4 — Test Everything

Create or extend `models/gold/schema.yml` with entries for all three models.

Minimum requirements per model:
- `unique` + `not_null` on the primary key (or the combination of columns that defines uniqueness)
- `not_null` on every metric column
- At least one `dbt_utils.expression_is_true` that validates a business rule — examples: total net revenue must be positive, revenue rank must be between 1 and the total number of suppliers, avg discount rate must be between 0 and 1

Also write one singular test in `tests/assert_supplier_revenue_matches_lineitem.sql`: the sum of `total_net_revenue` across all suppliers in `gold_supplier_revenue` must equal the sum of `net_price` across all rows in `bronze_tpch_lineitem`. If they don't match, your aggregation has a bug.

**Why:** A gold model without tests is just a view someone will trust blindly. The singular test is a cross-model consistency check — the kind of validation that catches fan-out joins (accidental row duplication) before they reach a dashboard.

---

## Step 5 — Document Everything

Add to `schema.yml`:
- A model-level `description` for each gold model — state the grain and business purpose in one sentence
- Column-level `description` for every metric column (not for IDs and names, those are self-explanatory)

Add the procurement exposure to `models/exposures.yml`:
```yaml
- name: procurement_dashboard
  label: "Supplier & Parts Analytics"
  type: dashboard
  maturity: low
  description: >
    Supplier revenue ranking, parts performance, and monthly revenue trends.
    Built in the capstone — connects to the Procurement team's BI tool.
  depends_on:
    - ref('gold_supplier_revenue')
    - ref('gold_part_performance')
    - ref('gold_supplier_monthly_revenue')
  owner:
    name: "Procurement Team"
    email: "procurement@example.com"
```

---

## Delivery

```bash
dbt build --select bronze_tpch_parts silver_supplier_sales gold_supplier_revenue gold_part_performance gold_supplier_monthly_revenue
```

All models must build and all tests must pass. Then generate docs:
```bash
dbt docs generate
```

Open the docs site from the **Artifacts** tab in dbt Cloud and verify:
- All three gold models appear in the DAG connected to their sources
- `procurement_dashboard` appears as a terminal node
- Each model has a description and metrics have column descriptions

---

## Criteria for Good Work

**Architecture:** bronze renames only, silver resolves joins, gold aggregates. No joins in gold that should be in silver.

**SQL style:** CTEs with descriptive names, not nested subqueries. Columns named for the business, not the database.

**Macros:** `revenue_tier`, `net_price`, and `generate_surrogate_key` used where they add clarity — not forced in where they don't fit.

**Tests:** every primary key tested for uniqueness and null, every metric tested for validity, singular test passes.

**Documentation:** someone who didn't write the model can understand its grain and purpose from the description alone.

---

## Hints (read only if stuck)

<details>
<summary>Hint 1: silver_supplier_sales structure</summary>

```sql
WITH lineitem AS (
    SELECT * FROM {{ ref('bronze_tpch_lineitem') }}
),
suppliers AS (
    SELECT * FROM {{ ref('bronze_tpch_suppliers') }}
),
nations AS (
    SELECT * FROM {{ ref('bronze_tpch_nations') }}
)
SELECT
    l.order_id,
    l.line_number,
    l.supplier_id,
    s.supplier_name,
    n.nation_name,
    l.part_id,
    l.quantity,
    l.extended_price,
    l.discount_rate,
    l.net_price,
    l.ship_date,
    YEAR(l.ship_date)  AS ship_year,
    MONTH(l.ship_date) AS ship_month
FROM lineitem l
JOIN suppliers s ON l.supplier_id = s.supplier_id
JOIN nations   n ON s.nation_id   = n.nation_id
```

</details>

<details>
<summary>Hint 2: gold_supplier_revenue structure</summary>

Aggregate from `silver_supplier_sales`, group by `supplier_id`, `supplier_name`, `nation_name`. Use `RANK() OVER (ORDER BY total_net_revenue DESC)` for the revenue rank.

</details>

<details>
<summary>Hint 3: mom_revenue_change_pct</summary>

```sql
LAG(monthly_net_revenue) OVER (
    PARTITION BY supplier_id
    ORDER BY ship_year, ship_month
) AS prev_month_revenue,

ROUND(
    (monthly_net_revenue - prev_month_revenue) / NULLIF(prev_month_revenue, 0) * 100,
    2
) AS mom_revenue_change_pct
```

</details>

<details>
<summary>Hint 4: singular test</summary>

```sql
WITH lineitem_total AS (
    SELECT SUM(net_price) AS total FROM {{ ref('bronze_tpch_lineitem') }}
),
supplier_total AS (
    SELECT SUM(total_net_revenue) AS total FROM {{ ref('gold_supplier_revenue') }}
)
SELECT ABS(l.total - s.total) AS discrepancy
FROM lineitem_total l, supplier_total s
WHERE ABS(l.total - s.total) > 1.0
```

</details>
