# Day 4 — Incremental, Macros, Packages & Snapshots

**Estimated time: ~7 hours (full day)**

Today you cover advanced dbt features: incremental models with real data mutation, reusable Jinja macros, the dbt package ecosystem, and SCD Type 2 history tables with snapshots.

## Learning objectives
- Demonstrate incremental load, MERGE behavior, and deduplication
- Write parametrized Jinja macros to eliminate SQL duplication
- Use `dbt-utils` and understand the package ecosystem
- Build SCD Type 2 history tables using snapshots

## Exercises

| Exercise | Topic | Time |
|---|---|---|
| [01 — Incremental (Part 2)](exercise-01-incremental.md) | Incremental model on customers_seed, MERGE, dedup | ~1 h |
| [02 — Macros](exercise-02-macros.md) | Write reusable macros, replace copy-paste SQL | ~2.5 h |
| [03 — Packages](exercise-03-packages.md) | dbt-utils, generate_surrogate_key, date_spine | ~1.5 h |
| [04 — Snapshots](exercise-04-snapshots.md) | SCD Type 2 history on the customers_seed table | ~2 h |

## Prerequisite

Exercise 01 and 04 require the `customers_seed` table to exist in `ANALYTICS.RAW`. If you didn't complete Day 3 Exercise 04, run `dbt seed` before starting today.
