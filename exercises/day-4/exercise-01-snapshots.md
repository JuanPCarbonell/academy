# Exercise 1 — Snapshots

**Time: ~2 hours**

## Goal
Use dbt snapshots to capture SCD Type 2 history. Because TPC-H is read-only, this exercise uses a small seed table (`customers_seed`) that students can mutate directly in Snowflake.

---

## Part A — Load the Seed Table

> *Guided — follow along, full solution shown.*

From the dbt Cloud IDE terminal:

```bash
dbt seed
```

This loads `customers_seed.csv` (30 rows) into `ANALYTICS.RAW.CUSTOMERS_SEED`.

Verify:
```sql
SELECT * FROM ANALYTICS.RAW.CUSTOMERS_SEED LIMIT 5;
```

---

## Part B — Create the Customer Snapshot

> *Guided — follow along, full solution shown.*

Create `snapshots/customers_snapshot.sql`:

```sql
{% snapshot customers_snapshot %}

{{ config(
    target_schema = 'snapshots',
    unique_key    = 'customer_id',
    strategy      = 'timestamp',
    updated_at    = 'updated_at',
) }}

SELECT
    customer_id,
    first_name,
    last_name,
    email,
    country,
    updated_at
FROM {{ source('raw', 'customers_seed') }}

{% endsnapshot %}
```

Add the source declaration to a new file `models/bronze/seed_sources.yml`:

```yaml
version: 2

sources:
  - name: raw
    database: ANALYTICS
    schema: raw
    tables:
      - name: customers_seed
        description: "Mutable customer profiles used for snapshot exercises"
```

Run the snapshot:
```bash
dbt snapshot
```

Inspect the result:
```sql
SELECT * FROM ANALYTICS.SNAPSHOTS.CUSTOMERS_SNAPSHOT ORDER BY customer_id;
```

Notice the four dbt-added columns: `dbt_scd_id`, `dbt_updated_at`, `dbt_valid_from`, `dbt_valid_to`.

**Questions:**
1. How many rows does the snapshot have right now?
2. What is `dbt_valid_to` for all current rows?
3. What is `dbt_scd_id`? How is it generated?

---

## Part C — Simulate a Profile Change

> *Guided — follow along, full solution shown.*

Update customer 1 — they moved to a new country:

```sql
UPDATE ANALYTICS.RAW.CUSTOMERS_SEED
SET country    = 'DE',
    updated_at = CURRENT_TIMESTAMP()
WHERE customer_id = 1;
```

Run the snapshot again:
```bash
dbt snapshot
```

Query the history:
```sql
SELECT customer_id, country, dbt_valid_from, dbt_valid_to
FROM ANALYTICS.SNAPSHOTS.CUSTOMERS_SNAPSHOT
WHERE customer_id = 1
ORDER BY dbt_valid_from;
```

**Questions:**
1. How many rows does customer 1 now have?
2. What is `dbt_valid_to` on the old row?
3. How would you query "what country was customer 1 in on a specific past date"?

---

## Part D — Query Historical Data

> *Guided — follow along, full solution shown.*

Write a query that shows what country each customer was in at the time of their TPC-H order. Save as `analyses/customer_country_at_order_time.sql`:

```sql
SELECT
    o.order_id,
    o.order_date,
    s.customer_id,
    s.country AS country_at_time_of_order
FROM {{ ref('gold_orders') }} o
JOIN ANALYTICS.SNAPSHOTS.CUSTOMERS_SNAPSHOT s
  ON o.customer_id  = s.customer_id
 AND o.order_date BETWEEN s.dbt_valid_from AND COALESCE(s.dbt_valid_to, CURRENT_TIMESTAMP())
```

Compile it:
```bash
dbt compile --select customer_country_at_order_time
```

Note: This join uses the seed's 30 customer IDs — only orders whose `customer_id` falls in 1–30 will match. That's fine for the exercise.

---

## Part E — Check Strategy

> *Practice — write this yourself.*

The `check` strategy detects changes by comparing column values instead of a timestamp:

```
# Your solution here
```

Create and run this second snapshot.

**Question:** What is the difference between `timestamp` and `check` strategies? Which requires an `updated_at` column? Which is safer if source systems don't maintain reliable update timestamps?

---

## ✅ Success Criteria

- [ ] `dbt seed` loads `customers_seed` into Snowflake
- [ ] `customers_snapshot` creates the history table with the 4 dbt columns
- [ ] After updating customer 1's country, the snapshot has 2 rows for that customer
- [ ] Old row has a non-NULL `dbt_valid_to`; current row has NULL
- [ ] `customers_snapshot_check` runs using the `check` strategy
- [ ] You can explain when to use `timestamp` vs `check`
