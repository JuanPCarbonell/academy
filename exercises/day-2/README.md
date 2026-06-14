# Day 2 — Refs, Materializations & Incremental

**Estimated time: ~7 hours (full day)**

You finished Day 1 with a complete bronze layer — all 5 TPC-H tables declared as sources and materialized as views. Today you build the rest of the pipeline: silver joins, gold aggregations, and an incremental fact table.

## Learning objectives
- Use `{{ ref() }}` to build model dependencies and let dbt manage execution order
- Understand all four materializations and when to use each
- Build an incremental model on TPC-H's 6M-row lineitem table

## Exercises

| Exercise | Topic | Time |
|---|---|---|
| [01 — Refs & DAG](exercise-01-refs-and-dag.md) | Build silver + gold layers, inspect the lineage graph | ~2.5 h |
| [02 — Materializations](exercise-02-materializations.md) | View vs table vs ephemeral — compare on real data | ~2 h |
| [03 — Incremental](exercise-03-incremental.md) | Build an incremental model on the lineitem table | ~2.5 h |

## What you'll build today

By end of day you will have a complete medallion pipeline:

```
TPC-H sources
  └── bronze_tpch_orders
  └── bronze_tpch_lineitem
  └── bronze_tpch_customers
  └── bronze_tpch_nations
  └── bronze_tpch_suppliers
        └── silver_orders_enriched  (joins all bronze → one row per order)
              └── gold_orders        (incremental fact table)
              └── gold_customers     (customer aggregations + tier)
              └── gold_nations_summary (revenue by country)
```
