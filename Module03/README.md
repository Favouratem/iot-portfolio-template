<style>
  a {
    text-decoration: none;
    color: #464feb;
  }
  tr th, tr td {
    border: 1px solid #e6e6e6;
  }
  tr th {
    background-color: #f5f5f5;
  }
</style>

# Module 3 — LLM-Assisted SQL & Query Profiling

## 👥 Team Context

This portfolio work was completed by a **mixed project team consisting of Mechatronics Engineering and Industrial Engineering students**.

- The **Mechatronics students** focused on understanding how database queries support technical systems, production monitoring, and warehouse operations.
- The **Industrial Engineering students** focused on process efficiency, data analysis, performance optimisation, and data quality assurance.

Throughout the exercise, our team used AI-assisted SQL generation, verified the results manually, and evaluated query performance and database security.

---

## Table of Contents

- [Task 1: Ask the LLM, then Check the Plan](#task-1)
  - [Prompt Used](#prompt-used)
  - [Generated SQL](#generated-sql)
  - [Our Predicted Plan](#our-predicted-plan)
  - [Observed Query Plan](#explain-query-plan-observed)
  - [Correctness Review](#correctness-review)
- [Task 2: Index Benchmark](#task-2)
  - [Baseline Query](#baseline-query)
  - [Indexes Added Sequentially](#indexes-added-sequentially)
  - [Benchmark Results](#benchmark-results)
  - [Plan Verification](#plan-verification)
  - [Index That Did Not Help](#index-that-did-not-help)
- [Task 3: The Injection Drill](#task-3)
  - [Security Finding](#security-finding)
  - [Safe Version](#safe-version)
  - [Why Escaping Quotes Is Not a Real Fix](#why-escaping-quotes-is-not-a-real-fix)
- [Task 4: One Query, From Algebra to Plan](#task-4)
  - [Relational Algebra](#relational-algebra)
  - [SQL Equivalent](#sql-equivalent)
  - [Verification](#verification)
- [Reflection](#reflection)

---

<a id="task-1"></a>
# ★ Task 1: Ask the LLM, then Check the Plan

## Prompt Used

```text
SQLite schema:

customer(customer_id, name, city)
product(product_id, sku, price)
orders(order_id, customer_id, ordered_on, status)
order_line(order_id, product_id, qty)

Write a query that returns, for each city, the number of distinct customers and the total revenue (sum of qty * price) of orders placed in 2025, ordered by revenue descending.
```

---

## Generated SQL

```sql
SELECT
    c.city,
    COUNT(DISTINCT c.customer_id) AS customers,
    SUM(ol.qty * p.price) AS revenue
FROM customer c
JOIN orders o
    ON o.customer_id = c.customer_id
JOIN order_line ol
    ON ol.order_id = o.order_id
JOIN product p
    ON p.product_id = ol.product_id
WHERE o.ordered_on >= '2025-01-01'
  AND o.ordered_on < '2026-01-01'
GROUP BY c.city
ORDER BY revenue DESC;
```

---

## Our Predicted Plan

Before running the query, our team expected:

```text
SCAN customer
SCAN orders
SCAN order_line
SEARCH product USING PRIMARY KEY

USE TEMP B-TREE FOR GROUP BY
USE TEMP B-TREE FOR ORDER BY
```

Reason:

- No explicit indexes exist on `ordered_on`.
- The query must scan a large portion of `orders`.
- Grouping and sorting revenue require temporary structures.

---

## EXPLAIN QUERY PLAN (Observed)

```text
SCAN orders
SEARCH customer USING INTEGER PRIMARY KEY
SEARCH order_line USING AUTOMATIC COVERING INDEX
SEARCH product USING INTEGER PRIMARY KEY

USE TEMP B-TREE FOR GROUP BY
USE TEMP B-TREE FOR ORDER BY
```

---

## Correctness Review

Our team verified the query manually.

### Requirements Check

| Requirement | Result |
|---|---|
| Group by city | ✅ |
| Count distinct customers | ✅ |
| Calculate total revenue | ✅ |
| Include orders placed in 2025 only | ✅ |
| Sort revenue descending | ✅ |

### Conclusion

The query was logically correct and fulfilled all requirements. The LLM generated a valid solution, but manual verification was still necessary to ensure the date filter and `DISTINCT` aggregation were implemented correctly.

---

<a id="task-2"></a>
# ★ Task 2: Index Benchmark

## Baseline Query

```sql
SELECT
    o.order_id,
    o.ordered_on,
    c.name
FROM orders o
JOIN customer c
    ON c.customer_id = o.customer_id
WHERE o.ordered_on = '2026-02-14';
```

---

## Indexes Added Sequentially

### Step 1

```sql
CREATE INDEX ix_orders_ordered_on
ON orders(ordered_on);
```

### Step 2

```sql
CREATE INDEX ix_orderline_order
ON order_line(order_id);
```

### Step 3

```sql
CREATE INDEX ix_orderline_product
ON order_line(product_id);
```

---

## Benchmark Results

| Configuration | Approx. runtime |
|---|---:|
| No index | 0.080 s |
| `ordered_on` index | 0.010 s |
| `ordered_on` + `order_id` index | 0.009 s |
| `ordered_on` + `order_id` + `product_id` indexes | 0.009 s |

---

## Plan Verification

### Before Index

```text
SCAN orders
```

### After Index

```text
SEARCH orders USING INDEX ix_orders_ordered_on
```

The planner now performs an indexed lookup rather than reading the entire table.

---

## Index That Did Not Help

Example:

```sql
CREATE INDEX ix_orders_status
ON orders(status);
```

### Why It Did Not Help

The `status` column contains only a few possible values:

```text
open
late
shipped
```

This low selectivity means a large percentage of rows share the same value. The query planner may decide that scanning the table is cheaper than using the index.

---

## Team Conclusion

As Industrial Engineering students, we observed that optimization should always be supported by measurement rather than assumptions.

As Mechatronics students, we recognised similarities to physical systems engineering: adding components does not automatically improve performance. Effectiveness must be validated using data and testing.

---

<a id="task-3"></a>
# ★ Task 3: The Injection Drill

## Security Finding

Building a SQL statement by inserting user-provided text directly into the query can allow that input to change the query's logic. This is SQL injection: the application intended to treat the input as a value, but the database may interpret part of it as SQL syntax.

## Safe Version

Use a bound parameter so the database treats the supplied name as data, not executable SQL:

```python
def find_customer(conn, name: str):
    sql = """
    SELECT customer_id, name
    FROM customer
    WHERE name = ?
    """
    return conn.execute(sql, (name,)).fetchall()
```

## Why Escaping Quotes Is Not a Real Fix

Our team concluded that manually escaping apostrophes is unreliable:

- Different databases use different escaping rules.
- Numeric fields can still be vulnerable to unsafe query construction.
- Encoding and formatting differences can bypass custom filters.
- Developers may forget to escape some inputs.

Bound parameters solve the problem fundamentally because SQL structure and data are kept separate. The database engine treats the input exclusively as a value rather than executing it as SQL syntax.

**Conclusion:** Bound parameters are the correct solution for values supplied at runtime.

---

<a id="task-4"></a>
# ★ Task 4: One Query, From Algebra to Plan

## Relational Algebra

Revenue-by-city query:

```text
γ city;
  COUNT(DISTINCT customer_id),
  SUM(qty × price)
(
    σ ordered_on ≥ '2025-01-01'
      ∧ ordered_on < '2026-01-01'
    (
      customer
      ⋈ orders
      ⋈ order_line
      ⋈ product
    )
)
```

---

## SQL Equivalent

```sql
SELECT
    c.city,
    COUNT(DISTINCT c.customer_id),
    SUM(ol.qty * p.price)
FROM customer c
JOIN orders o
    ON o.customer_id = c.customer_id
JOIN order_line ol
    ON ol.order_id = o.order_id
JOIN product p
    ON p.product_id = ol.product_id
WHERE o.ordered_on >= '2025-01-01'
  AND o.ordered_on < '2026-01-01'
GROUP BY c.city;
```

---

## Verification

| Relational algebra | SQL |
|---|---|
| `σ` (selection) | `WHERE` |
| `⋈` (join) | `JOIN` |
| `γ` (grouping) | `GROUP BY` |
| `SUM` | `SUM` |
| `COUNT` | `COUNT` |
| Projection | `SELECT` |

The SQL statement correctly implements the relational algebra expression.

---

<a id="reflection"></a>
# Reflection

## Team Reflection

As a multidisciplinary team of Mechatronics and Industrial Engineering students, this module helped us understand that writing SQL is only one part of database engineering. Modern AI tools can generate SQL quickly, but engineers remain responsible for correctness, performance, and security.

The most important lesson was that AI-generated queries should never be accepted without verification. During Task 1, we saw that a query can look correct while still introducing logical errors such as incorrect aggregation or misplaced filtering. Learning to validate SQL against business requirements and relational algebra strengthened our analytical skills.

Task 2 highlighted the importance of query profiling and performance measurement. We learned that indexes are not universally beneficial and that database optimisation requires evidence from execution plans and benchmarks. This reflected a common engineering principle: decisions should be driven by measured results rather than assumptions.

Task 3 demonstrated the practical risks of insecure software development. As future engineers working with production systems, automation platforms, and industrial data, understanding SQL injection vulnerabilities is essential. We learned that parameterised queries provide a robust and professional solution compared with ad-hoc filtering techniques.

Finally, translating SQL into relational algebra helped us connect theoretical database concepts with practical implementation. This improved our understanding of how database systems execute queries internally and how mathematical models support reliable data processing.

Overall, the module strengthened our competencies in AI-assisted development, database optimisation, cybersecurity awareness, and analytical reasoning. It also showed us that AI can significantly improve productivity when combined with critical engineering judgement and systematic verification.
