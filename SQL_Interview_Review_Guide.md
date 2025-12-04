# SQL Interview Review Guide

## Table of Contents
1. [Window Functions](#window-functions)
2. [RANGE vs ROWS - Critical Distinction](#range-vs-rows)
3. [Common Table Expressions (CTEs)](#ctes)
4. [Aggregations and GROUP BY](#aggregations)
5. [JOINs](#joins)
6. [Date Functions](#date-functions)
7. [Common Patterns](#common-patterns)
8. [Interview Tips](#interview-tips)

---

## Window Functions

### Basic Syntax
```sql
function_name() OVER (
    PARTITION BY column1, column2
    ORDER BY column3
    ROWS/RANGE BETWEEN start AND end
)
```

### Common Window Functions

**Ranking Functions:**
- `ROW_NUMBER()` - Sequential number (1, 2, 3, 4...)
- `RANK()` - Same rank for ties, skips numbers (1, 2, 2, 4...)
- `DENSE_RANK()` - Same rank for ties, no gaps (1, 2, 2, 3...)

**Aggregate Functions:**
- `SUM()`, `AVG()`, `COUNT()`, `MIN()`, `MAX()`

**Offset Functions:**
- `LAG(column, n)` - Value from n rows before
- `LEAD(column, n)` - Value from n rows after
- `FIRST_VALUE()` - First value in window
- `LAST_VALUE()` - Last value in window

**Percentile Functions (PostgreSQL):**
- `PERCENTILE_CONT(0.5)` - Continuous percentile (interpolates)
- `PERCENTILE_DISC(0.5)` - Discrete percentile (actual value)

### Example: Running Totals
```sql
SELECT 
    user_id,
    transaction_date,
    amount,
    SUM(amount) OVER (
        PARTITION BY user_id 
        ORDER BY transaction_date
    ) AS running_total
FROM transactions;
```

---

## RANGE vs ROWS - Critical Distinction

### ⚠️ THIS IS CRUCIAL FOR INTERVIEWS

When you use `ORDER BY` in a window function, the **default frame is RANGE**, not ROWS!

### Default Behavior
```sql
AVG(amount) OVER (PARTITION BY user_id ORDER BY transaction_date)
-- Equivalent to:
AVG(amount) OVER (
    PARTITION BY user_id 
    ORDER BY transaction_date
    RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
)
```

### RANGE vs ROWS Difference

**RANGE**: Groups rows with **same ORDER BY value** together  
**ROWS**: Counts **physical rows** sequentially

### Example That Shows the Difference

**Data:**
```
user_id | transaction_date | amount
--------|------------------|-------
1       | 2025-01-10      | 100
1       | 2025-01-10      | 200  ← SAME DATE
1       | 2025-01-15      | 150
```

**Using RANGE (default):**
```sql
SELECT 
    user_id,
    transaction_date,
    amount,
    AVG(amount) OVER (
        PARTITION BY user_id 
        ORDER BY transaction_date
    ) AS running_mean
FROM transactions;
```

**Result:**
```
user_id | date       | amount | running_mean
--------|------------|--------|-------------
1       | 2025-01-10 | 100    | 150.00  ← Both Jan 10 rows included!
1       | 2025-01-10 | 200    | 150.00  ← Same value!
1       | 2025-01-15 | 150    | 150.00
```
Both rows on Jan 10 get the same running mean: (100 + 200) / 2 = 150

**Using ROWS (with tie-breaker):**
```sql
SELECT 
    user_id,
    transaction_date,
    amount,
    AVG(amount) OVER (
        PARTITION BY user_id 
        ORDER BY transaction_date, amount
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_mean
FROM transactions;
```

**Result:**
```
user_id | date       | amount | running_mean
--------|------------|--------|-------------
1       | 2025-01-10 | 100    | 100.00  ← Only first row
1       | 2025-01-10 | 200    | 150.00  ← First + second
1       | 2025-01-15 | 150    | 150.00
```

### When to Use Each

**Use RANGE when:**
- You want ties in ORDER BY values treated as a group
- Calculating aggregates where simultaneous events should be grouped
- Default behavior is acceptable

**Use ROWS when:**
- You want strict row-by-row calculation
- Running calculations where order matters even within same timestamp
- Working with PERCENTILE functions (more predictable)
- Better performance (usually faster than RANGE)

### Best Practice for Interviews
**Be explicit!** Always specify `ROWS BETWEEN` or `RANGE BETWEEN` to show you understand the difference.

---

## Common Table Expressions (CTEs)

### Basic Syntax
```sql
WITH cte_name AS (
    SELECT ...
    FROM ...
),
another_cte AS (
    SELECT ...
    FROM cte_name
)
SELECT ...
FROM another_cte;
```

### Benefits
- Improves readability
- Makes complex queries modular
- Can reference earlier CTEs
- Easier to debug

### Example: Multi-step Analysis
```sql
WITH user_first_purchase AS (
    SELECT 
        user_id,
        MIN(purchase_date) AS first_date
    FROM purchases
    GROUP BY user_id
),
user_cohorts AS (
    SELECT 
        p.user_id,
        p.purchase_date,
        DATE_TRUNC('month', ufp.first_date) AS cohort_month
    FROM purchases p
    JOIN user_first_purchase ufp ON p.user_id = ufp.user_id
)
SELECT 
    cohort_month,
    COUNT(DISTINCT user_id) AS cohort_size
FROM user_cohorts
GROUP BY cohort_month;
```

---

## Aggregations and GROUP BY

### Key Rules
1. Every column in SELECT must be either:
   - In the GROUP BY clause, OR
   - Inside an aggregate function
2. Aggregate functions: `COUNT()`, `SUM()`, `AVG()`, `MIN()`, `MAX()`
3. `HAVING` filters after aggregation, `WHERE` filters before

### Example
```sql
SELECT 
    user_id,
    DATE_TRUNC('month', purchase_date) AS month,
    COUNT(*) AS num_purchases,
    SUM(amount) AS total_spent,
    AVG(amount) AS avg_purchase
FROM purchases
WHERE purchase_date >= '2025-01-01'
GROUP BY user_id, DATE_TRUNC('month', purchase_date)
HAVING COUNT(*) >= 3;
```

### Common Aggregate Patterns

**Count distinct:**
```sql
COUNT(DISTINCT user_id)
```

**Conditional aggregation:**
```sql
SUM(CASE WHEN status = 'completed' THEN amount ELSE 0 END) AS completed_amount
```

**Multiple aggregates:**
```sql
SELECT 
    category,
    COUNT(*) AS total_count,
    COUNT(DISTINCT user_id) AS unique_users,
    SUM(amount) AS total_revenue,
    AVG(amount) AS avg_order_value
FROM orders
GROUP BY category;
```

---

## JOINs

### Types of JOINs

**INNER JOIN**: Only matching rows from both tables
```sql
SELECT *
FROM table_a a
INNER JOIN table_b b ON a.id = b.a_id;
```

**LEFT JOIN**: All rows from left table, matching rows from right (NULL if no match)
```sql
SELECT *
FROM table_a a
LEFT JOIN table_b b ON a.id = b.a_id;
```

**RIGHT JOIN**: All rows from right table, matching rows from left
```sql
SELECT *
FROM table_a a
RIGHT JOIN table_b b ON a.id = b.a_id;
```

**FULL OUTER JOIN**: All rows from both tables
```sql
SELECT *
FROM table_a a
FULL OUTER JOIN table_b b ON a.id = b.a_id;
```

**CROSS JOIN**: Cartesian product (every row with every row)
```sql
SELECT *
FROM table_a a
CROSS JOIN table_b b;
```

### Self-Join Example
Finding users who made purchases on consecutive days:
```sql
SELECT DISTINCT
    p1.user_id,
    p1.purchase_date AS day1,
    p2.purchase_date AS day2
FROM purchases p1
JOIN purchases p2 
    ON p1.user_id = p2.user_id
    AND p2.purchase_date = p1.purchase_date + INTERVAL '1 day';
```

---

## Date Functions

### Common Date Operations (PostgreSQL)

**Extract parts:**
```sql
EXTRACT(YEAR FROM date_column)
EXTRACT(MONTH FROM date_column)
EXTRACT(DAY FROM date_column)
```

**Truncate to period:**
```sql
DATE_TRUNC('month', date_column)  -- First day of month
DATE_TRUNC('week', date_column)   -- First day of week
DATE_TRUNC('year', date_column)   -- First day of year
```

**Date arithmetic:**
```sql
date_column + INTERVAL '1 day'
date_column - INTERVAL '3 months'
date_column + INTERVAL '2 hours'
```

**Difference between dates:**
```sql
-- Days between
date2 - date1

-- Months between (PostgreSQL)
(EXTRACT(YEAR FROM date2) - EXTRACT(YEAR FROM date1)) * 12 +
(EXTRACT(MONTH FROM date2) - EXTRACT(MONTH FROM date1))
```

**Format dates:**
```sql
TO_CHAR(date_column, 'YYYY-MM')     -- '2025-01'
TO_CHAR(date_column, 'Month YYYY')  -- 'January 2025'
```

### BETWEEN with Dates
**Important**: `BETWEEN` is **inclusive** on both ends!
```sql
WHERE purchase_date BETWEEN '2025-01-01' AND '2025-01-31'
-- Includes both Jan 1 and Jan 31
```

---

## Common Patterns

### Pattern 1: Running Calculations
```sql
-- Running total
SUM(amount) OVER (PARTITION BY user_id ORDER BY date 
                  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)

-- Running average
AVG(amount) OVER (PARTITION BY user_id ORDER BY date
                  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
```

### Pattern 2: Cohort Analysis
```sql
WITH cohorts AS (
    SELECT 
        user_id,
        DATE_TRUNC('month', MIN(purchase_date)) AS cohort_month
    FROM purchases
    GROUP BY user_id
),
cohort_activity AS (
    SELECT 
        c.cohort_month,
        DATE_TRUNC('month', p.purchase_date) AS activity_month,
        COUNT(DISTINCT p.user_id) AS active_users
    FROM purchases p
    JOIN cohorts c ON p.user_id = c.user_id
    GROUP BY c.cohort_month, DATE_TRUNC('month', p.purchase_date)
)
SELECT 
    cohort_month,
    (EXTRACT(YEAR FROM activity_month) - EXTRACT(YEAR FROM cohort_month)) * 12 +
    (EXTRACT(MONTH FROM activity_month) - EXTRACT(MONTH FROM cohort_month)) AS month_number,
    active_users
FROM cohort_activity;
```

### Pattern 3: Finding First/Last Values
```sql
-- First purchase per user
SELECT 
    user_id,
    MIN(purchase_date) AS first_purchase
FROM purchases
GROUP BY user_id;

-- Or with window function
SELECT DISTINCT
    user_id,
    FIRST_VALUE(purchase_date) OVER (
        PARTITION BY user_id 
        ORDER BY purchase_date
    ) AS first_purchase
FROM purchases;
```

### Pattern 4: Year-over-Year Comparison
```sql
WITH monthly_revenue AS (
    SELECT 
        DATE_TRUNC('month', order_date) AS month,
        SUM(amount) AS revenue
    FROM orders
    GROUP BY DATE_TRUNC('month', order_date)
)
SELECT 
    month,
    revenue,
    LAG(revenue, 12) OVER (ORDER BY month) AS revenue_last_year,
    revenue - LAG(revenue, 12) OVER (ORDER BY month) AS yoy_change,
    ROUND(100.0 * (revenue - LAG(revenue, 12) OVER (ORDER BY month)) / 
          LAG(revenue, 12) OVER (ORDER BY month), 2) AS yoy_percent_change
FROM monthly_revenue;
```

### Pattern 5: Moving Averages
```sql
SELECT 
    date,
    value,
    AVG(value) OVER (
        ORDER BY date 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7day
FROM daily_metrics;
```

---

## Interview Tips

### 1. Think Before You Code
- Understand the question completely
- Sketch out the logic on paper
- Identify what CTEs or subqueries you'll need
- Consider edge cases

### 2. Build Incrementally
- Start with the innermost query or first CTE
- Test each piece before adding complexity
- Use CTEs to break down complex problems

### 3. Common Mistakes to Avoid
- Forgetting to handle NULL values
- Not considering ties in rankings
- Mixing aggregate and non-aggregate columns without GROUP BY
- Using WHERE instead of HAVING for aggregate filters
- Forgetting DISTINCT when counting unique values
- Not being explicit about ROWS vs RANGE in window functions

### 4. Optimization Awareness
- Mention indexes when relevant
- Be aware of Cartesian products (CROSS JOIN)
- Know when to use EXISTS vs IN
- Understand LEFT JOIN vs INNER JOIN performance

### 5. Communication
- Explain your approach before coding
- Walk through your logic as you write
- Test with sample data mentally
- Explain what your output means

### 6. Key Questions to Ask
- What SQL dialect are we using?
- Are there any performance constraints?
- How should we handle NULL values?
- What should happen with edge cases (ties, empty results)?
- Should results be ordered in a specific way?

---

## Quick Reference: Window Function Frames

```sql
-- Include all previous rows and current row (running calculation)
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW

-- Include all rows in partition
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING

-- Moving window (e.g., 7-day moving average)
ROWS BETWEEN 6 PRECEDING AND CURRENT ROW

-- Centered window
ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING

-- Only current row
ROWS BETWEEN CURRENT ROW AND CURRENT ROW
```

---

## Common SQL Interview Question Types

1. **Cohort Analysis** - Track user behavior over time
2. **Running Calculations** - Cumulative metrics
3. **Ranking/Top N** - Find top performers, rank items
4. **Time Series** - Trends, moving averages, YoY comparisons
5. **Funnel Analysis** - Conversion rates, drop-offs
6. **Percentiles/Medians** - Statistical distributions
7. **Self-Joins** - Find relationships within same table
8. **Complex Aggregations** - Multiple levels of grouping

---

## Practice Problems Summary

### Easy
- Filter and aggregate sales by category
- Find top 5 customers by revenue
- Calculate month-over-month growth

### Medium
- **Cohort retention analysis** (the one we practiced!)
- Running median calculation
- Find users who made purchases on consecutive days
- Calculate customer lifetime value

### Hard
- Multi-level funnel conversion rates
- Time-weighted averages
- Complex date range overlaps
- Recursive CTEs for hierarchical data

---

**Good luck with your interviews! Remember: clarity and correctness beat cleverness every time.**
