# Exercise 4 — Capstone: Supplier & Parts Analytics

**Time: ~2.5 hours**

## Goal
Apply everything you've learned this week to build a new analytics area **from scratch**, with no step-by-step instructions. You own the design.

---

## The Brief

The Procurement team wants a self-service gold layer to answer these questions:

1. **Which suppliers generate the most revenue across all orders?**
2. **Which part types have the highest discount rates on average?**
3. **What is each supplier's monthly revenue trend?**
4. **Which parts are most frequently ordered, and what is their average order quantity?**
5. **How does supplier performance vary by nation?**

---

## Setup: Add bronze_tpch_parts

TPC-H's `PART` table is your "product catalog". Add it to your bronze layer first.

Create `models/bronze/bronze_tpch_parts.sql`:

```sql
SELECT
    p_partkey       AS part_id,
    p_name          AS part_name,
    p_mfgr          AS manufacturer,
    p_type          AS part_type,
    p_size          AS size,
    p_retailprice   AS retail_price
FROM {{ source('tpch', 'part') }}
```

Add `part` to `tpch_sources.yml`:
```yaml
- name: part
  description: "200K parts catalog"
```

---

## Requirements

### `models/gold/gold_supplier_revenue.sql`
Grain: one row per `supplier_id`.

Must include:
- `supplier_id`, `supplier_name`, `nation_name`
- `total_line_items`: number of line items fulfilled by this supplier
- `total_gross_revenue`: sum of `extended_price`
- `total_net_revenue`: sum of `net_price`
- `avg_discount_rate`: average discount applied
- `revenue_rank`: rank by total_net_revenue globally
- Use `{{ safe_divide(...) }}` for any division

### `models/gold/gold_part_performance.sql`
Grain: one row per `part_id`.

Must include:
- `part_id`, `part_name`, `manufacturer`, `part_type`
- `retail_price`
- `total_orders`: distinct orders containing this part
- `total_units_sold`: sum of quantity
- `total_revenue`: sum of net_price
- `avg_quantity_per_order`
- `avg_discount_rate`
- Surrogate key using `{{ dbt_utils.generate_surrogate_key(['part_id']) }}`

### `models/gold/gold_supplier_monthly_revenue.sql`
Grain: one row per `(supplier_id, ship_year, ship_month)`.

Must include:
- `supplier_id`, `supplier_name`, `nation_name`
- `ship_year`, `ship_month`
- `monthly_net_revenue`
- `monthly_line_items`
- `mom_revenue_change_pct`: month-over-month % change (LAG window function)
- Use `{{ date_dimensions('ship_date') }}` macro or inline equivalents

---

## Testing Requirements

Create `models/gold/schema.yml` entries for all three models. Include at minimum:

- `unique` + `not_null` on every primary key
- `not_null` on all metric columns
- At least one `dbt_utils.expression_is_true` per model (e.g. `"total_net_revenue >= 0"`)
- One singular test: `tests/assert_supplier_revenue_matches_lineitem.sql` — total net revenue in `gold_supplier_revenue` must equal total `net_price` in `bronze_tpch_lineitem` (within rounding)

---

## Documentation Requirements

- Model-level descriptions for all 3 models (grain, business purpose)
- Column descriptions for every metric column
- Add an exposure `procurement_dashboard` in `models/exposures.yml` that depends on all 3 models

---

## Macro Requirements

Use at least:
- `{{ dbt_utils.generate_surrogate_key([...]) }}`
- `{{ safe_divide(...) }}`
- `{{ net_price(...) }}` or `{{ date_dimensions(...) }}`

---

## Delivery

```bash
dbt build --select gold_supplier_revenue+ gold_part_performance+ gold_supplier_monthly_revenue+
```

All models must build and all tests must pass. Then:

```bash
dbt docs generate && dbt docs serve
```

All three models must appear in the DAG with full documentation.

---

## Grading Rubric

| Criterion | Points |
|---|---|
| All 3 models run without errors | 20 |
| Correct grain in each model | 15 |
| All required columns present and correctly computed | 20 |
| All tests pass | 15 |
| Documentation complete (model + column descriptions) | 10 |
| Macros used correctly | 10 |
| Exposure declared | 5 |
| Code readability (CTEs, naming, comments) | 5 |
| **Total** | **100** |

---

## Hints (read only if stuck)

<details>
<summary>Hint 1: gold_supplier_revenue joins</summary>

Start from `bronze_tpch_lineitem` and join `bronze_tpch_suppliers` on `supplier_id`, then join `bronze_tpch_nations` on `nation_id`.

</details>

<details>
<summary>Hint 2: gold_part_performance joins</summary>

Join `bronze_tpch_lineitem` to `bronze_tpch_parts` on `part_id`.

</details>

<details>
<summary>Hint 3: mom_revenue_change_pct</summary>

```sql
LAG(monthly_net_revenue) OVER (
    PARTITION BY supplier_id
    ORDER BY ship_year, ship_month
) AS prev_month_revenue

{{ safe_divide('monthly_net_revenue - prev_month_revenue', 'prev_month_revenue') }} * 100
    AS mom_revenue_change_pct
```

</details>

<details>
<summary>Hint 4: assert_supplier_revenue_matches_lineitem</summary>

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
