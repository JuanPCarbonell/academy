# Exercise 1 — Generic Tests

**Time: ~2 hours**

## Goal
Add schema tests to your bronze and gold models. Learn how dbt's built-in tests catch data quality issues before they reach analysts.

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
        tests:
          - unique
          - not_null
      - name: customer_id
        tests:
          - not_null
          - relationships:
              to: ref('bronze_tpch_customers')
              field: customer_id
      - name: order_status
        tests:
          - accepted_values:
              values: ['O', 'F', 'P']
      - name: total_price
        tests:
          - not_null

  - name: bronze_tpch_lineitem
    columns:
      - name: order_id
        tests:
          - not_null
          - relationships:
              to: ref('bronze_tpch_orders')
              field: order_id
      - name: quantity
        tests:
          - not_null
      - name: extended_price
        tests:
          - not_null

  - name: bronze_tpch_customers
    columns:
      - name: customer_id
        tests:
          - unique
          - not_null
      - name: market_segment
        tests:
          - accepted_values:
              values: ['AUTOMOBILE', 'BUILDING', 'FURNITURE', 'MACHINERY', 'HOUSEHOLD']

  - name: bronze_tpch_nations
    columns:
      - name: nation_id
        tests:
          - unique
          - not_null
      - name: nation_name
        tests:
          - unique
          - not_null
```

Run the tests:
```bash
dbt test --select bronze.*
```

---

## Part B — Test Your Gold Models

> *Practice — write this yourself.*

Create `models/gold/schema.yml`:

```
# Your solution here
```

Run:
```
# Your solution here
```

---

## Part C — Intentionally Break a Test

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

## Part D — Warn vs Error

> *Guided — follow along, full solution shown.*

Not all test failures should stop the pipeline. Configure a test to warn rather than error:

```yaml
- name: order_status
  tests:
    - accepted_values:
        values: ['O', 'F', 'P']
        severity: warn
```

**Question:** When would you use `severity: warn` vs the default `error`? Give a real-world example of each.

---

## Part E — Run Tests Selectively

> *Guided — follow along, full solution shown.*

```bash
# Test only one model
dbt test --select bronze_tpch_orders

# Test a model and all its children
dbt test --select bronze_tpch_orders+

# Test only relationship tests
dbt test --select bronze_tpch_orders --test-name relationships
```

**Question:** If `bronze_tpch_customers` fails its `unique` test, which downstream models are at risk?

---

## ✅ Success Criteria

- [ ] All bronze models have `not_null` and `unique` on primary keys
- [ ] `bronze_tpch_orders.customer_id` has a relationships test to `bronze_tpch_customers`
- [ ] `order_status` accepted_values test passes with `['O', 'F', 'P']`
- [ ] You intentionally broke the unique test and read the error output
- [ ] You can run tests selectively by model and by test type
