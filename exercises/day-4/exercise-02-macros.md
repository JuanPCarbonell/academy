# Exercise 2 — Macros

**Time: ~2.5 hours**

## Goal
Identify repeated SQL patterns in your TPC-H models and extract them into reusable Jinja macros.

---

## Part A — Your First Macro: discount_amount

> *Guided — follow along, full solution shown.*

You've written `l_extendedprice * l_discount` (and variations of it) multiple times across models. Create a macro for it.

Create `macros/discount_amount.sql`:

```sql
{% macro discount_amount(price_col, discount_col, precision=2) %}
    ROUND({{ price_col }} * {{ discount_col }}, {{ precision }})
{% endmacro %}
```

And a companion macro `macros/net_price.sql` for net price:

```sql
{% macro net_price(price_col, discount_col, precision=2) %}
    ROUND({{ price_col }} * (1 - {{ discount_col }}), {{ precision }})
{% endmacro %}
```

Update `bronze_tpch_lineitem.sql` to use both:

```sql
-- Before
ROUND(l_extendedprice * (1 - l_discount), 2) AS net_price,
ROUND(l_extendedprice * l_discount, 2)        AS discount_amount

-- After
{{ net_price('l_extendedprice', 'l_discount') }}      AS net_price,
{{ discount_amount('l_extendedprice', 'l_discount') }} AS discount_amount
```

Verify:
```bash
dbt run --select bronze_tpch_lineitem
```

Open `bronze_tpch_lineitem.sql` in the Cloud IDE and click **Compile** to see the macro expanded into plain SQL.

---

## Part B — Macro with Overridable Defaults

> *Guided — follow along, full solution shown.*

Create `macros/revenue_tier.sql` that classifies a numeric value into Low / Medium / High with configurable thresholds:

```sql
{% macro revenue_tier(col, low=1000, high=10000) %}
    CASE
        WHEN {{ col }} < {{ low }}  THEN 'Low'
        WHEN {{ col }} < {{ high }} THEN 'Medium'
        ELSE 'High'
    END
{% endmacro %}
```

Use it in `gold_orders.sql` with the defaults — they make sense at the scale of a single order:

```sql
{{ revenue_tier('net_revenue') }} AS revenue_tier
```

Then use it again in `gold_customers.sql`, but override the thresholds — customer lifetime revenue operates at a different scale:

```sql
{{ revenue_tier('total_revenue', 1000000, 3000000) }} AS revenue_tier
```

Same macro, different thresholds depending on context. That's the point of defaults: sensible out of the box, overridable when the context changes.

---

## Part C — A Macro That Generates Multiple Columns

> *Guided — follow along, full solution shown.*

Create `macros/date_dimensions.sql` that generates common date fields from a date column:

```sql
{% macro date_dimensions(date_column) %}
    YEAR({{ date_column }})         AS {{ date_column | replace('.', '_') }}_year,
    MONTH({{ date_column }})        AS {{ date_column | replace('.', '_') }}_month,
    DAY({{ date_column }})          AS {{ date_column | replace('.', '_') }}_day,
    DAYOFWEEK({{ date_column }})    AS {{ date_column | replace('.', '_') }}_day_of_week,
    QUARTER({{ date_column }})      AS {{ date_column | replace('.', '_') }}_quarter
{% endmacro %}
```

Use it in `gold_orders.sql`:

```sql
SELECT
    order_id,
    order_date,
    {{ date_dimensions('order_date') }},
    customer_name,
    net_revenue,
    ...
```

Run and check the compiled output:
```bash
dbt run --select gold_orders --full-refresh
```

---

## Part D — An Operational Macro: drop_old_relations

> *Guided — follow along, full solution shown.*

Create `macros/drop_old_relations.sql`. This is an operation macro (called with `dbt run-operation`), not used inside models.

The macro accepts a `relation_type` parameter so it can target tables, views, or both:

