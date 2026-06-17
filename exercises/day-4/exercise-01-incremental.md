# Exercise 1 — Incremental Models (Part 2)

**Time: ~1 hour**

---

## Theory Recap

In Day 2 you built `gold_orders` as an incremental model. Before applying the pattern to a new case, a quick summary of how incremental works:

**First run** — the table doesn't exist yet. `is_incremental()` returns `false`. dbt runs the full SELECT and creates the table from scratch.

**Subsequent runs** — `is_incremental()` returns `true`. dbt appends the `WHERE` block to filter only new rows, then merges them into the existing table.

```sql
{% if is_incremental() %}
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

**`unique_key`** — tells dbt to use a `MERGE` instead of a plain `INSERT`. Without it, re-running the model appends duplicates. With it, existing rows get updated and new rows get inserted.

**`--full-refresh`** — ignores `is_incremental()`, drops the table and rebuilds from scratch. Required when you add columns or change the filter logic.

---

## Part A — Create the Incremental Model

> *Guided — follow along, full solution shown.*

Create `models/silver/silver_customers.sql`. This model reads from the seed you loaded in Day 3 and adds a `loaded_at` timestamp so you can track exactly when each row was written:

```sql
{{ config(
    materialized = 'incremental',
    unique_key   = 'customer_id'
) }}

SELECT
    customer_id,
    first_name || ' ' || last_name AS full_name,
    email,
    country,
    account_balance,
    updated_at,
    CURRENT_TIMESTAMP()            AS loaded_at
FROM {{ ref('customers_seed') }}

{% if is_incremental() %}
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

Run it (first run — full load):
```bash
dbt run --select silver_customers
```

Verify all 30 rows loaded and note the `loaded_at` timestamp:
```sql
SELECT COUNT(*), MIN(loaded_at), MAX(loaded_at)
FROM ANALYTICS.SILVER.SILVER_CUSTOMERS;
```

---

## Part B — Add New Customers to the Seed

> *Guided — follow along, full solution shown.*

The instructor has provided `customers_seed_v2.csv` in the Day 4 folder. It contains the original 30 rows plus 5 new customers (IDs 31–35) registered in October 2024.

Replace `seeds/customers_seed.csv` in your dbt project with this file. Then reload the seed:

```bash
dbt seed
```

Verify the seed table now has 35 rows:
```sql
SELECT COUNT(*) FROM ANALYTICS.RAW.CUSTOMERS_SEED;
-- Expected: 35
```

---

## Part C — Run the Incremental Model

> *Practice — write this yourself.*

Run `silver_customers` again — without `--full-refresh`:

```bash
# Your command here
```

Then verify the result with these two queries:

```sql
-- Total rows in the model
SELECT COUNT(*) FROM ANALYTICS.SILVER.SILVER_CUSTOMERS;

-- Only rows loaded in the latest run
SELECT *
FROM ANALYTICS.SILVER.SILVER_CUSTOMERS
WHERE loaded_at = (SELECT MAX(loaded_at) FROM ANALYTICS.SILVER.SILVER_CUSTOMERS)
ORDER BY customer_id;
```

**Questions:**
1. How many rows did this run process? Why exactly that many?
2. What does the `loaded_at` column tell you about the rows that were already in the table?
3. What would happen if you ran it a third time immediately, without changing the seed?

---

## Part D — Updating an Existing Row (MERGE)

> *Guided — follow along, full solution shown.*

So far you've only inserted new rows. But what happens when a row that already exists in the target gets picked up by the filter?

Replace `seeds/customers_seed.csv` with `customers_seed_v3.csv` from the Day 4 folder. This file has the same 35 rows, but James Wilson (customer_id = 1) has a new `account_balance` and an `updated_at` in November 2024:

```bash
dbt seed
```

Verify the seed change:
```sql
SELECT customer_id, account_balance, updated_at
FROM ANALYTICS.RAW.CUSTOMERS_SEED
WHERE customer_id = 1;
-- account_balance = 9500.00, updated_at = 2024-11-01
```

Now run the incremental model:
```bash
dbt run --select silver_customers
```

Check the result:
```sql
-- How many rows total?
SELECT COUNT(*) FROM ANALYTICS.SILVER.SILVER_CUSTOMERS;

-- What happened to customer 1?
SELECT customer_id, full_name, account_balance, updated_at, loaded_at
FROM ANALYTICS.SILVER.SILVER_CUSTOMERS
WHERE customer_id = 1;
```

**Expected:** still 35 rows. Customer 1's `account_balance` is now 9500.00 and `loaded_at` updated. No duplicate.

This is what `unique_key = 'customer_id'` does — dbt generates a `MERGE` statement. When the incoming row shares a key with an existing row, the existing row gets **updated**. Without it, dbt would use `INSERT` and create a duplicate row.

---

## Part E — Full Refresh

> *Practice — write this yourself.*

Run `silver_customers` with `--full-refresh` and observe the difference:

```bash
# Your command here
```

```sql
SELECT MIN(loaded_at), MAX(loaded_at)
FROM ANALYTICS.SILVER.SILVER_CUSTOMERS;
```

**Question:** All 35 rows now have the same `loaded_at`. When would you use `--full-refresh` in a production pipeline?

---

## ✅ Success Criteria

- [ ] `silver_customers` loads 30 rows on the first run
- [ ] After updating the seed, the second run processes exactly 5 new rows
- [ ] `loaded_at` shows two distinct timestamps — original rows vs new rows
- [ ] With `unique_key`, an updated customer row is merged (not duplicated) — still 35 rows
- [ ] `--full-refresh` resets all `loaded_at` to the same value
