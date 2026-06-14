# Exercise 4 — Your First Models

**Time: ~90 minutes**

## Goal
Write your first dbt models using data that is already in your Snowflake account — no loading required. You'll use the TPC-H benchmark dataset (`SNOWFLAKE_SAMPLE_DATA.TPCH_SF1`) which is built into every Snowflake account.

This exercise teaches the most fundamental concept of the bronze layer: **rename and normalize ugly source columns into a clean, consistent schema** that the rest of the pipeline can rely on.

---

## Part A — Run the Example Models

> *Guided — follow along, full solution shown.*

Before writing any SQL, let's run the example models that dbt Cloud already generated for you. Open the file tree and look inside `models/example/` — you'll see two files: `my_first_dbt_model.sql` and `my_second_dbt_model.sql`. Open each one and read them briefly.

Notice that `my_second_dbt_model` references `my_first_dbt_model` using `{{ ref('my_first_dbt_model') }}`. This is how dbt models depend on each other.

Now run them:

```bash
dbt build
```

`dbt build` runs seeds, models, and tests in one command (in dependency order). Watch the output — you'll see `my_first_dbt_model` run, then its test fail, and then **`my_second_dbt_model` never runs**. This is intentional: dbt ships the example models with a `not_null` test on a column that deliberately contains a `null` value. It's designed to show you what a test failure looks like before you cause one yourself.

The default severity is `error` — a failed test blocks everything downstream. `my_first_dbt_model` itself got materialized (the `run` already happened before the test), but `my_second_dbt_model` was skipped entirely.

Read the error output carefully — dbt tells you exactly which test failed, on which model, and on which column.

Open `models/example/schema.yml` and find the test that failed. If you wanted the build to continue despite the failure, you would add `severity: warn` to that test. We'll cover this in depth on Day 3.

Now look at the **Lineage** tab at the bottom of the IDE. You'll see a small DAG: `my_first_dbt_model → my_second_dbt_model`. This is dbt's dependency graph.

**Questions:**
- `my_first_dbt_model` has a `{{ config() }}` block at the top, `my_second_dbt_model` doesn't. What does the config block do, and what happens to a model that doesn't have one?
- What does `{{ ref() }}` do that a plain SQL `FROM` doesn't?
- What would you add to the test definition in `schema.yml` to make the build continue even when the test fails?

Once you've explored the examples, you can delete the `models/example/` folder — you won't need it.

---

## Part B — Configure the Project Structure

> *Guided — follow along, full solution shown.*

Before creating any real models, configure dbt to build them into the right schemas (`BRONZE`, `SILVER`, `GOLD`) instead of the default development schema.

### Step 1 — Override the schema naming macro

By default dbt appends the layer name as a suffix to your dev schema (e.g. `DBT_DEV_ALICE_BRONZE`). To get clean schema names instead, create `macros/generate_schema_name.sql`:

```sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- if custom_schema_name is none -%}
        {{ target.schema }}
    {%- else -%}
        {{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

This tells dbt: if a model has a custom schema defined, use that name exactly. If not (like the example models), use the default dev schema.

### Step 2 — Configure the bronze schema in dbt_project.yml

Open `dbt_project.yml` and add the bronze configuration under the existing `models:` section:

```yaml
models:
  my_new_project:
    bronze:
      +materialized: view
      +schema: bronze
```

You'll add `silver` and `gold` in later days when you actually need them.

Now create the folder structure in the file tree: `models/bronze/`.

---

## Part C — Explore the Source Data

> *Guided — follow along, full solution shown.*

Before writing any SQL, open a Snowflake worksheet and inspect the raw data:

```sql
-- What does the raw orders table look like?
SELECT * FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS LIMIT 5;

-- What does the raw line items table look like?
SELECT * FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.LINEITEM LIMIT 5;
```

Notice the column names: `O_ORDERKEY`, `L_EXTENDEDPRICE`, `L_SUPPKEY`. This is typical of real-world source systems — abbreviated, prefixed, sometimes cryptic. The bronze layer's job is to fix this before anyone downstream has to deal with it.

---

## Part D — bronze_tpch_orders

> *Guided — follow along, full solution shown.*

In the dbt Cloud IDE, create `models/bronze/bronze_tpch_orders.sql`:

```sql
SELECT
    o_orderkey                          AS order_id,
    o_custkey                           AS customer_id,
    o_orderstatus                       AS order_status,
    o_totalprice                        AS total_price,
    o_orderdate                         AS order_date,
    o_orderpriority                     AS order_priority,
    o_shippriority                      AS ship_priority
FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS
```

Run it in the IDE Commands panel:

```bash
dbt run --select bronze_tpch_orders
```

Then verify in Snowflake:

```sql
SELECT * FROM ANALYTICS.BRONZE.BRONZE_TPCH_ORDERS LIMIT 5;
```

The columns are now readable English — `order_id`, `customer_id`, `total_price`. That's the whole point of the bronze layer.

---

## Part E — bronze_tpch_lineitem

> *Practice — write this yourself.*

Create `models/bronze/bronze_tpch_lineitem.sql`. Rename all columns to readable English and add a computed column:

- `net_price`: `l_extendedprice * (1 - l_discount)`

When done, run it:

```bash
dbt run --select bronze_tpch_lineitem
```

**Question:** This table has 6 million rows. Did it take longer to run than `bronze_tpch_orders`? Why or why not? (Hint: what materialization is the bronze layer using?)

---

## Part F — Verify the Materialization

> *Guided — follow along, full solution shown.*

Your bronze models are configured as `view` in `dbt_project.yml`. Verify this in Snowflake:

```sql
SHOW OBJECTS IN SCHEMA ANALYTICS.BRONZE;
```

Look at the `kind` column — you should see `VIEW`, not `TABLE`.

You can also use the **Compile** button in the IDE to see how dbt resolves the Jinja templating: open `bronze_tpch_orders.sql` and click **Compile**. You'll see the `{{ ref() }}` calls replaced by the actual schema and table names that dbt resolved for your environment.

**Questions:**
- What is the difference between a `VIEW` and a `TABLE` in this context?
- The project default is `table`. Under what circumstances would you prefer `view` for bronze models instead? And when would `table` be the better choice?
- What would you change in `dbt_project.yml` to switch bronze models to `view`?

---

## Part G — Run Both Models and Break One

> *Guided — follow along, full solution shown.*

```bash
dbt run --select bronze.*
```

Now deliberately introduce an error — misspell a column name in `bronze_tpch_orders.sql` and run it again:

```bash
dbt run --select bronze_tpch_orders
```

Read the error message carefully. dbt shows you the exact SQL that failed and the Snowflake error. Fix it, re-run, confirm it passes.

---

## ✅ Success Criteria

- [ ] `bronze_tpch_orders` view exists in Snowflake with readable column names
- [ ] `bronze_tpch_lineitem` view exists with computed `net_price` column
- [ ] `dbt run --select bronze.*` runs both models without errors