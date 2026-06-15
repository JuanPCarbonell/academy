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

And a companion macro for net price:

```sql
{% macro net_price(price_col, discount_col, precision=2) %}
    ROUND({{ price_col }} * (1 - {{ discount_col }}), {{ precision }})
{% endmacro %}
```

Update `bronze_tpch_lineitem.sql` to use them:

```sql
-- Before
ROUND(l_extendedprice * (1 - l_discount), 2) AS net_price

-- After
{{ net_price('l_extendedprice', 'l_discount') }} AS net_price
```

Verify:
```bash
dbt run --select bronze_tpch_lineitem
```

Open `bronze_tpch_lineitem.sql` in the Cloud IDE and click **Compile** to see the macro expanded into plain SQL.

---

## Part B — Macro with Default Arguments

> *Guided — follow along, full solution shown.*

Create `macros/safe_divide.sql` that handles division by zero:

```sql
{% macro safe_divide(numerator, denominator, default=0) %}
    CASE
        WHEN {{ denominator }} = 0 OR {{ denominator }} IS NULL
        THEN {{ default }}
        ELSE {{ numerator }} / {{ denominator }}
    END
{% endmacro %}
```

Use it in `gold_customers.sql` when calculating average order value:

```sql
{{ safe_divide('total_revenue', 'total_orders') }} AS avg_order_value
```

And in `gold_orders.sql` for both division columns:

```sql
{{ safe_divide('net_revenue', 'total_items') }}                         AS revenue_per_item,
{{ safe_divide('gross_revenue - net_revenue', 'gross_revenue', 0) }}    AS effective_discount_rate,
```

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
dbt run --select gold_orders
```

---

## Part D — An Operational Macro: drop_old_relations

> *Guided — follow along, full solution shown.*

Create `macros/drop_old_relations.sql`. This is an operation macro (called with `dbt run-operation`), not used inside models:

```sql
{% macro drop_old_relations(schema, days_old=7) %}

    {% set query %}
        SELECT table_name
        FROM information_schema.tables
        WHERE table_schema = UPPER('{{ schema }}')
          AND created < DATEADD(day, -{{ days_old }}, CURRENT_TIMESTAMP())
    {% endset %}

    {% set results = run_query(query) %}

    {% if execute %}
        {% for row in results %}
            {% set drop_stmt %}
                DROP TABLE IF EXISTS {{ schema }}.{{ row[0] }}
            {% endset %}
            {{ log('Dropping: ' ~ row[0], info=True) }}
            {% do run_query(drop_stmt) %}
        {% endfor %}
    {% endif %}

{% endmacro %}
```

Run it (dry-run against a dev schema):
```bash
dbt run-operation drop_old_relations --args '{"schema": "BRONZE", "days_old": 30}'
```

---

## Part E — Jinja Conditionals and Loops

> *Practice — write this yourself.*

Create `macros/union_relations.sql`:

```
# Your solution here
```

Test it in `analyses/union_test.sql`:
```
# Your solution here
```

```
# Your solution here
```

---

## ✅ Success Criteria

- [ ] `net_price` macro replaces the inline ROUND expression in `bronze_tpch_lineitem`
- [ ] `safe_divide` macro used in `gold_customers` and `gold_orders`
- [ ] `date_dimensions` macro generates year/month/day/quarter columns in `gold_orders`
- [ ] `drop_old_relations` operation macro runs without error
- [ ] You used the **Compile** button to verify macro expansion
