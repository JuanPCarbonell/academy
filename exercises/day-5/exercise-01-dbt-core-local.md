# Exercise 1 — dbt Core Locally

**Time: ~90 minutes**

## Goal
Run the same project you've been developing in dbt Cloud, but now from your local machine using dbt Core. You'll understand what dbt Cloud abstracts away, when local development makes sense, and how the two environments complement each other.

---

## Part A — Install dbt Core

**Prerequisites — verify before starting:**
```bash
python3 --version   # needs 3.9 or higher
pip --version       # comes with Python
git --version       # needs to be installed
```

If any of these are missing:

- **Python**: download the installer from [python.org/downloads](https://www.python.org/downloads/). On Windows, tick *Add Python to PATH* during installation.
- **pip**: comes bundled with Python 3.4+. If missing, run `python3 -m ensurepip --upgrade`.
- **git**: download the installer from [git-scm.com/downloads](https://git-scm.com/downloads).

```bash
# Create and activate a virtual environment (keeps dbt isolated from system Python)
python3 -m venv dbt-env
source dbt-env/bin/activate        # macOS/Linux
# dbt-env\Scripts\activate         # Windows

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

## Part C — Create the Production Database in Snowflake

Before configuring local profiles, create the `PRODUCTION` database. The `ANALYTICS` database has been used for development all week — production needs its own isolated database.

Run as `ACCOUNTADMIN` in Snowflake:

```sql
USE ROLE ACCOUNTADMIN;

-- Production database
CREATE DATABASE IF NOT EXISTS PRODUCTION;

-- Same schema structure as ANALYTICS
CREATE SCHEMA IF NOT EXISTS PRODUCTION.RAW;
CREATE SCHEMA IF NOT EXISTS PRODUCTION.BRONZE;
CREATE SCHEMA IF NOT EXISTS PRODUCTION.SILVER;
CREATE SCHEMA IF NOT EXISTS PRODUCTION.GOLD;
CREATE SCHEMA IF NOT EXISTS PRODUCTION.SNAPSHOTS;
CREATE SCHEMA IF NOT EXISTS PRODUCTION.DEFAULT;    -- fallback for models without +schema

-- Grant TRANSFORMER full access
GRANT ALL PRIVILEGES ON DATABASE PRODUCTION                   TO ROLE TRANSFORMER;
GRANT ALL PRIVILEGES ON ALL SCHEMAS IN DATABASE PRODUCTION    TO ROLE TRANSFORMER;
GRANT ALL PRIVILEGES ON FUTURE SCHEMAS IN DATABASE PRODUCTION TO ROLE TRANSFORMER;
GRANT ALL PRIVILEGES ON ALL TABLES IN DATABASE PRODUCTION     TO ROLE TRANSFORMER;
GRANT ALL PRIVILEGES ON FUTURE TABLES IN DATABASE PRODUCTION  TO ROLE TRANSFORMER;
GRANT ALL PRIVILEGES ON ALL VIEWS IN DATABASE PRODUCTION      TO ROLE TRANSFORMER;
GRANT ALL PRIVILEGES ON FUTURE VIEWS IN DATABASE PRODUCTION   TO ROLE TRANSFORMER;
```

Verify access:
```sql
USE ROLE TRANSFORMER;
SHOW SCHEMAS IN DATABASE PRODUCTION;
-- Expected: RAW, BRONZE, SILVER, GOLD, SNAPSHOTS, DEFAULT
```

---

## Part D — Configure profiles.yml

This is the file dbt Cloud manages invisibly. Locally, dbt generates it interactively:

```bash
dbt init
```

dbt detects the existing `dbt_project.yml` and walks you through the connection setup. Fill in these values when prompted:

| Field | Value |
|---|---|
| account | your Snowflake account identifier (e.g. `xy12345.eu-west-1`) |
| user | `DBT_USER` |
| password | `<your password>` |
| role | `TRANSFORMER` |
| database | `ANALYTICS` |
| warehouse | `DBT_WH` |
| schema | `default` |

`dbt init` only generates the `dev` target. Open `~/.dbt/profiles.yml` and Add the `prod` target manually **inside `outputs:`**, at the same indentation level as `dev:`

The final file should look like this:

```yaml
default:
  target: dev
  outputs:
    dev:
      type: snowflake
      threads: 4
      account: <YOUR ACCOUNT>
      user: DBT_USER
      database: ANALYTICS
      warehouse: DBT_WH
      schema: DEFAULT
      password: <YOUR PASSWORD>
      role: TRANSFORMER

    prod:
      type: snowflake
      threads: 8
      account: <YOUR ACCOUNT>
      user: DBT_USER
      database: PRODUCTION
      warehouse: DBT_WH
      schema: DEFAULT
      password: <YOUR PASSWORD>
      role: TRANSFORMER
```

---

## Part E — Run the Full Project

```bash
dbt deps                    # install packages (creates dbt_packages/)
dbt seed                    # load CSVs into Snowflake
dbt run                     # build all models
dbt test                    # run all tests
```

Check Snowflake: you should see models in `ANALYTICS.BRONZE, ANALYTICS.SILVER, and ANALYTICS.GOLD`.

---

## Part F — Explore What dbt Cloud Was Hiding

After `dbt build` runs, dbt writes several files to the `target/` folder. First compile the project to generate them:

```powershell
dbt compile
```

Then explore:

```powershell
# Read the compiled SQL — the folder name matches name: in dbt_project.yml
Get-Content "target\compiled\my_new_project\models\gold\gold_orders.sql"

# Run results — timing and status per model
(Get-Content target\run_results.json | ConvertFrom-Json).results | Select-Object unique_id, status, execution_time

# The manifest — all nodes in the DAG
(Get-Content target\manifest.json | ConvertFrom-Json).nodes.PSObject.Properties.Name | Select-Object -First 20
```

**Questions:**
- What does `target/manifest.json` contain? Why is it important for CI?
- What is the difference between `target/compiled/` and `target/run/`?
- In dbt Cloud, where do you find the equivalent of `run_results.json`?

---

## Part G — Local vs Cloud: When to Use Each

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

## Part H — Switch Targets

`profiles.yml` defines multiple targets (`dev` and `prod`). In dbt Cloud each environment is a separate UI configuration. Locally you switch with one flag — no UI, no redeployment:

```bash
# Default: uses the dev target (dbt_dev_local_<your_name> schema)
dbt run --select gold_orders

# Switch to the prod target
dbt run --select gold_orders --target prod
```

Check Snowflake: the prod run should write to the schemas defined in `dbt_project.yml` (bronze, silver, gold) instead of your dev schema.

**Question:** When would you use `--target prod` locally? What risks does it carry?

---

## ✅ Success Criteria

- [ ] `dbt debug` passes all checks locally
- [ ] `dbt build` completes without errors
- [ ] Models appear in `ANALYTICS.BRONZE`, `ANALYTICS.SILVER`, `ANALYTICS.GOLD` in Snowflake
- [ ] You can explain what `target/manifest.json` is used for
- [ ] You can run `dbt run --select gold_orders --full-refresh` and see the effect
