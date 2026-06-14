# Exercise 2 — Singular Tests

**Time: ~1.5 hours**

## Goal
Write custom SQL tests that encode business rules too complex for generic tests.

---

## Part A — Your First Singular Test

> *Guided — follow along, full solution shown.*

Singular tests live in the `tests/` folder and are plain SQL files that return rows when the assertion fails — zero rows means the test passes.

Create `tests/assert_positive_net_revenue.sql`:

```sql
-- Every order's net_revenue should be greater than zero.
-- Returns failing rows.
SELECT
    order_id,
    net_revenue
FROM {{ ref('gold_orders') }}
WHERE net_revenue IS NULL
   OR net_revenue <= 0
```

Run it:
```bash
dbt test --select assert_positive_net_revenue
```

---

## Part B — Cross-Model Consistency Test

> *Practice — write this yourself.*

Create `tests/assert_lineitem_revenue_matches_orders.sql`:

This test validates that the sum of `net_price` in `bronze_tpch_lineitem` per order matches `net_revenue` in `gold_orders` (within a small rounding tolerance):

```
# Your solution here
```

Run it:
```
# Your solution here
```

If rows are returned, investigate: is it a rounding issue or a join problem?

---

## Part C — Referential Integrity Test

> *Practice — write this yourself.*

Create `tests/assert_all_orders_have_customers.sql`:

```
# Your solution here
```

---

## Part D — Parameterize a Singular Test with Variables

> *Guided — follow along, full solution shown.*

dbt variables let you make tests configurable. Create `tests/assert_minimum_order_revenue.sql`:

```sql
-- Orders below a minimum revenue threshold are suspicious.
-- Override threshold with: dbt test --vars '{"min_order_revenue": 1000}'
SELECT
    order_id,
    net_revenue
FROM {{ ref('gold_orders') }}
WHERE net_revenue < {{ var('min_order_revenue', 0.01) }}
```

Run with the default:
```bash
dbt test --select assert_minimum_order_revenue
```

Run with a custom threshold:
```bash
dbt test --select assert_minimum_order_revenue --vars '{"min_order_revenue": 500}'
```

**Question:** How many orders fall below $500 net revenue?

---

## ✅ Success Criteria

- [ ] `assert_positive_net_revenue` passes (zero rows returned)
- [ ] `assert_lineitem_revenue_matches_orders` passes or you can explain any discrepancies found
- [ ] `assert_all_orders_have_customers` passes
- [ ] `assert_minimum_order_revenue` works with and without the `--vars` override
