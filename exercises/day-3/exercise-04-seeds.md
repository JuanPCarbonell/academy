# Exercise 4 — Seeds

**Time: ~1 hour**

## Goal

Understand what dbt seeds are, when to use them, and how to configure them. You will load `customers_seed` — the 30-row mutable customer table that you will use tomorrow in the snapshots exercise — and explore the mechanics of seed configuration.

---

## Part A — What Are Seeds?

> *Guided — follow along, full solution shown.*

Seeds are CSV files that dbt manages as tables in your warehouse. They are meant for **small, static or slowly-changing reference data** that makes sense to version-control alongside your SQL — things like status mappings, country codes, or test fixtures.

Seeds are **not** for large datasets, data that updates frequently, or anything that originates in a source system (use `{{ source() }}` for that).

The instructor has provided `customers_seed.csv` in the Day 3 exercise folder — copy it into the `seeds/` folder of your dbt project. Open it and inspect the columns:

```
customer_id, first_name, last_name, email, country, account_balance, updated_at
```

This table has 30 rows — a small, controlled dataset you can mutate directly in Snowflake to simulate real-world profile changes. That's exactly what makes it useful for the snapshots exercise tomorrow.

---

## Part B — Configure the RAW Schema

> *Guided — follow along, full solution shown.*

Seeds don't go to `BRONZE` — they represent data that lands directly in the warehouse, like an ingestion tool would. The convention is to put them in `RAW`.

Open `dbt_project.yml` and add the seeds block below `models:`. Include `column_types` to avoid that dbt store everything as `VARCHAR`:

```yaml
seeds:
  my_new_project:
    +schema: raw
    +column_types:
      customer_id: integer
      account_balance: float
```

Without `column_types`, dbt infers all CSV columns as `VARCHAR` — which will cause type mismatch errors tomorrow when the snapshot joins `customer_id` as a string against integer keys.

To also persist descriptions to Snowflake as column comments, add `persist_docs`:

```yaml
seeds:
  my_new_project:
    +schema: raw
    +column_types:
      customer_id: integer
      account_balance: float
    +persist_docs:
      relation: true
      columns: true
```

With `persist_docs`, run `dbt seed --full-refresh` — Snowflake needs to recreate the table to apply the comments.

---

## Part C — Load the Seed

> *Guided — follow along, full solution shown.*

From the dbt Cloud IDE terminal, run:

```bash
dbt seed
```

dbt reads every CSV in `seeds/`, creates a table for each one, and populates it. The table lands in the `RAW` schema you just configured.

Verify it loaded:
```sql
SELECT * FROM ANALYTICS.RAW.CUSTOMERS_SEED LIMIT 5;
```

Check column types:
```sql
SHOW COLUMNS IN TABLE ANALYTICS.RAW.CUSTOMERS_SEED;
```

`customer_id` should be `NUMBER` and `account_balance` should be `FLOAT` — not `TEXT`.

**Questions:**
1. Which schema did the seed land in? Why `RAW` and not `BRONZE`?
2. What happens if you run `dbt seed` a second time without `--full-refresh`? Try it.

---

## Part D — Add a seeds.yml

> *Guided — follow along, full solution shown.*

Create `seeds/seeds.yml` to add descriptions — the column types are already handled by `dbt_project.yml`:

```yaml
version: 2

seeds:
  - name: customers_seed
    description: "30 customer profiles used for the SCD Type 2 snapshot exercise on Day 4."
    columns:
      - name: customer_id
        description: "Primary key."
      - name: account_balance
        description: "Current account balance in USD."
      - name: updated_at
        description: "Timestamp of the last profile update. Used by the snapshot strategy to detect changes."
```

Run `dbt seed` again and verify nothing broke.

---

## Part E — When NOT to Use Seeds

> *Practice — write this yourself.*

Seeds have a narrow, well-defined use case. For each scenario below, decide: **seed**, **source**, or **neither** — and write one sentence of reasoning.

1. A 3-row CSV you maintain in git mapping order status codes (`O`, `F`, `P`) to readable labels
2. A list of 200 internal cost centers, updated quarterly by Finance and delivered as a CSV
3. Currency exchange rates that an Airflow pipeline loads into Snowflake every morning from a third-party API
4. 3 years of historical sales transactions sitting in a 10M-row CSV on a local machine

---

## ✅ Success Criteria

- [ ] `customers_seed` table exists in `ANALYTICS.RAW`
- [ ] `d