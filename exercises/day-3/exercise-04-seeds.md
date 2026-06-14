# Exercise 4 — Seeds

**Time: ~1 hour**

## Goal

Understand what dbt seeds are, when to use them, and how to configure them. You will load `customers_seed` — the 30-row mutable customer table that you will use tomorrow in the snapshots exercise — and explore the mechanics of seed configuration.

---

## Part A — What Are Seeds?

> *Guided — follow along, full solution shown.*

Seeds are CSV files that dbt manages as tables in your warehouse. They are meant for **small, static or slowly-changing reference data** that makes sense to version-control alongside your SQL — things like status mappings, country codes, or test fixtures.

Seeds are **not** for large datasets, data that updates frequently, or anything that originates in a source system (use `{{ source() }}` for that).

The `customers_seed.csv` file lives in `seeds/`. Open it and inspect the columns:

```
customer_id, first_name, last_name, email, country, updated_at
```

This table has 30 rows — a small, controlled dataset you can mutate directly in Snowflake to simulate real-world profile changes. That's exactly what makes it useful for the snapshots exercise tomorrow.

---

## Part B — Configure the RAW Schema

> *Guided — follow along, full solution shown.*

Seeds don't go to `BRONZE` — they represent data that lands directly in the warehouse, like an ingestion tool would. The convention is to put them in `RAW`.

Open `dbt_project.yml` and add the seeds configuration below the existing `models:` block:

```yaml
seeds:
  my_new_project:
    +schema: raw
```

This tells dbt to materialize all seeds into the `RAW` schema. You only need to add this once — any future seeds will also land here automatically.

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

You should see 30 rows with customer profiles including `updated_at` timestamps.

**Questions:**
1. Which schema did the seed land in? Why `RAW` and not `BRONZE`?
2. What happens if you run `dbt seed` a second time without `--full-refresh`? Try it.

---

## Part D — Inspect the seeds.yml Configuration

> *Guided — follow along, full solution shown.*

Open `seeds/seeds.yml`. Notice:

```yaml
seeds:
  - name: customers_seed
    description: "30 customer profiles used for SCD Type 2 snapshot exercises."
    columns:
      - name: customer_id
        description: "Primary key"
      - name: updated_at
        description: "Last profile update timestamp — used by the timestamp snapshot strategy"
```

By default, dbt infers all column types as `VARCHAR`. You can override this with `column_types` in the config block:

```yaml
  - name: customers_seed
    config:
      column_types:
        customer_id: integer
        account_balance: float
```

**Question:** Open the seed table in Snowflake and check the actual column types dbt chose. Is `customer_id` stored as `VARCHAR` or `INTEGER`? Does it matter for this exercise?

---

## Part E — When NOT to Use Seeds

> *Practice — write this yourself.*

Seeds have a narrow, well-defined use case. For each scenario below, decide: **seed**, **source**, or **neither** — and write one sentence of reasoning.

1. A CSV with 50 currency exchange rates, updated every business day
2. A list of 200 internal cost centers, updated quarterly by Finance
3. A 10-row table mapping order status codes (`O`, `F`, `P`) to readable labels
4. 3 years of historical sales transactions exported from a legacy ERP

---

## ✅ Success Criteria

- [ ] `customers_seed` table exists in `ANALYTICS.RAW`
- [ ] `dbt seed` runs without errors
- [ ] You can explain when to use seeds vs sources
- [ ] `seeds:` block with `+schema: raw` is configured in `dbt_project.yml`