```sql
{% macro drop_old_relations(schema, days_old=7, relation_type='TABLE') %}

    {% set type_filter %}
        {% if relation_type | upper == 'ALL' %}
            table_type IN ('BASE TABLE', 'VIEW')
        {% elif relation_type | upper == 'TABLE' %}
            table_type = 'BASE TABLE'
        {% else %}
            table_type = 'VIEW'
        {% endif %}
    {% endset %}

    {% set query %}
        SELECT table_name, table_type
        FROM information_schema.tables
        WHERE table_schema = UPPER('{{ schema }}')
          AND {{ type_filter }}
          AND created < DATEADD(day, -{{ days_old }}, CURRENT_TIMESTAMP())
    {% endset %}

    {% set results = run_query(query) %}

    {% if execute %}
        {% for row in results %}
            {% set obj_type = 'VIEW' if row[1] == 'VIEW' else 'TABLE' %}
            {% set drop_stmt %}
                DROP {{ obj_type }} IF EXISTS {{ schema }}.{{ row[0] }}
            {% endset %}
            {{ log('Dropping ' ~ obj_type ~ ': ' ~ row[0], info=True) }}
            {% do run_query(drop_stmt) %}
        {% endfor %}
    {% endif %}

{% endmacro %}
```

Run it against your dev schema — replace `YOUR_SCHEMA` with the actual schema name (e.g. `DBT_DEV_JUAN_BRONZE`):

```bash
# Drop stale tables (default)
dbt run-operation drop_old_relations --args '{schema: YOUR_SCHEMA, days_old: 30}'

# Drop stale views
dbt run-operation drop_old_relations --args '{schema: YOUR_SCHEMA, days_old: 7, relation_type: VIEW}'

# Drop everything older than 14 days
dbt run-operation drop_old_relations --args '{schema: YOUR_SCHEMA, days_old: 14, relation_type: ALL}'
```

If you dropped something by mistake, Snowflake lets you recover it with `UNDROP`. Run this directly in the Snowflake console or worksheet — it's not a dbt command:

```sql
-- Recover a table
UNDROP TABLE YOUR_SCHEMA.TABLE_NAME;

-- Recover a view (views are not recoverable with UNDROP — recreate from dbt)
dbt run --select your_view_model
```

Snowflake retains dropped tables for up to 90 days (depending on your `DATA_RETENTION_TIME_IN_DAYS` setting). Views have no retention — if you drop one, rebuild it with dbt.

---

## Part E — Practice: shipping_days

> *Practice — write this yourself.*

TPC-H tracks several dates per line item: when the order was placed, when it was committed to ship, when it actually shipped, and when it was received. Create a macro that calculates the number of time units between two date columns.

Create `macros/shipping_days.sql`:

```sql
-- Your solution here
-- Hint: use Snowflake's DATEDIFF function
-- Hint: the unit ('day', 'week') should be a parameter with a sensible default
```

Then add two columns to `bronze_tpch_lineitem.sql` using the macro:

```sql
-- Days from order to actual shipment (use the default unit)
_______________________________ AS days_to_ship,

-- Days from shipment to receipt — override the unit to weeks
_______________________________ AS weeks_to_receive,
```

Run and verify:

```bash
dbt run --select bronze_tpch_lineitem
```

```sql
SELECT order_id, days_to_ship, weeks_to_receive
FROM ANALYTICS.BRONZE.BRONZE_TPCH_LINEITEM
LIMIT 10;
```

**Questions:**
1. Which date columns are available in `bronze_tpch_lineitem`? Which pair makes most sense for measuring delivery performance?
2. What happens if you swap `start_col` and `end_col`?

---

## ✅ Success Criteria

- [ ] `net_price` macro replaces the inline ROUND expression in `bronze_tpch_lineitem`
- [ ] `safe_divide` macro used in `gold_customers` and `gold_orders`
- [ ] `date_dim