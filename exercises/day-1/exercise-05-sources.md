# Exercise 5 — Sources

**Time: ~60 minutes**

## Goal

You hardcoded `FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS` directly in your SQL. Declare TPC-H as a **dbt source** instead, so those raw tables appear in the lineage graph, get freshness checks, and can be renamed in one place. You'll also complete the bronze layer with the three remaining TPC-H tables.

---

## Part A — Declare TPC-H as a Source

> *Guided — follow along, full solution shown.*

Create `models/bronze/tpch_sources.yml`:

```yaml
version: 2

sources:
  - name: tpch
    database: SNOWFLAKE_SAMPLE_DATA
    schema: TPCH_SF1
    description: "TPC-H benchmark dataset — read-only, built into every Snowflake account"

    tables:
      - name: orders
        description: "1.5M order headers"
        config:
          loaded_at_field: "o_orderdate::timestamp"
          freshness:
            warn_after: {count: 1, period: day}
            error_after: {count: 30, period: day}
        columns:
          - name: o_orderkey
            description: "Primary key"
          - name: o_custkey
            description: "Foreign key to customer"

      - name: lineitem
        description: "6M line items — one row per part per order"
        # no config block — dbt skips freshness for this table automatically

      - name: customer
        description: "150K customer accounts"

      - name: nation
        description: "25 country reference rows"

      - name: supplier
        description: "10K suppliers"
```

---

## Part B — Update Existing Models to Use source()

> *Guided — follow along, full solution shown.*

Open `bronze_tpch_orders.sql` and `bronze_tpch_lineitem.sql`. Replace the hardcoded paths:

```sql
-- ❌ Before: hardcoded, invisible to dbt's DAG
FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS

-- ✅ After: tracked by dbt, appears in lineage
FROM {{ source('tpch', 'orders') }}
```

Apply the same change to `bronze_tpch_lineitem.sql` using `{{ source('tpch', 'lineitem') }}`.

Run to confirm nothing broke:
```bash
dbt run --select bronze_tpch_orders bronze_tpch_lineitem
```

Now open the **Lineage** tab in the IDE. You should see the TPC-H source nodes appearing upstream of your bronze models — they were invisible before.

**Question:** What would happen to all models that use `{{ source('tpch', 'orders') }}` if the source schema changed from `TPCH_SF1` to `TPCH_SF10`? How many files would you need to edit?

---

## Part C — Add bronze_tpch_customers

> *Guided — follow along, full solution shown.*

Create `models/bronze/bronze_tpch_customers.sql`:

```sql
SELECT
    c_custkey                   AS customer_id,
    c_name                      AS customer_name,
    c_nationkey                 AS nation_id,
    c_phone                     AS phone,
    c_acctbal                   AS account_balance,
    c_mktsegment                AS market_segment
FROM {{ source('tpch', 'customer') }}
```

Run it:
```bash
dbt run --select bronze_tpch_customers
```

---

## Part D — Add bronze_tpch_nations and bronze_tpch_suppliers

> *Practice — write this yourself.*

**`models/bronze/bronze_tpch_nations.sql`** — rename all columns to readable English. Source: `{{ source('tpch', 'nation') }}`.

**`models/bronze/bronze_tpch_suppliers.sql`** — rename all columns to readable English. Source: `{{ source('tpch', 'supplier') }}`.

Refer to the TPC-H table descriptions in the Day 1 README if you need the column names.

Run all bronze models to confirm everything works:
```bash
dbt run --select bronze.*
```

---

## Part E — Check Source Freshness

> *Guided — follow along, full solution shown.*

```bash
dbt source freshness
```

Only `orders` has a `loaded_at_field` configured, so dbt only checks freshness for that table. The rest are skipped automatically — no `freshness: null` needed.

`orders` will immediately warn (and eventually error) because TPC-H is static benchmark data — the last `o_orderdate` is from 1998. That's expected and intentional: the point is to see how the thresholds work in practice.

**Question:** In a real pipeline, when would you want `error_after: {count: 1, period: hour}` on a source? Give a concrete example.

---

## ✅ Success Criteria

- [ ] `tpch_sources.yml` declares all 5 TPC-H tables under the `tpch` source
- [ ] `bronze_tpch_orders` and `bronze_tpch_lineitem` use `{{ source('tpch', ...) }}` — no hardcoded paths
- [ ] All 5 bronze models exist and `dbt run --select bronze.*` completes without errors
- [ ] TPC-H source nodes appear in the lineage graph upstream of bronze models
- [ ] `dbt source freshness` runs and shows expected warnings for static data
