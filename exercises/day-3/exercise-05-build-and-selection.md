# Exercise 5 — dbt build & Node Selection

**Time: ~1.5 hours**

## Goal
Master the node selection syntax and understand when to use `dbt build` vs `dbt run + dbt test`. By the end you'll be able to run exactly the right subset of your DAG in any situation.

---

## Part A — dbt build vs dbt run

`dbt run` executes models. `dbt test` runs tests. `dbt build` does both in DAG order — for each node it runs the model first, then immediately tests it before moving to downstream nodes.

```bash
# These two are equivalent (but build is safer):
dbt run && dbt test

dbt build
```

The critical difference: with `dbt build`, if `bronze_tpch_orders` fails its tests, dbt **stops** and won't run `silver_orders_enriched` or `gold_orders` — downstream models never receive bad data.

**Task:** Run both approaches and compare the output:

```bash
# Approach 1
dbt run && dbt test

# Approach 2 — start fresh
dbt build
```

**Questions:**
- In what order does `dbt build` execute nodes?
- What happens to downstream models when an upstream test fails?
- When would you still prefer `dbt run` over `dbt build`?

---

## Part B — Basic Node Selection

```bash
# Single model
dbt build --select bronze_tpch_customers

# All models in a folder (path selector)
dbt build --select models/bronze/

# Shorthand folder selector
dbt build --select bronze.*

# A model and everything upstream of it (+model)
dbt build --select +gold_orders

# A model and everything downstream of it (model+)
dbt build --select bronze_tpch_customers+

# Both parents and children
dbt build --select +gold_orders+
```

**Task:** For each command below, write what you expect it to run *before* executing it, then verify:

1. `dbt build --select silver_orders_enriched+`
2. `dbt build --select +silver_orders_enriched`
3. `dbt build --select bronze.*+`

---

## Part C — Tag Selectors

Add tags to all layers in `dbt_project.yml`:

```yaml
models:
  my_new_project:
    bronze:
      +materialized: view
      +schema: bronze
      +tags: ['bronze', 'daily']
    silver:
      +materialized: view
      +schema: silver
      +tags: ['silver', 'daily']
    gold:
      +materialized: table
      +schema: gold
      +persist_docs:
        relation: true
        columns: true
      +tags: ['gold', 'daily']
```

Now select by tag:

```bash
dbt build --select tag:bronze
dbt build --select tag:gold
dbt build --select tag:daily
```

**Question:** If some models run hourly and some daily, how would you use tags to schedule them differently in dbt Cloud?

---

## Part D — Excluding Nodes

```bash
# Run everything except TPC-H models
dbt build --exclude bronze_tpch_orders bronze_tpch_lineitem

# Run bronze layer but skip a known-broken model
dbt build --select bronze.* --exclude bronze_tpch_lineitem
```

**Task:** You need to run a full pipeline rebuild but skip the incremental `gold_orders` (it takes too long). Write the `dbt build` command.

---

## Part E — Practical Scenarios

Write the correct `dbt build` command for each scenario:

1. **Monday morning full refresh**: rebuild every incremental model from scratch
2. **Hotfix**: only `silver_orders_enriched` changed — run it and all downstream
3. **Bronze-only CI check**: validate the entire bronze layer with tests
4. **Debugging gold**: run `gold_orders` but skip the upstream bronze/silver (assume they're already up to date)
5. **Skip a flaky test temporarily**: run everything, but exclude `assert_payment_matches_order`

---

## ✅ Success Criteria

- [ ] `dbt build` runs the full project and all tests pass
- [ ] You can select by path, tag, graph operator (`+model`, `model+`), and `--exclude`
- [ ] You added a `gold` tag in `dbt_project.yml` and verified `--select tag:gold` works
- [ ] You can write correct commands for all 5 scenarios in Part E
