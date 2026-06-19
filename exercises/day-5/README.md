# Day 5 — dbt Core, dbt in Snowflake & Capstone

**Estimated time: ~7 hours (full day)**

The final day. You step outside the Cloud IDE and run dbt in two other environments — locally on your machine and natively inside Snowflake. Then you apply everything you've learned this week in an independent capstone challenge.

## Learning objectives
- Run dbt Core locally: install, profiles.yml, CLI workflow
- Understand how local development differs from dbt Cloud
- Run dbt models natively inside Snowflake using the dbt Snowflake Native App
- Configure production jobs and Slim CI in dbt Cloud
- Build a new analytics area independently from scratch

## Exercises

| Exercise | Topic | Time |
|---|---|---|
| [01 — dbt Core Locally](exercise-01-dbt-core-local.md) | Install dbt Core, profiles.yml, CLI vs Cloud IDE | ~90 min |
| [02 — dbt in Snowflake](exercise-02-dbt-in-snowflake.md) | Connect GitHub to Snowflake Workspaces, run models from Snowsight | ~45 min |
| [03 — Jobs & Slim CI](exercise-03-jobs-and-slim-ci.md) | Production job, Slim CI, state-based selection | ~2 h |
| [04 — Capstone](exercise-04-capstone.md) | Build supplier + parts analytics area independently | ~2.5 h |

## Capstone overview

In Exercise 04 you will independently build three gold models using SUPPLIER and PART data from TPC-H — tables you haven't touched yet in the course. No guided solution is provided. You design the pipeline, write the SQL, add tests, and document everything.

Models to build:
- `gold_supplier_revenue` — revenue and rankings by supplier
- `gold_part_performance` — sales volume and revenue by part
- `gold_supplier_monthly_revenue` — monthly revenue trend per supplier with month-over-month change
