# Exercise 4 — Snapshots

**Time: ~2 hours**

## Goal
Use dbt snapshots to capture SCD Type 2 history. Because TPC-H is read-only, this exercise uses `customers_seed` — a small table you can mutate directly in Snowflake.

---

## Part A — Load the Seed Table

> *Guided — follow along, full solution shown.*

This exercise needs the original 30-row seed. Run with `--full-refresh` and the original `customers_seed.csv` (not v2 or v3) to start clean:

```bash
dbt seed --full-refresh
```

Verify:
```sql
SELECT COUNT(*) FROM ANALYTICS.RAW.CUSTOMERS_SEED;
-- Expected: 30
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
    *
FROM {{ ref('customers_seed') }}

{% endsnapshot %}
```

`customers_seed` is a seed managed by dbt — no source declaration needed. Reference it directly with `ref()`.

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

## Part C — Simulate Profile Changes

> *Guided — follow along, full solution shown.*

Run three updates to simulate real-world profile changes. Both the changed column and `updated_at` must be updated — otherwise the snapshot won't detect the change:

```sql
-- Customer 1 moved to Germany
UPDATE ANALYTICS.RAW.CUSTOMERS_SEED
SET country    = 'Germany',
    updated_at = CURRENT_TIMESTAMP()
WHERE customer_id = 1;

-- Customer 2 changed email
UPDATE ANALYTICS.RAW.CUSTOMERS_SEED
SET email      = 'sofia.garcia.new@email.com',
    updated_at = CURRENT_TIMESTAMP()
WHERE customer_id = 2;

-- Customer 3 updated their account balance
UPDATE ANALYTICS.RAW.CUSTOMERS_SEED
SET account_balance = 9999.99,
    updated_at      = CURRENT_TIMESTAMP()
WHERE customer_id = 3;
```

Run the snapshot:
```bash
dbt snapshot
```

Verify dbt detected exactly 3 changes:
```sql
SELECT customer_id, dbt_valid_from, dbt_valid_to
FROM ANALYTICS.SNAPSHOTS.CUSTOMERS_SNAPSHOT
WHERE dbt_valid_to IS NOT NULL
ORDER BY customer_id;
-- Expected: 3 rows with non-NULL dbt_valid_to (the closed versions)
```

**Questions:**
1. How many total rows does the snapshot now have?
2. What is `dbt_valid_to` on the old rows?
3. How would you query "what country was customer 1 in on a specific past date"?

---

## Part D — Query Historical Data

> *Guided — follow along, full solution shown.*

Now that you have real history from Part C, run these queries and explain what each one returns.

**1. Full history of customers that changed:**
```sql
SELECT customer_id, first_name, country, email, account_balance,
       dbt_valid_from, dbt_valid_to
FROM ANALYTICS.SNAPSHOTS.CUSTOMERS_SNAPSHOT
WHERE customer_id IN (1, 2, 3)
ORDER BY customer_id, dbt_valid_from;
```

**2. State of all customers before the updates (point-in-time):**

Replace `<timestamp>` with a timestamp from before you ran the updates in Part C — you can find it in `dbt_valid_from` of the original rows:

```sql
SELECT customer_id, first_name, country, email, account_balance
FROM ANALYTICS.SNAPSHOTS.CUSTOMERS_SNAPSHOT
WHERE <timestamp> >= dbt_valid_from
  AND (dbt_valid_to > <timestamp> OR dbt_valid_to IS NULL)
ORDER BY customer_id;
-- Expected: 30 rows — the original state of all customers
```

**3. Current state only:**
```sql
SELECT customer_id, first_name, country, email, account_balance
FROM ANALYTICS.SNAPSHOTS.CUSTOMERS_SNAPSHOT
WHERE dbt_valid_to IS NULL
ORDER BY customer_id;
-- Expected: 30 rows — one current version per customer
```

**Questions:**
1. Query 1 returns 6 rows for customers 1, 2, and 3. Why 6 and not 3?
2. Query 2 returns the original values for all 3 changed customers. What does that prove about snapshots?
3. Query 3 always returns exactly 30 rows regardless of how many changes were made. Why?

---

## Part E — Check Strategy

> *Practice — write this yourself.*

The `timestamp` strategy relies on `updated_at` being reliably maintained. Not all source systems guarantee this. The `check` strategy detects changes by comparing column values directly on every snapshot run.

Create `snapshots/customers_snapshot_check.sql` using the `check` strategy, watching the `country` and `email` columns:

```sql
-- Hint: replace strategy = 'timestamp' and updated_at with:
--   strategy   = 'check'
--   check_cols = ['country', 'email']
```

Run it:
```bash
dbt snapshot --select customers_snapshot_check
```

Now update customer 2's email **without touching `updated_at`** — this is the key difference from the `timestamp` strategy:


Run the snapshot again:
```bash
dbt snapshot --select customers_snapshot_check
```

Verify it detected the change:
```sql
SELECT customer_id, email, dbt_valid_from, dbt_valid_to
FROM ANALYTICS.SNAPSHOTS.CUSTOMERS_SNAPSHOT_CHECK
WHERE customer_id = 2
ORDER BY dbt_valid_from;
-- Expected: 2 rows — one closed version, one current
```

**Question:** What is the main trade-off between `timestamp` and `check`? When would you use each?

---

## ✅ Success Criteria

- [ ] `customers_snapshot` creates the history table with the 4 dbt columns
- [ ] After updating customer 1's country, the snapshot has 2 rows for that customer
- [ ] Old row has a non-NULL `dbt_valid_to`; current row has NULL
- [ ] `customers_snapshot_check` detects a column-level change without relying on `updated_at`
- [ ] You can explain when to use `timestamp` vs `check`
