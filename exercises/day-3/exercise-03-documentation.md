# Exercise 3 — Documentation

**Time: ~1 hour**

## Goal
Write meaningful descriptions for every model and column, use doc blocks for reusable content, and publish the documentation site.

---

## Part A — Add Descriptions to schema.yml Files

Go through each `schema.yml` file and add a `description` to:
1. Every model
2. Every column

Use real, meaningful descriptions — not "this is the id column." 

Example of a good description vs a bad one:

```yaml
# ❌ Bad
- name: customer_id
  description: "The customer id"

# ✅ Good
- name: customer_id
  description: >
    Surrogate key uniquely identifying a customer account.
    Maps to customer_id in bronze_tpch_customers and is used as
    the join key across all order-related models.
```

Minimum: add descriptions to all columns in `gold_orders` and `gold_customers`.

---

## Part B — Document Business Logic with Doc Blocks

For complex columns, doc blocks let you write longer markdown explanations that can be reused.

Create `models/docs.md`:

```markdown
{% docs customer_tier %}
Segmentation tier based on lifetime net revenue. Platinum: over 500000. Gold: over 100000. Silver: any revenue. No Orders: never purchased.
{% enddocs %}

{% docs net_revenue %}
Total order value in USD after discounts. Sum of net_price across all line items for the order.
{% enddocs %}
```

Reference the doc blocks in `models/gold/schema.yml` — `customer_tier` belongs to `gold_customers` and `net_revenue` belongs to `gold_orders`:

```yaml
# in gold_customers
- name: customer_tier
  description: "{{ doc('customer_tier') }}"

# in gold_orders
- name: net_revenue
  description: "{{ doc('net_revenue') }}"
```

---

## Part C — Generate and Serve Documentation

```bash
dbt docs generate
```

Navigate to:
1. The lineage DAG — find `gold_orders` and trace its full lineage upstream
2. The model detail page for `gold_customers` — verify your descriptions appear
3. The source page for `tpch` — verify the freshness config is shown

**Tasks in the UI:**
- Find a model that has no description — add one
- Find the column with the doc block description — does the markdown table render?
- Click the "expand" button on the DAG to see the full project graph

---

## Part D — Add a Model-level Description

For `gold_orders`, add a plain-text description that will persist cleanly to Snowflake:

```yaml
- name: gold_orders
  description: >
    Fact table with one row per order. Aggregates TPC-H orders with customer,
    nation, and line item data. Order status values: O = open, F = fulfilled,
    P = processing. Materialized as an incremental table, updated on each run.
```

---

## Part E — Persist Docs to Snowflake

dbt can persist column descriptions as Snowflake comments:

Add to your model configs in `dbt_project.yml`:
```yaml
models:
  my_new_project:
    gold:
      +materialized: table
      +persist_docs:
        relation: true
        columns: true
```

Run `dbt run --select gold.*` and then in Snowflake:
```sql
SHOW COLUMNS IN TABLE ANALYTICS.GOLD.GOLD_ORDERS;
```

Do you see the `comment` column populated?

---

## ✅ Success Criteria

- [ ] All models in `bronze/`, `silver/`, `gold/` have descriptions
- [ ] All columns in `gold_orders` and `gold_customers` have descriptions
- [ ] At least 2 doc blocks in `docs.md` and referenced in `schema.yml`
- [ ] `dbt docs serve` shows the full documentation site with lineage graph
- [ ] Column descriptions are persisted in Snowflake (Part E)
