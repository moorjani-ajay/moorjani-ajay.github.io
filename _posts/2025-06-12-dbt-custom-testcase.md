---
title: "Threshold‑Aware dbt Tests: When Not Null Is Too Strict"
date: 2025-06-12
categories: [dbt, Data Engineering, Data Quality]
description: "How to create custom dbt tests that allow for a small percentage of duplicates or nulls, keeping your batch green while still surfacing issues."
tags:
  [
    dbt-core,
    custom test,
    data testing,
    jinja,
    macro,
    threshold,
    data pipeline,
    data observability,
  ]
---

# Threshold‑Aware dbt Tests: When **Not Null** Is Too Strict

## Context

On a recent warehouse project we kept the orchestration dead‑simple: every night we ran

```bash
dbt run && dbt test
```

To guard our primary keys we relied on two generic tests:

- `not_null` – the key must be present.
- `unique` – the key must appear only once.

That worked until real‑world noise hit. A _single_ duplicate in a 50 million‑row table made the whole batch red and blocked downstream transformations. One bad row shouldn’t wreck an entire load.

## Goal

1. Keep the batch green when the defect rate is tiny.
2. Still surface problems so we can fix them.
3. Stay 100 % inside dbt—no extra services or plugins.

---

## The `duplicates_pct_threshold` test

We wrote a custom macro called `duplicates_pct_threshold`.

```sql
{% raw %}
{% test duplicates_pct_threshold(model, column_name, max_dup_pct=2) %}
-- Fails if duplicate_pct > max_dup_pct, otherwise warns.
-- (Full code in duplicates_pct_threshold.sql)
{% endtest %}
{% rawend %}
```

How it works:

| Step  | Action                                                                   |
| ----- | ------------------------------------------------------------------------ |
| **1** | Count total rows and distinct keys.                                      |
| **2** | Compute `duplicate_pct` = (total − distinct) / total × 100.              |
| **3** | If `duplicate_pct` > `max_dup_pct` → `exceptions.raise_compiler_error()` |
| **4** | Else if `duplicate_pct` > 0 → `exceptions.warn()`                        |

A warning returns exit code 0, so dbt treats the test as _passed_ but prints the alert in the log. A compiler error returns non‑zero, stopping the run.

---

## Example Usage

```yaml
version: 2
models:
  - name: fact_orders
    tests:
      - duplicates_pct_threshold:
          column_name: order_id
          max_dup_pct: 2 # warn ≤ 2 %, fail > 2 %
```

---

Here is the full code for the custom test case:

```sql
{% raw %}
{% test duplicates_pct_threshold(model, column_name, max_dup_pct=2) %}

/*

    Fail when the percentage of duplicate values in `column_name`
    exceeds `max_dup_pct`.

    Parameters
    ----------
    model           : refable   – table / view under test
    column_name     : text      – column whose duplicate rate we measure
    max_dup_pct     : numeric   – allowed duplicate percentage (0-100)
                                    default = 2
*/
    {% set pct_sql %}
        with stats as (
            select
                count(*) as total_rows,
                count(distinct {{ column_name }}) as distinct_rows
            from {{ model }}
        )
         select
            case
                when total_rows = 0 then 0
                else (total_rows - distinct_rows) * 100.0 / total_rows
            end as duplicate_pct,
            total_rows,
            distinct_rows
        from stats
    {% endset %}
    {# log(pct_sql, info=True) #}

    {% if execute %}
        {% set results = run_query(pct_sql) %}
        {% set pct = results.columns[0].values()[0] | float | round(2) %}
        {% set total_rows = results.columns[1].values()[0] %}
        {% set distinct_rows = results.columns[2].values()[0] %}
        {% set duplicates = total_rows - distinct_rows %}
    {% else %}
        {% set pct = 0 %}
        {% set duplicates = 0 %}
        {% set total_rows = 0 %}
    {% endif %}
    -- {#log( "Duplicate percentage: " ~ pct, info=True)#}

    -- return error with actual count if it exceeds the threshold
    {% if pct > max_dup_pct %}
        {{ exceptions.raise_compiler_error(
            "Duplicate percentage of " ~ pct ~ "% exceeds the maximum allowed of " ~ max_dup_pct ~ "%. " ~
            "Total duplicates found: " ~ duplicates
        ) }}
    {% endif %}

    -- return warn if it exceeds the threshold but is below the error level
    {% if pct < max_dup_pct and pct > 0 %}
        {{ exceptions.warn(
            "Warning: Duplicate percentage of " ~ pct ~ "% exceeds the recommended threshold of " ~ max_dup_pct ~ "%. " ~
            "Total duplicates found: " ~ duplicates ~ ". "
        ) }}
    {% endif %}

    -- This select is just to ensure the test runs without errors
    select 1 as dummy_column where 1=0

{% rawend %}
{% endtest %}

```

### And here is the null threshold-aware test case:

```sql
{% raw %}
{% test null_pct_threshold(model, column_name, max_null_pct=2) %}

    /*
        Fail when the percentage of NULLs in `column_name`
        exceeds `max_null_pct`.

        Parameters
        ----------
        model           : refable   – table / view under test
        column_name     : text      – column whose NULL rate we measure
        max_null_pct    : numeric   – allowed NULL percentage (0-100)
                                        default = 2
    */

    {% set pct_sql %}
        with stats as (
            select
                count(*)                                           as total_rows,
                sum(case when {{ column_name }} is null then 1 end) as null_rows
            from {{ model }}

        )
        select
            total_rows,
            null_rows,
            case
                when total_rows = 0 then 0
                else null_rows * 100.0 / total_rows
            end                                                 as null_pct
        from stats
    {% endset %}

    -- {# log(pct_sql, info=True) #}

    {% if execute %}
        {% set results = run_query(pct_sql) %}
        {% set null_pct = results.columns[2].values()[0] | float | round(2) %}
        {% set total_rows = results.columns[0].values()[0] %}
        {% set null_rows = results.columns[1].values()[0] %}
    {% else %}
        {% set null_pct = 0 %}
        {% set total_rows = 0 %}
        {% set null_rows = 0 %}
    {% endif %}

    -- {# log("Null percentage: " ~ null_pct, info=True) #}
    -- return error with actual count if it exceeds the threshold
    {% if null_pct > max_null_pct %}
        {{ exceptions.raise_compiler_error(
            "Null percentage of " ~ null_pct ~ "% exceeds the maximum allowed of " ~ max_null_pct ~ "%. " ~
            "Total NULLs found: " ~ null_rows
        ) }}
    {% endif %}
    -- return warn if it exceeds the threshold but is below the error level
    {% if null_pct < max_null_pct and null_pct > 0 %}
        {{ exceptions.warn(
            "Warning: Null percentage of " ~ null_pct ~ "% exceeds the recommended threshold of " ~ max_null_pct ~ "%. " ~
            "Total NULLs found: " ~ null_rows ~ ". "
        ) }}
    {% endif %}

    -- This select is just to ensure the test runs without errors
    select 1 as dummy_column where 1=0


{% endtest %}
{% endraw %}

```

Please make sure to remove {% raw %} and {% endraw %} tags before using the code in your dbt project.

Happy testing! 🚀
