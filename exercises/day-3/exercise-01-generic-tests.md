# Exercise 1 — Generic Tests

**Time: ~2 hours**

## Goal
Add data tests to your sources, bronze, silver, and gold layers. Learn how dbt's built-in tests catch quality issues before they reach analysts.

---

## Part A — Test Your Bronze Models

> *Guided — follow along, full solution shown.*

Create `models/bronze/schema.yml`:

```yaml
version: 2

models:
  - name: bronze_tpch_orders
    columns:
      - name: order_id
        data_tests:
          - unique
          - not_null
      - name: customer_id
        data_tests:
          - not_null
          - relationships:
              arguments:
                to: ref('bronze_tpch_customers')
                field: customer_id
      - name: order_status
        data_tests:
          - accepted_values:
              arguments:
                values: ['O', 'F', 'P']
      - name: total_price
        data_tests:
          - not_null

  - name: bronze_tpch_lineitem
    columns:
      - name: order_id
        data_tests:
          - not_null
          - relationships:
              arguments:
                to: ref('bronze_tpch_orders')
                field: order_id
      - name: quantity
        data_tests:
          - not_null
      - name: extended_price
        data_tests:
          - not_null

  - name: bronze_tpch_customers
    columns:
      - name: customer_id
        data_tests:
          - unique
          - not_null
      - name: market_segment
        data_tests:
          - accepted_values:
              arguments:
                values: ['AUTOMOBILE', 'BUILDING', 'FURNITURE', 'MACHINERY', 'HOUSEHOLD']

  - name: bronze_tpch_nations
    columns:
      - name: nation_id
        data_tests:
          - unique
          - not_null
      - name: nation_name
        data_tests:
          - unique
          - not_null
```

Run the tests:
```bash
dbt test --select bronze.*
```

---

## Part B — Test Your Sources

> *Practice — write this yourself.*

In Day 1 you created `models/bronze/tpch_sources.yml` declaring the TPC-H tables. Now extend it with `data_tests` — sources support the same tests as models and run directly against the raw tables, before any bronze transformation.

You'll need to add `columns:` blocks to the tables that don't have them yet. Add tests to at least these columns:

| Table | Column | Tests |
|-------|--------|-------|
| orders | `o_orderkey` | unique, not_null |
| orders | `o_orderstatus` | accepted_values: `['O', 'F', 'P']` |
| orders | `o_custkey` | not_null |
| lineitem | `l_orderkey` | not_null |
| lineitem | `l_linenumber` | not_null |
| customer | `c_custkey` | unique, not_null |
| nation | `n_nationkey` | unique, not_null |

Run:
```bash
dbt test --select source:tpch
```

**Question:** These tests run against the raw source tables, not your bronze models. If a source test fails but the equivalent bronze test passes, what does that tell you?

---

## Part C — Test Your Silver Models

> *Guided — follow along, full solution shown.*

Create `models/silver/schema.yml`. Silver has one testable model — `silver_orders_enriched` (`silver_lineitem_totals` is ephemeral and never materializes, so there's nothing to test against).

```yaml
version: 2

models:
  - name: silver_orders_enriched
    columns:
      - name: order_id
        data_tests:
          - unique
          - not_null
      - name: customer_id
        data_tests:
          - not_null
          - relationships:
              arguments:
                to: ref('bronze_tpch_customers')
                field: customer_id
      - name: order_status
        data_tests:
          - accepted_values:
              arguments:
                values: ['O', 'F', 'P']
      - name: market_segment
        data_tests:
          - accepted_values:
              arguments:
                values: ['AUTOMOBILE', 'BUILDING', 'FURNITURE', 'MACHINERY', 'HOUSEHOLD']
      - name: total_items
        data_tests:
          - not_null
      - name: gross_revenue
        data_tests:
          - not_null
      - name: net_revenue
        data_tests:
          - not_null
```

Run the tests:
```bash
dbt test --select silver.*
```

**Question:** `silver_orders_enriched` has a `relationships` test pointing to `bronze_tpch_customers`. If that test fails, what does it tell you about the join logic in the model?

---

## Part D — Test Your Gold Models

> *Practice — write this yourself.*

Create `models/gold/schema.yml`. Think about which tests make sense at this layer:

- Primary keys should still be `unique` and `not_null`
- `customer_tier` in `gold_customers` has a fixed set of values: `'Platinum'`, `'Gold'`, `'Silver'`, `'No Orders'`
- `order_size_band` in `gold_orders` has a fixed set of values: `'Large'`, `'Medium'`, `'Small'`
- Revenue and aggregation columns should never be null

```
# Your solution here
```

Run:
```bash
dbt test --select gold.*
```

---

## Part E — Intentionally Break a Test

> *Guided — follow along, full solution shown.*

In `bronze_tpch_orders.sql`, add a UNION that introduces a duplicate `order_id`:

```sql
SELECT * FROM {{ source('tpch', 'orders') }}
UNION ALL
SELECT * FROM {{ source('tpch', 'orders') }} LIMIT 1
```

Run:
```bash
dbt run --select bronze_tpch_orders
dbt test --select bronze_tpch_orders
```

Read the error output carefully. dbt tells you exactly how many records failed and shows you the query that found them.

Fix it (remove the UNION), re-run, confirm tests pass.

---

## Part F — Warn vs Error

> *Guided — follow along, full solution shown.*

Not all failures should stop the pipeline. In `models/bronze/schema.yml`, change `order_status`'s accepted values to only `['O', 'F']` — intentionally excluding `'P'` (pending orders) — and set severity to `warn`:

```yaml
- name: order_status
  data_tests:
    - accepted_values:
        arguments:
          values: ['O', 'F']
        config:
          severity: warn
```

Run `dbt test --select bronze_tpch_orders` and observe how warnings appear in the log — the test finds the 'P' rows, reports how many failed, but the overall command exits with success instead of error.

After verifying, restore `'P'` to the values list so the test is correct going forward.

**Question:** When would you use `severity: warn` vs the default `error`? Give a concrete example of each from a real analytics project.

---

## Part G — Run Tests Selectively

> *Guided — follow along, full solution shown.*

```bash
# Test only one model
dbt test --select bronze_tpch_orders

# Test a model and all its downstream children
dbt test --select bronze_tpch_orders+

# Test only one specific test by name across all models
dbt test --select test_name:relationships
```

**Question:** If `bronze_tpch_customers` fails its `unique` test, which downstream models are at risk? How would you identify them without running everything?

---

## ✅ Success Criteria

- [ ] All bronze models have `not_null` and `unique` on primary keys
- [ ] `sources.yml` has `data_tests` on raw TPC-H columns
- [ ] `silver_orders_enriched` has unique + not_null on `order_id` and a relationships test on `customer_id`
- [ ] `models/gold/schema.yml` tests primary keys, accepted values, and not_null on numeric columns
- [ ] You intentionally broke the unique test and read the error output
- [ ] You configured at least one test with `severity: warn`
- [ ] You can run tests selectively by model, layer, and test type
