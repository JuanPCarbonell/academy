# Exercise 3 — Packages

**Time: ~1.5 hours**

## Goal
Use `dbt_utils` — the most widely used dbt package — to replace manual SQL patterns with battle-tested macros.

---

## Part A — Install dbt_utils

> *Guided — follow along, full solution shown.*

`packages.yml` already includes `dbt_utils`. If you haven't installed it yet:

```bash
dbt deps
```

Verify the package installed:
```bash
ls dbt_packages/dbt_utils/
```

---

## Part B — generate_surrogate_key

> *Guided — follow along, full solution shown.*

TPC-H's `bronze_tpch_lineitem` has a composite primary key (`order_id` + `line_number`). Generate a proper surrogate key:

```sql
SELECT
    {{ dbt_utils.generate_surrogate_key(['order_id', 'line_number']) }} AS lineitem_id,
    order_id,
    line_number,
    ...
FROM {{ source('tpch', 'lineitem') }}
```

Update `bronze_tpch_lineitem.sql` to include `lineitem_id` as the first column.

Run:
```bash
dbt run --select bronze_tpch_lineitem
```

**Question:** What SQL does `generate_surrogate_key` expand to? Use the **Compile** button to find out.

---

## Part C — star()

> *Guided — follow along, full solution shown.*

The `dbt_utils.star()` macro selects all columns from a model except the ones you exclude — useful for avoiding `SELECT *` while still being explicit:

```sql
SELECT
    {{ dbt_utils.star(from=ref('bronze_tpch_orders'), except=["order_priority", "ship_priority"]) }}
FROM {{ ref('bronze_tpch_orders') }}
```

Use this pattern in `silver_orders_enriched.sql` to pull all order fields without listing each one manually.

---

## Part D — date_spine

> *Practice — write this yourself.*

Generate a continuous sequence of dates (useful for filling gaps in time-series data):

Create `models/gold/gold_date_spine.sql`:

```
# Your solution here
```

Run it and verify the row count:
```
# Your solution here
```

```
# Your solution here
```

**Use case:** Join `gold_orders` to `gold_date_spine` on `order_date` to surface days with zero orders — days missing from the orders table entirely.

---

## Part E — expression_is_true (dbt_utils test)

> *Practice — write this yourself.*

`dbt_utils` also adds test helpers. In `models/gold/schema.yml`:

```
# Your solution here
```

Run:
```
# Your solution here
```

---

## ✅ Success Criteria

- [ ] `dbt deps` installs `dbt_utils` without errors
- [ ] `bronze_tpch_lineitem` has a `lineitem_id` surrogate key via `generate_surrogate_key`
- [ ] `dbt_utils.star()` used in at least one model
- [ ] `gold_date_spine` is built with ~2557 rows
- [ ] `expression_is_true` tests pass on `gold_orders`
