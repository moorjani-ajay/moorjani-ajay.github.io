---
title: "Unit Tests vs Data Quality Checks"
date: 2025-06-26
categories: ["Data Engineering"]
description: "Understanding the difference between unit tests and data quality checks in data engineering."
tags:
  [
    unit testing,
    data quality,
    data engineering,
    dbt,
    testing,
    data pipelines,
    software testing,
  ]
---

# Unit Tests vs Data Quality Checks

Another day... another Reddit post to dissect. According to me, it's essential to draw a clear line between two commonly confused concepts in data engineering: Unit Testing and Data Quality Checks.

I recently came across a Reddit discussion titled, "Unit tests != data quality checks." The original poster argued a point I've long agreed with—though related, unit tests and data quality checks serve fundamentally different purposes.

## What are Unit Tests?

Unit tests verify that individual units of code (functions, modules, classes) behave as expected. They are conducted before deploying or merging code into production. Their primary goal is to catch issues early in the development lifecycle—whether those issues arise from refactoring, new features, or integration problems.

<!-- Take SQL example -->

In DE world, unit tests can be written to validate SQL queries, ensuring that they return the expected results for given inputs. These tests are typically run in a controlled environment with known data sets, allowing developers to focus on the logic of the code rather than the data itself.

A unit test might check if a function correctly calculates the average sales for a given date range. The test would focus on the logic of the `calculate_daily_avg` function, ensuring it returns the expected output for known inputs.

```python
def calculate_daily_avg(sales_data: list) -> dict:
    """
    Calculates daily average sales from transaction data
    Args:
        sales_data: List of tuples with (date, amount)
    Returns:
        dict: Daily averages {date: average_amount}
    """
    daily_totals = {}
    daily_counts = {}

    for date, amount in sales_data:
        daily_totals[date] = daily_totals.get(date, 0) + amount
        daily_counts[date] = daily_counts.get(date, 0) + 1

    return {date: daily_totals[date]/daily_counts[date]
            for date in daily_totals}

class TestCalculateDailyAvg(unittest.TestCase):
    def test_calculate_daily_avg(self):
        sales_data = [
            ('2023-01-01', 100),
            ('2023-01-01', 200),
            ('2023-01-02', 300)
        ]
        expected = {
            '2023-01-01': 150.0,
            '2023-01-02': 300.0
        }
        self.assertEqual(calculate_daily_avg(sales_data), expected)

```

## What are Data Quality Checks?

So, what about data quality checks? DQ checks, on the other hand, ensure that production data meets certain criteria each time it flows through a pipeline or database. These checks validate data integrity, completeness, and consistency at runtime—after the code has been deployed.

A common way of implementing data quality checks in data engineering is using DBT. For example, to ensure a critical column such as customer_id in a table is neither null nor duplicated, you might have DBT tests as follows:

```yaml
version: 2

models:
  - name: transactions
    columns:
      - name: customer_id
        tests:
          - not_null
          - unique
```

These tests validate data at runtime, ensuring ongoing trust and reliability of the data pipeline.

## So to summarize:

- Unit tests run during build or pre-deployment.
- Data Quality checks run post-deployment, continuously monitoring live data.

## And what is the purpose ?

- Unit tests catch issues related to logic and code changes.
- Data Quality checks catch issues arising from unexpected or faulty data.

# My take :

In short, both unit tests and data quality checks are indispensable, serving different yet complementary roles. Confusing these two can lead to gaps in coverage and reliability issues.

Full Reddit thread : https://www.reddit.com/r/dataengineering/comments/1ljq8r4/unit_tests_data_quality_checks_cmv/

Thank you!!!!
