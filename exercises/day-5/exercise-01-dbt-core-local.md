# Exercise 1 — dbt Core Locally

**Time: ~90 minutes**

## Goal
Run the same project you've been developing in dbt Cloud, but now from your local machine using dbt Core. You'll understand what dbt Cloud abstracts away, when local development makes sense, and how the two environments complement each other.

---

## Part A — Install dbt Core

```bash
# Install dbt with the Snowflake adapter
pip install dbt-snowflake

# Verify
dbt --version
```

---

## Part B — Clone the Project

```bash
git clone https://github.com/<your_username>/dbt-academy.git
cd dbt-academy
```

---

## Part C — Configure profiles.yml

This is the file dbt Cloud manages for you invisibly. Locally, you have to create it yourself.

```bash
# Create the dbt home directory if it doesn't exist
mkdir -p ~/.dbt
```

Create `~/.dbt/profiles.yml`:

```yaml
my_new_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: <your_account>         # e.g. xy12345.eu-west-1
      user: DBT_USER
      password: "{{ env_var('DBT_PASSWORD') }}"
      role: TRANSFORMER
      database: ANALYTICS
      warehouse: DBT_WH
      schema: dbt_dev_local_<your_name>   # fallback schema for models without a custom schema
      threads: 4

    prod:
      type: snowflake
      account: <your_account>
      user: DBT_USER
      password: "{{ env_var('DBT_PASSWORD') }}"
      role: TRANSFORMER
      database: ANALYTICS
      warehouse: DBT_WH
      schema: gold                        # production schemas come from dbt_project.yml
      threads: 8
```

```bash
export DBT_PASSWORD="your_password_here"
dbt debug   # should print: All checks passed!
```

Notice: the `prod` target uses a higher thread count. In dbt Cloud, this is configured per environment in the UI — same concept, different interface.

---

## Part D — Run the Full Project

```bash
dbt deps                    # install packages (creates dbt_packages/)
dbt seed                    # load CSVs into Snowflake
dbt run                     # build all models
dbt test                    # run all tests
```

Or in a single command:

```bash
dbt build
```

Check Snowflake: you should see models in `ANALYTICS.BRONZE, ANALYTICS.SILVER, and ANALYTICS.GOLD`.

---

## Part E — Explore What dbt Cloud Was Hiding

Locally you can see files that dbt Cloud manages transparently:

```bash
# The compiled SQL dbt sends to Snowflake
cat target/compiled/dbt_academy/models/gold/gold_orders.sql

# The run results with timing and row counts
cat target/run_results.json | python3 -m json.tool | head -60

# The manifest — the full DAG as JSON (used by Slim CI)
cat target/manifest.json | python3 -m json.tool | grep '"unique_id"' | head -20
```

**Questions:**
- What does `target/manifest.json` contain? Why is it important for CI?
- What is the difference between `target/compiled/` and `target/run/`?
- In dbt Cloud, where do you find the equivalent of `run_results.json`?

---

## Part F — Local vs Cloud: When to Use Each

| Scenario | Local (dbt Core) | dbt Cloud IDE |
|----------|-----------------|---------------|
| Writing a new model | ✓ (your editor, git, autocomplete) | ✓ (browser, lineage visible) |
| Running the full project in prod | — | ✓ (scheduled jobs) |
| Debugging a failing model | ✓ (`dbt run --select model` in terminal) | ✓ (same, in IDE terminal) |
| CI/CD on pull requests | Needs external setup | ✓ (built-in Slim CI) |
| Sharing docs with stakeholders | Manual | ✓ (hosted docs site) |
| Working offline | ✓ | — |
| Advanced Jinja debugging | ✓ (`dbt compile`, inspect files) | ✓ (Compile button in IDE) |

**Discussion:** Your team has 5 data engineers and 10 analysts. Which parts of the workflow would you run locally vs in dbt Cloud? Why?

---

## Part G — Override a Variable from the CLI

One advantage of local development: you can override project variables without changing any files.

```bash
# Override the start_date var defined in dbt_project.yml
dbt run --select gold_orders --vars '{"start_date": "2024-01-01"}'

# Run with full-refresh to rebuild an incremental model from scratch
dbt run --select gold_orders --full-refresh
```

Try both. Check the row count difference in Snowflake.

---

## ✅ Success Criteria

- [ ] `dbt debug` passes all checks locally
- [ ] `dbt build` completes without errors
- [ ] Models appear in `ANALYTICS.BRONZE`, `ANALYTICS.SILVER`, `ANALYTICS.GOLD` in Snowflake
- [ ] You can explain what `target/manifest.json` is used for
- [ ] You can run `dbt run --select gold_orders --full-refresh` and see the effect
