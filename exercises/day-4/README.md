# Day 4 — Snapshots, Macros & Packages

**Estimated time: ~7 hours (full day)**

Today you tackle the advanced features: tracking historical changes with snapshots, writing reusable Jinja macros, leveraging the dbt package ecosystem, and building incremental models on large tables.

## Learning objectives
- Build SCD Type 2 history tables using snapshots
- Write parametrized Jinja macros to eliminate SQL duplication
- Use `dbt-utils` and understand the package ecosystem
- Build an incremental model on TPC-H's 6M-row lineitem table

## Exercises

| Exercise | Topic | Time |
|---|---|---|
| [01 — Snapshots](exercise-01-snapshots.md) | SCD Type 2 history on the customers_seed table | ~2 h |
| [02 — Macros](exercise-02-macros.md) | Write reusable macros, replace copy-paste SQL | ~2.5 h |
| [03 — Packages](exercise-03-packages.md) | dbt-utils, generate_surrogate_key, date_spine | ~1.5 h |
| [04 — Incremental on LINEITEM](exercise-04-incremental.md) | Incremental model on 6M-row fact table | ~1 h |

## Prerequisite

Exercise 01 requires the `customers_seed` table to exist in `ANALYTICS.RAW`. If you didn't complete Day 3 Exercise 04, run `dbt seed` before starting today.
