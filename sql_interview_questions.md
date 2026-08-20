# Comprehensive SQL Interview Questions - Detailed Answers

## **Window Functions: RANK, DENSE_RANK, ROW_NUMBER**

### **Q1: The Tie-Breaker Trap**

**Question:**
```sql
SELECT name, salary, 
       ROW_NUMBER() OVER (ORDER BY salary DESC) as rn,
       RANK() OVER (ORDER BY salary DESC) as rnk,
       DENSE_RANK() OVER (ORDER BY salary DESC) as drnk
FROM employees
WHERE salary IN (100000, 100000, 90000, 90000, 90000, 80000);
```

**Answer:**

| name | salary | rn | rnk | drnk |
|------|--------|----|----|------|
| John | 100000 | 1  | 1  | 1    |
| Jane | 100000 | 2  | 1  | 1    |
| Mike | 90000  | 3  | 3  | 2    |
| Mary | 90000  | 4  | 3  | 2    |
| Bob  | 90000  | 5  | 3  | 2    |
| Alice| 80000  | 6  | 6  | 3    |

**Detailed Explanation:**

- **ROW_NUMBER()**: Assigns a unique sequential integer to each row, even for ties. Always returns 1, 2, 3, 4, 5, 6. It's non-deterministic for ties unless you add a secondary sort criterion.

- **RANK()**: Assigns the same rank to ties, then skips numbers. Two people at rank 1, next person gets rank 3 (skips 2). This is like Olympic medal standings.

- **DENSE_RANK()**: Assigns the same rank to ties but doesn't skip numbers. Two people at rank 1, next person gets rank 2. This is continuous ranking.

**For "Top 3 Unique Salary Values":**
Use **DENSE_RANK()** because:
- It gives you distinct salary levels (100000=1, 90000=2, 80000=3)
- RANK() would give you 1, 3, 6 (skipping numbers)
- ROW_NUMBER() would give you 6 different values for 3 salary levels

```sql
SELECT DISTINCT salary
FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) as drnk
    FROM employees
) WHERE drnk <= 3;
```

**If you use ROW_NUMBER():** You'd get arbitrary employees based on physical row order, not all employees at those top 3 salary levels.

---

### **Q2: Partition Reset Confusion**

**Answer:**

**Version A:**
```sql
SELECT *, ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) as rn
FROM employees;
```
- Row numbers RESET for each department
- Dept A: 1, 2, 3; Dept B: 1, 2, 3; Dept C: 1, 2, 3
- This is for "top N per department" queries

**Version B:**
```sql
SELECT *, ROW_NUMBER() OVER (ORDER BY dept, salary DESC) as rn
FROM employees;
```
- Row numbers are CONTINUOUS across all rows
- Just sorts by dept first, then salary
- Result: 1, 2, 3, 4, 5, 6, 7, 8, 9...

**When They're The Same:**
Never! They fundamentally differ in behavior.

**When They APPEAR Similar:**
If you only have one department, or if you're only looking at the first row of each result set.

**Key Difference:**
- PARTITION BY creates separate "windows" (isolated groups)
- ORDER BY without PARTITION BY creates one big window (entire result set)

**Real-World Example:**
```sql
-- Get top 2 earners per department (Version A)
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) as rn
    FROM employees
) WHERE rn <= 2;  -- Returns 2 employees per department

-- Version B would just give you the first 2 rows overall
```

---

### **Q3: The NULL Ordering Puzzle**

**Answer:**

**SQL Server:**
- NULLs appear FIRST by default in DESC ordering
- NULLs appear LAST by default in ASC ordering
- Logic: SQL Server treats NULL as the "smallest" value

**PostgreSQL:**
- NULLs appear LAST by default in DESC ordering
- NULLs appear FIRST by default in ASC ordering  
- Logic: PostgreSQL treats NULL as the "largest" value

**MySQL:**
- Similar to SQL Server (NULLs are smallest)

**Oracle:**
- Similar to PostgreSQL (NULLs are largest)

**Force NULLs Last in SQL Server:**

```sql
-- Method 1: Using CASE
SELECT name, commission,
       RANK() OVER (ORDER BY 
           CASE WHEN commission IS NULL THEN 1 ELSE 0 END,
           commission DESC
       ) as rnk
FROM sales_reps;

-- Method 2: Using ISNULL/COALESCE with extreme value
SELECT name, commission,
       RANK() OVER (ORDER BY 
           ISNULL(commission, -999999) DESC
       ) as rnk
FROM sales_reps;

-- Method 3: SQL Server 2022+ supports NULLS LAST
SELECT name, commission,
       RANK() OVER (ORDER BY commission DESC NULLS LAST) as rnk
FROM sales_reps;
```

**Why This Matters:**
When ranking sales performance, you don't want NULL commissions (new employees, non-sales staff) to appear as "top performers" or mess up your top-10 list.

---

## **Query Execution Order & Precedence**

### **Q4: The Classic ORDER Trap**

**Question:**
```sql
SELECT department, AVG(salary) as avg_sal
FROM employees
WHERE avg_sal > 50000  -- ERROR HERE
GROUP BY department
HAVING COUNT(*) > 5
ORDER BY avg_sal;
```

**Answer: This fails because of SQL's logical execution order.**

**Complete Logical Execution Order:**
1. **FROM** - Identify source tables
2. **WHERE** - Filter rows (operates on base table columns only)
3. **GROUP BY** - Create groups
4. **HAVING** - Filter groups (can use aggregates)
5. **SELECT** - Compute columns and aliases
6. **DISTINCT** - Remove duplicates
7. **ORDER BY** - Sort results (can use aliases)
8. **LIMIT/TOP** - Restrict row count

**Why It Fails:**
- WHERE clause executes BEFORE SELECT
- At WHERE execution time, `avg_sal` doesn't exist yet
- You can only reference actual table columns in WHERE

**Correct Version:**
```sql
SELECT department, AVG(salary) as avg_sal
FROM employees
GROUP BY department
HAVING AVG(salary) > 50000  -- Use HAVING for aggregates
   AND COUNT(*) > 5
ORDER BY avg_sal;  -- ORDER BY can use aliases
```

**Key Insights:**
- **WHERE** filters individual rows (before grouping)
- **HAVING** filters groups (after grouping)
- **ORDER BY** is the only clause that can reliably use SELECT aliases
- Some databases allow aliases in HAVING, but it's not standard SQL

**Practical Example:**
```sql
-- Get departments with >5 employees earning average >50k
SELECT 
    department, 
    COUNT(*) as emp_count,
    AVG(salary) as avg_sal,
    MAX(salary) as max_sal
FROM employees
WHERE status = 'ACTIVE'        -- Filter rows first
GROUP BY department
HAVING COUNT(*) > 5            -- Filter groups
   AND AVG(salary) > 50000
ORDER BY avg_sal DESC;         -- Sort final results
```

---

### **Q5: CTE Evaluation Timing**

**Answer:**

**Execution Order:**
1. **high_earners CTE** is defined (but not necessarily executed yet)
2. **dept_avg CTE** is defined (can reference high_earners if needed)
3. **Main query** executes, which triggers CTE evaluation

**Important Points:**

**1. CTE Scope:**
```sql
WITH high_earners AS (
    SELECT * FROM employees WHERE salary > 100000
),
dept_avg AS (
    SELECT department, AVG(salary) as avg_sal
    FROM high_earners  -- YES, this is allowed!
    GROUP BY department
)
SELECT * FROM dept_avg;
```
- Later CTEs CAN reference earlier CTEs
- Earlier CTEs CANNOT reference later ones
- Order matters!

**2. CTE Evaluation (Vendor-Specific):**

**SQL Server:**
- CTEs are like "inline views" or "query macros"
- Not materialized by default
- If referenced once: evaluated once
- If referenced twice: might be evaluated twice (optimizer decides)

**PostgreSQL:**
- CTEs are "optimization fences" by default
- Always materialized (before PostgreSQL 12)
- PostgreSQL 12+: Optimizer can inline simple CTEs

**3. Recursive CTE Evaluation:**
```sql
WITH RECURSIVE cte AS (
    SELECT 1 as n  -- Anchor member (executed once)
    UNION ALL
    SELECT n + 1 FROM cte WHERE n < 10  -- Recursive member (loops)
)
SELECT * FROM cte;
```
Execution:
1. Execute anchor member
2. Execute recursive member using anchor results
3. Execute recursive member using previous iteration
4. Repeat until no new rows (or max recursion hit)

**Practical Impact:**
```sql
-- CTE referenced multiple times
WITH sales_data AS (
    SELECT product_id, SUM(amount) as total
    FROM sales
    WHERE sale_date >= '2024-01-01'
    GROUP BY product_id
)
SELECT 
    (SELECT COUNT(*) FROM sales_data) as total_products,
    (SELECT AVG(total) FROM sales_data) as avg_sales,
    (SELECT MAX(total) FROM sales_data) as max_sales;

-- May execute sales_data CTE 3 times in some databases!
-- Better to use:
WITH sales_data AS (...)
SELECT 
    COUNT(*) as total_products,
    AVG(total) as avg_sales,
    MAX(total) as max_sales
FROM sales_data;
```

---

### **Q6: Window Function Execution Phase**

**Question:**
```sql
SELECT name, salary,
       AVG(salary) OVER () as company_avg
FROM employees
WHERE salary > AVG(salary) OVER ();  -- ERROR!
```

**Answer: This fails because window functions execute AFTER WHERE.**

**Logical Query Processing Order (Extended):**
1. FROM
2. WHERE
3. GROUP BY
4. HAVING
5. SELECT
6. **WINDOW FUNCTIONS** ← Execute here!
7. DISTINCT
8. ORDER BY
9. LIMIT/OFFSET

**Why Window Functions Are Special:**
- They operate on the result set AFTER filtering/grouping
- They can't be used in WHERE, GROUP BY, or HAVING
- They can only appear in SELECT and ORDER BY clauses

**Correct Approach - Use Subquery:**
```sql
SELECT name, salary, company_avg
FROM (
    SELECT name, salary,
           AVG(salary) OVER () as company_avg
    FROM employees
) sub
WHERE salary > company_avg;
```

**Or Use CTE:**
```sql
WITH emp_with_avg AS (
    SELECT name, salary,
           AVG(salary) OVER () as company_avg
    FROM employees
)
SELECT * FROM emp_with_avg
WHERE salary > company_avg;
```

**Why This Design?**
Window functions need to see the "final" row set after filtering. If they executed earlier, what window would they compute over?

**Common Mistakes:**
```sql
-- WRONG: Window function in WHERE
WHERE salary > AVG(salary) OVER ()

-- WRONG: Window function in GROUP BY  
GROUP BY RANK() OVER (ORDER BY salary)

-- WRONG: Window function in HAVING
HAVING COUNT(*) > ROW_NUMBER() OVER ()

-- CORRECT: Window function in SELECT
SELECT RANK() OVER (ORDER BY salary) as rnk

-- CORRECT: Window function in ORDER BY
ORDER BY ROW_NUMBER() OVER (ORDER BY salary)
```

**Practical Example:**
```sql
-- Find employees earning more than their department average
SELECT name, department, salary, dept_avg
FROM (
    SELECT 
        name, 
        department, 
        salary,
        AVG(salary) OVER (PARTITION BY department) as dept_avg
    FROM employees
) sub
WHERE salary > dept_avg;
```

---

## **JOIN Questions**

### **Q7: The OUTER JOIN Filter Placement**

**Answer:**

**Query A:**
```sql
SELECT e.name, d.dept_name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.id
WHERE d.budget > 100000;
```
**Result:** Behaves like INNER JOIN!
- LEFT JOIN creates rows with NULL for non-matching departments
- WHERE clause filters out those NULLs (d.budget is NULL)
- Only employees with matching departments AND budget > 100000 remain

**Query B:**
```sql
SELECT e.name, d.dept_name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.id AND d.budget > 100000;
```
**Result:** True LEFT JOIN behavior
- Join condition includes budget filter
- Employees without departments: kept (dept_name = NULL)
- Employees with departments but budget ≤ 100000: kept (dept_name = NULL)
- Only employees with departments AND budget > 100000: have dept_name populated

**Visual Example:**

Employees: John (dept_id=1), Jane (dept_id=2), Bob (dept_id=NULL)
Departments: Dept 1 (budget=150k), Dept 2 (budget=80k)

**Query A Result:**
```
John | Dept 1
```
(Jane and Bob filtered out by WHERE)

**Query B Result:**
```
John | Dept 1
Jane | NULL
Bob  | NULL
```

**Rule of Thumb:**
- **ON clause conditions**: Applied during join (affects which rows match)
- **WHERE clause conditions**: Applied after join (filters final result)
- For OUTER JOINs: Filters on the "outer" table → WHERE clause
- For OUTER JOINs: Filters on the "inner" table → ON clause (if you want to preserve outer rows)

**Practical Application:**
```sql
-- Find all employees, show department name only if budget > 100k
SELECT e.name, d.dept_name
FROM employees e
LEFT JOIN departments d 
    ON e.dept_id = d.id 
    AND d.budget > 100000;

-- Find employees in well-funded departments only
SELECT e.name, d.dept_name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.id
WHERE d.budget > 100000;
-- or simply use INNER JOIN
```

---

### **Q8: Multiple JOIN Types in Sequence**

**Answer:**

**What Happens:**
```sql
SELECT *
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id      -- Step 1
LEFT JOIN shipments s ON o.order_id = s.order_id    -- Step 2
INNER JOIN products p ON o.product_id = p.id;       -- Step 3
```

**Execution Flow:**
1. **INNER JOIN customers**: Filters orders to only those with matching customers
2. **LEFT JOIN shipments**: Adds shipment data, keeps rows even if no shipment (s.* = NULL)
3. **INNER JOIN products**: **FILTERS OUT rows where p.id doesn't match**

**Critical Insight:**
If Step 3 (INNER JOIN products) finds no match, that row is **eliminated**, even though Step 2 was a LEFT JOIN.

**Example Data:**

Orders: 
- Order 1: customer_id=1, product_id=1
- Order 2: customer_id=1, product_id=999 (product doesn't exist)

After Step 2 (LEFT JOIN shipments):
- Order 1: customer data, shipment data or NULL
- Order 2: customer data, NULL shipment

After Step 3 (INNER JOIN products):
- Order 1: Kept (product exists)
- Order 2: **ELIMINATED** (product_id=999 doesn't exist)

**Does Order Matter?**
YES! Join order affects intermediate result sets and can change query results.

```sql
-- Different result!
SELECT *
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id
INNER JOIN products p ON o.product_id = p.id
LEFT JOIN shipments s ON o.order_id = s.order_id;
-- Now all orders with valid customers AND products are kept
-- Shipment data is optional
```

**Performance Implications:**
```sql
-- Poor: Large intermediate result set
SELECT *
FROM large_table (1M rows)
LEFT JOIN medium_table (100k rows)
INNER JOIN small_table (1k rows);

-- Better: Filter early with INNER JOIN
SELECT *
FROM large_table (1M rows)
INNER JOIN small_table (1k rows)  -- Reduces to ~10k rows
LEFT JOIN medium_table (100k rows);
```

**Best Practice:**
- Place INNER JOINs before LEFT JOINs when possible
- Use parentheses for complex joins if your database supports it
- Test join order for performance

---

### **Q9: Self-Join Performance Trap**

**Answer:**

**Version A (JOIN):**
```sql
SELECT e.name
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

**Version B (Subquery):**
```sql
SELECT e.name
FROM employees e
WHERE e.salary > (SELECT salary FROM employees WHERE id = e.manager_id);
```

**Performance Comparison:**

**Modern Optimizers (SQL Server, PostgreSQL):**
- Often produce IDENTICAL execution plans
- Both may use index seeks on manager_id
- Optimizer can transform subquery into join (or vice versa)

**When Version A is Better:**
- You need multiple columns from manager record
- Clearer execution plan (explicit join visible)
- Better for complex multi-table scenarios

**When Version B is Better:**
- Potentially better with NULL handling (see below)
- Clearer intent for single-value comparison
- Some older optimizers handle it better

**NULL Handling Difference:**

**Version A:**
```sql
-- If manager_id is NULL, JOIN fails to match
-- Employee with NULL manager_id is excluded
```

**Version B:**
```sql
-- Subquery returns NULL
-- e.salary > NULL evaluates to UNKNOWN (false)
-- Employee with NULL manager_id is excluded
```

Both exclude NULLs, but to include CEO (no manager):

**Version A Fix:**
```sql
SELECT e.name
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary OR e.manager_id IS NULL;
```

**Version B Fix:**
```sql
SELECT e.name
FROM employees e
WHERE e.salary > (SELECT salary FROM employees WHERE id = e.manager_id)
   OR e.manager_id IS NULL;
```

**Execution Plan Analysis:**

```sql
-- Both typically use:
-- 1. Index Seek on employees (main scan)
-- 2. Index Seek on employees (manager lookup) for EACH row
-- 3. Filter on salary comparison

-- For 10,000 employees:
-- Version A: Hash Join or Nested Loop (10,000 index seeks)
-- Version B: Correlated Subquery (10,000 index seeks)
-- Result: Similar performance
```

**When They Differ (Edge Case):**
```sql
-- If multiple managers have same ID (data error), Version A multiplies rows
-- Version B fails (subquery returns multiple rows)
```

**Recommendation:**
- Use JOIN for readability and multi-column access
- Modern databases optimize both similarly
- Test with EXPLAIN/EXPLAIN ANALYZE for your specific data

---

## **HAVING Clause Tricks**

### **Q10: HAVING without GROUP BY**

**Question:**
```sql
SELECT AVG(salary)
FROM employees
HAVING AVG(salary) > 50000;
```

**Answer: This IS valid SQL!**

**What It Does:**
- Treats entire table as single group
- Computes AVG(salary) across all rows
- Returns result only if average > 50000
- Returns empty result set if condition fails

**Example:**

If table has salaries: 40k, 50k, 60k, 70k
- AVG = 55k
- Condition 55k > 50k is TRUE
- Returns: 55000

If table has salaries: 30k, 40k, 50k, 40k
- AVG = 40k
- Condition 40k > 50k is FALSE
- Returns: (empty result set)

**Difference from WHERE:**

```sql
-- HAVING without GROUP BY
SELECT AVG(salary)
FROM employees
HAVING AVG(salary) > 50000;
-- Returns: 1 row with average (if condition true), or 0 rows

-- WHERE version (INVALID!)
SELECT AVG(salary)
FROM employees
WHERE AVG(salary) > 50000;
-- ERROR: Cannot use aggregates in WHERE

-- Closest WHERE equivalent (different logic)
SELECT AVG(salary)
FROM employees
WHERE salary > 50000;
-- Returns: Average of salaries > 50k (different calculation!)
```

**Real-World Usage:**
```sql
-- Check if company-wide average meets threshold
SELECT 
    COUNT(*) as total_employees,
    AVG(salary) as avg_salary,
    MIN(salary) as min_salary,
    MAX(salary) as max_salary
FROM employees
HAVING AVG(salary) > 50000;
-- Returns row only if company average > 50k
-- Useful for validation queries

-- Practical example: Ensure data quality
SELECT COUNT(*) as error_count
FROM transactions
WHERE status = 'ERROR'
HAVING COUNT(*) = 0;
-- Returns row only if no errors (count = 0)
-- Empty result means there are errors!
```

**Vendor Differences:**
- SQL Server: Supported
- PostgreSQL: Supported
- MySQL: Supported
- Oracle: Supported (standard SQL)

---

### **Q11: Column Ambiguity in HAVING**

**Answer:**

```sql
SELECT department, COUNT(*) as employee_count
FROM employees
GROUP BY department
HAVING employee_count > 5;  -- Alias in HAVING
```

**Vendor Support:**

**SQL Server:** ✅ Supported
```sql
HAVING employee_count > 5  -- Works fine
```

**PostgreSQL:** ❌ Not supported in older versions, ✅ Supported in 9.5+
```sql
-- PostgreSQL < 9.5: ERROR
HAVING employee_count > 5

-- Must use:
HAVING COUNT(*) > 5
```

**MySQL:** ✅ Supported
```sql
HAVING employee_count > 5  -- Works
```

**Oracle:** ❌ Not supported
```sql
HAVING employee_count > 5  -- ERROR: invalid identifier
-- Must use:
HAVING COUNT(*) > 5
```

**SQL Standard Approach:**
Use the actual aggregate function, not aliases:

```sql
SELECT department, COUNT(*) as employee_count
FROM employees
GROUP BY department
HAVING COUNT(*) > 5;  -- Standard SQL, works everywhere
```

**Why the Difference?**
- SQL execution order: HAVING executes before SELECT
- Aliases are defined in SELECT
- Some vendors "extend" SQL to allow alias references in HAVING
- Oracle is strictest about following logical execution order

**Best Practice:**
```sql
-- Portable (works everywhere)
SELECT 
    department, 
    COUNT(*) as employee_count,
    AVG(salary) as avg_salary
FROM employees
GROUP BY department
HAVING COUNT(*) > 5 
   AND AVG(salary) > 50000;

-- SQL Server/MySQL friendly (not portable)
SELECT 
    department, 
    COUNT(*) as employee_count,
    AVG(salary) as avg_salary
FROM employees
GROUP BY department
HAVING employee_count > 5 
   AND avg_salary > 50000;
```

**Workaround for Complex Expressions:**
```sql
-- If you have complex aggregate expression
SELECT 
    department,
    SUM(CASE WHEN status = 'ACTIVE' THEN salary ELSE 0 END) as active_salary
FROM employees
GROUP BY department
HAVING SUM(CASE WHEN status = 'ACTIVE' THEN salary ELSE 0 END) > 100000;

-- Repetitive! Use CTE or subquery:
WITH dept_salaries AS (
    SELECT 
        department,
        SUM(CASE WHEN status = 'ACTIVE' THEN salary ELSE 0 END) as active_salary
    FROM employees
    GROUP BY department
)
SELECT * FROM dept_salaries
WHERE active_salary > 100000;  -- Now use WHERE with alias
```

---

### **Q12: HAVING with Window Functions**

**Question:**
```sql
SELECT department, COUNT(*) as cnt
FROM employees
GROUP BY department
HAVING COUNT(*) > AVG(COUNT(*)) OVER ();
```

**Answer: This will FAIL!**

**Why It Fails:**
- HAVING executes before window functions
- Window functions execute after HAVING
- Cannot use window functions in HAVING clause

**Execution Order:**
1. FROM
2. WHERE
3. GROUP BY
4. **HAVING** ← Executes here
5. SELECT
6. **Window Functions** ← Execute here (after HAVING!)
7. ORDER BY

**Correct Approach - Use Subquery:**

```sql
SELECT department, cnt
FROM (
    SELECT 
        department, 
        COUNT(*) as cnt,
        AVG(COUNT(*)) OVER () as avg_cnt
    FROM employees
    GROUP BY department
) sub
WHERE cnt > avg_cnt;
```

**Or Use CTE:**

```sql
WITH dept_counts AS (
    SELECT 
        department, 
        COUNT(*) as cnt,
        AVG(COUNT(*)) OVER () as avg_cnt
    FROM employees
    GROUP BY department
)
SELECT department, cnt
FROM dept_counts
WHERE cnt > avg_cnt;
```

**Step-by-Step Execution:**

```sql
-- Step 1: Group and count
GROUP BY department
-- Result: IT (15), Sales (20), HR (8)

-- Step 2: Add window function
AVG(COUNT(*)) OVER () 
-- Computes: (15 + 20 + 8) / 3 = 14.33

-- Step 3: Filter in outer query
WHERE cnt > avg_cnt
-- Result: IT (15 > 14.33), Sales (20 > 14.33)
```

**Practical Example:**

```sql
-- Find departments with above-average employee count
WITH dept_stats AS (
    SELECT 
        department,
        COUNT(*) as emp_count,
        AVG(COUNT(*)) OVER () as avg_dept_size,
        MAX(COUNT(*)) OVER () as largest_dept
    FROM employees
    GROUP BY department
)
SELECT 
    department,
    emp_count,
    avg_dept_size,
    ROUND(100.0 * emp_count / largest_dept, 2) as pct_of_largest
FROM dept_stats
WHERE emp_count > avg_dept_size
ORDER BY emp_count DESC;
```

**Common Misunderstandings:**

```sql
-- WRONG: Window function in HAVING
HAVING ROW_NUMBER() OVER (ORDER BY COUNT(*)) <= 3

-- WRONG: Window function in GROUP BY
GROUP BY RANK() OVER (ORDER BY salary)

-- CORRECT: Window function in SELECT
SELECT RANK() OVER (ORDER BY COUNT(*)) as rnk

-- CORRECT: Window function then filter
SELECT * FROM (
    SELECT *, RANK() OVER (ORDER BY cnt) as rnk
    FROM (SELECT department, COUNT(*) as cnt FROM employees GROUP BY department)
) WHERE rnk <= 3
```

---

## **Memory & Performance Optimization**

### **Q13: CTE vs Temp Table vs Subquery**

**Question:**
```sql
WITH sales_summary AS (
    SELECT product_id, SUM(amount) as total
    FROM sales
    GROUP BY product_id
)
SELECT * FROM sales_summary WHERE total > 1000
UNION ALL
SELECT * FROM sales_summary WHERE total <= 1000;
```

**Answer: CTE referenced twice - materialization depends on database!**

**SQL Server:**
- CTEs are **NOT materialized** by default
- Think of them as "macros" that get expanded
- If referenced twice → executed twice (usually)
- Optimizer MAY choose to spool results (rare)

```sql
-- Executed TWICE in SQL Server
WITH sales_summary AS (SELECT product_id, SUM(amount) as total FROM sales GROUP BY product_id)
SELECT * FROM sales_summary WHERE total > 1000
UNION ALL
SELECT * FROM sales_summary WHERE total <= 1000;

-- Fix: Use temp table
SELECT product_id, SUM(amount) as total
INTO #sales_summary
FROM sales
GROUP BY product_id;

SELECT * FROM #sales_summary WHERE total > 1000
UNION ALL
SELECT * FROM #sales_summary WHERE total <= 1000;
-- Executed ONCE, queried twice
```

**PostgreSQL (< 12):**
- CTEs are **ALWAYS materialized** ("optimization fence")
- Executed ONCE, even if referenced multiple times
- Can be slower for simple CTEs that could be inlined

**PostgreSQL (12+):**
- Simple CTEs can be inlined
- Complex CTEs still materialized
- Use `MATERIALIZED` hint to force behavior:

```sql
-- Force materialization (execute once)
WITH sales_summary AS MATERIALIZED (
    SELECT product_id, SUM(amount) as total FROM sales GROUP BY product_id
)
SELECT * FROM sales_summary WHERE total > 1000
UNION ALL
SELECT * FROM sales_summary WHERE total <= 1000;

-- Force inlining (execute twice, let optimizer optimize)
WITH sales_summary AS NOT MATERIALIZED (...)
```

**MySQL:**
- Before 8.0: No CTE support
- MySQL 8.0+: Similar to SQL Server (not materialized)

**Memory Consumption Comparison:**

| Method | Memory Usage | Execution Count | Pros | Cons |
|--------|--------------|-----------------|------|------|
| **CTE (SQL Server)** | Low (streaming) | 2x | Clean syntax | Potential waste |
| **Temp Table** | Medium (stored) | 1x | Guaranteed single execution | I/O overhead, tempdb contention |
| **Table Variable** | Medium (memory/tempdb) | 1x | No transaction log | No statistics, poor estimates |
| **CTE (PostgreSQL <12)** | Medium (materialized) | 1x | Automatic optimization | Can't be optimized away |

**When to Use Each:**

```sql
-- Use CTE when: Simple query, referenced once
WITH recent_orders AS (
    SELECT * FROM orders WHERE order_date > '2024-01-01'
)
SELECT * FROM recent_orders WHERE status = 'PENDING';

-- Use Temp Table when: Complex query, referenced multiple times, needs indexes
SELECT product_id, SUM(amount) as total
INTO #sales_summary
FROM sales
WHERE sale_date >= '2024-01-01'
GROUP BY product_id;

CREATE INDEX ix_total ON #sales_summary(total);

SELECT * FROM #sales_summary WHERE total > 1000;
SELECT * FROM #sales_summary WHERE total <= 1000;
SELECT AVG(total) FROM #sales_summary;

-- Use Table Variable when: Small dataset (<1000 rows), simple operations
DECLARE @summary TABLE (product_id INT, total DECIMAL);
INSERT INTO @summary
SELECT product_id, SUM(amount) FROM sales GROUP BY product_id;

SELECT * FROM @summary;
```

**Memory Impact Example:**

```sql
-- Sales table: 100M rows

-- CTE (executed twice) - SQL Server
WITH sales_summary AS (
    SELECT product_id, SUM(amount) as total 
    FROM sales  -- 100M row scan
    GROUP BY product_id  -- Hash aggregate: ~500MB memory
)
SELECT * FROM sales_summary WHERE total > 1000  -- First scan
UNION ALL
SELECT * FROM sales_summary WHERE total <= 1000;  -- Second scan!
-- Total: 2 scans, ~1GB memory (twice the hash aggregate)

-- Temp Table (executed once)
SELECT product_id, SUM(amount) as total
INTO #sales_summary
FROM sales  -- 100M row scan ONCE
GROUP BY product_id;  -- Hash aggregate: ~500MB memory
-- Total: 1 scan, ~500MB memory + tempdb storage
```

**Best Practice:**
```sql
-- For single reference: CTE
-- For multiple references: Temp table (if expensive query)
-- For multiple references: CTE MATERIALIZED (PostgreSQL 12+)
```

---

### **Q14: EXISTS vs IN for Large Datasets**

**Answer:**

```sql
-- Version A: EXISTS
SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);

-- Version B: IN
SELECT * FROM customers c
WHERE c.id IN (SELECT customer_id FROM orders);
```

**Memory Usage:**

**EXISTS:**
- **Semi-join** operation
- Stops scanning as soon as first match found
- No materialization of subquery results
- Memory: Minimal (streaming)
- For customer with 1000 orders: Checks 1 order, stops

**IN:**
- **Subquery materialized** (in many databases)
- All distinct customer_ids loaded into memory/temp structure
- Then hash match against customers
- Memory: Full distinct list of customer_ids
- If orders has 10M rows → might store 100K distinct customer_ids

**NULL Handling (Critical Difference!):**

```sql
-- Test data:
customers: id IN (1, 2, 3, 4)
orders: customer_id IN (1, 2, NULL, NULL)

-- EXISTS: Returns customers 1, 2
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id)
-- Logic: NULLs don't match anything, ignored

-- IN: Returns customers 1, 2
WHERE c.id IN (SELECT customer_id FROM orders)
-- Logic: NULLs are filtered out by IN

-- But with NOT IN - HUGE DIFFERENCE:

-- NOT EXISTS: Returns customers 3, 4
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id)

-- NOT IN: Returns NOTHING (!)
WHERE c.id NOT IN (SELECT customer_id FROM orders)
-- Logic: 
-- c.id NOT IN (1, 2, NULL)
-- = c.id <> 1 AND c.id <> 2 AND c.id <> NULL
-- = c.id <> 1 AND c.id <> 2 AND UNKNOWN
-- = UNKNOWN (because of NULL)
-- Result: NO ROWS
```

**Performance Comparison:**

**Customers: 100K rows, Orders: 10M rows (1M distinct customers)**

```sql
-- EXISTS (better for large order sets)
SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
-- Execution:
-- 1. Scan customers (100K rows)
-- 2. For each customer, seek orders index (100K seeks)
-- 3. Stop at first match
-- Memory: ~Constant (just index buffers)
-- I/O: 100K index seeks

-- IN (better if few distinct customers, problematic if many)
SELECT * FROM customers c
WHERE c.id IN (SELECT customer_id FROM orders);
-- Execution:
-- 1. Scan orders, get distinct customer_ids (10M rows → 1M distinct)
-- 2. Store 1M IDs in memory hash table (~8MB)
-- 3. Scan customers, hash match (100K rows)
-- Memory: ~8MB for hash table
-- I/O: Full order scan
```

**When IN is Better:**
```sql
-- Subquery returns very small set
SELECT * FROM customers c
WHERE c.id IN (1, 2, 3, 4, 5);
-- Faster than EXISTS, uses index seek directly

-- Or with small reference table
SELECT * FROM customers c
WHERE c.status IN (SELECT status FROM active_statuses);
-- If active_statuses has 3 rows, IN is fine
```

**When EXISTS is Better:**
```sql
-- Large subquery results
SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);

-- When you need correlated logic
SELECT * FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.customer_id = c.id 
    AND o.amount > c.credit_limit
);
```

**Modern Optimizer Behavior:**
- SQL Server: Often converts IN to EXISTS internally
- PostgreSQL: Smart optimization, both similar
- MySQL: Historically poor IN performance, better in 8.0+
- Oracle: Mature optimizer, handles both well

**Best Practice:**

```sql
-- Use EXISTS for:
-- 1. Large subquery results
-- 2. Correlated subqueries
-- 3. NOT EXISTS (always safer than NOT IN)

-- Use IN for:
-- 1. Small literal lists
-- 2. Small subquery results (<1000 rows)
-- 3. Better readability for simple cases

-- NEVER use NOT IN with nullable columns:
-- BAD (returns unexpected results if NULLs present)
WHERE c.id NOT IN (SELECT customer_id FROM orders)

-- GOOD
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id)

-- Or explicitly handle NULLs:
WHERE c.id NOT IN (SELECT customer_id FROM orders WHERE customer_id IS NOT NULL)
```

---

### **Q15: Index Impact on Memory**

**Question:**
```sql
SELECT * FROM large_table WHERE status = 'ACTIVE' ORDER BY created_date DESC;
```

**You can create ONE index. Choose:**
1. Index on (status)
2. Index on (created_date)
3. Composite index on (status, created_date)

**Answer: Option 3 - Composite index (status, created_date)**

**Detailed Analysis:**

**Option 1: Index on (status)**
```sql
CREATE INDEX ix_status ON large_table(status);
```

**Execution Plan:**
1. Index seek on status = 'ACTIVE' (fast)
2. Retrieve matching rows (maybe 50% of table)
3. **SORT operation** on created_date (EXPENSIVE!)

**Memory Usage:**
- Sort buffer required: potentially GBs
- If data doesn't fit in memory → Tempdb/disk spill
- Large_table = 10M rows, 50% active = 5M rows to sort
- Assuming 200 bytes/row = 1GB sort buffer needed!

**Option 2: Index on (created_date)**
```sql
CREATE INDEX ix_created_date ON large_table(created_date);
```

**Execution Plan:**
1. Index scan on created_date DESC (ordered)
2. Filter status = 'ACTIVE' (scan entire table)
3. Stop when enough rows found? NO - must scan all to find all ACTIVE

**Memory Usage:**
- No sort needed (index provides order)
- But must scan entire index to filter status
- If only 1% of rows are ACTIVE → scan 10M rows to find 100K

**Option 3: Composite Index (status, created_date)**
```sql
CREATE INDEX ix_status_created_date ON large_table(status, created_date DESC);
```

**Execution Plan:**
1. Index seek on status = 'ACTIVE'
2. Data already ordered by created_date (within status group)
3. No sort needed!
4. Efficient retrieval

**Memory Usage:**
- **Minimal!** No sort buffer needed
- Index seek directly to relevant rows
- Rows returned in correct order
- Memory: Just buffer pool pages for index reads

**Memory Comparison (10M row table, 5M ACTIVE rows):**

| Index Type | Sort Buffer | Temp Space | Total Memory |
|------------|-------------|------------|--------------|
| (status) | 1GB | Possible spill | 1GB+ |
| (created_date) | 0 | 0 | 100MB (buffer pool) |
| **(status, created_date)** | **0** | **0** | **50MB (buffer pool)** |

**Why Composite is Best:**

```sql
-- Index structure visualized:
(status, created_date)
-----------------------------------
| ACTIVE, 2024-03-15 | → Row ptr |
| ACTIVE, 2024-03-14 | → Row ptr |
| ACTIVE, 2024-03-13 | → Row ptr |
| ...                             |
| ACTIVE, 2023-01-01 | → Row ptr |
| INACTIVE, 2024-03-15 |         |
| INACTIVE, 2024-03-14 |         |
-----------------------------------

-- Query execution:
-- 1. Seek to first "ACTIVE" entry
-- 2. Scan forward (data already sorted!)
-- 3. Stop when hit "INACTIVE" or need only top N rows
```

**Include Columns for SELECT *:**

```sql
-- Even better: covering index (no table lookup needed)
CREATE INDEX ix_status_created_date_covering 
ON large_table(status, created_date DESC)
INCLUDE (id, name, email, other_columns);

-- Now the entire query is served from index
-- Memory: Even less (no table page reads)
```

**Sort Operation Memory Details:**

When sort is required:
```sql
-- SQL Server: Uses sort buffer (part of memory grant)
-- If data > buffer: Spills to tempdb
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

SELECT * FROM large_table 
WHERE status = 'ACTIVE' 
ORDER BY created_date DESC;

-- Output might show:
-- Table 'Worktable'. Scan count 0, logical reads 50000  ← Tempdb spill!
-- Sort Warnings: true
```

**Real-World Impact:**

```sql
-- Monitoring query with bad index:
SELECT TOP 100 * 
FROM transactions 
WHERE status = 'PENDING'
ORDER BY created_date DESC;

-- With index on (status) only:
-- - Memory grant: 200MB
-- - Execution time: 5 seconds
-- - Tempdb reads: 50,000 pages

-- With index on (status, created_date DESC):
-- - Memory grant: 10MB
-- - Execution time: 0.1 seconds  
-- - Tempdb reads: 0
```

**Additional Considerations:**

```sql
-- If query also filtered by date range:
WHERE status = 'ACTIVE' 
  AND created_date >= '2024-01-01'
ORDER BY created_date DESC;

-- Same composite index is perfect!
-- Seek to (ACTIVE, 2024-01-01), scan forward

-- If query filtered by created_date first:
WHERE created_date >= '2024-01-01'
  AND status = 'ACTIVE'
ORDER BY created_date DESC;

-- Index (created_date, status) might be better
-- But original is still usable with index skip scan
```

**Answer Summary:**
**Choose composite index (status, created_date DESC)** because:
1. Eliminates expensive sort operation
2. Minimal memory usage (no sort buffer)
3. No tempdb spill risk
4. Fastest execution time
5. Scales well as table grows

---

### **Q16: SELECT * vs SELECT specific columns**

**Answer:**

**Beyond obvious data transfer, memory impact during query execution:**

**1. Query Execution Plan Memory:**

```sql
-- Table: 50 columns, 1GB row size, 10M rows

-- SELECT * query:
SELECT * FROM large_table WHERE status = 'ACTIVE';

-- Execution plan must allocate memory for ALL 50 columns:
-- - Column metadata
-- - Data type conversions
-- - NULL bitmap
-- - Variable length column tracking
```

**Memory Grant Calculation:**
- SQL Server estimates memory needed based on:
  - Number of rows expected
  - **Size of each row (all 50 columns)**
  - Operations performed (sort, hash, etc.)

```sql
-- Example memory grants:
-- SELECT *: 500MB grant (all 50 columns × estimated rows)
-- SELECT col1, col2, col3: 50MB grant (just 3 columns)
```

**2. Hash Join Operations:**

```sql
-- Query with SELECT *:
SELECT * 
FROM large_table lt
JOIN another_table at ON lt.id = at.large_table_id
WHERE lt.status = 'ACTIVE';

-- Hash join builds hash table in memory:
-- Hash table stores: ALL 50 columns from large_table
-- Size: 50 columns × row size × number of rows
-- Memory: Could be GIGABYTES

-- Query with SELECT specific:
SELECT lt.id, lt.name, at.description
FROM large_table lt
JOIN another_table at ON lt.id = at.large_table_id
WHERE lt.status = 'ACTIVE';

-- Hash table stores: Only id, name (2 columns)
-- Size: 2 columns × row size × number of rows  
-- Memory: Much smaller, maybe MEGABYTES
```

**Real Example:**

```sql
-- Products table: 50 columns, 100MB row size
-- Orders table: 1M rows

-- BAD (SELECT *):
SELECT *
FROM orders o
JOIN products p ON o.product_id = p.id;

-- Execution:
-- 1. Build hash table with ALL 50 product columns
-- 2. Memory: 100MB × 10K products = 1GB
-- 3. Probe with 1M order rows
-- 4. Result set: 50 columns × 1M rows = massive

-- GOOD (specific columns):
SELECT o.order_id, o.quantity, p.product_name, p.price
FROM orders o
JOIN products p ON o.product_id = p.id;

-- Execution:
-- 1. Build hash table with just product_name, price
-- 2. Memory: 50 bytes × 10K products = 500KB
-- 3. Probe with 1M order rows
-- 4. Result set: 4 columns × 1M rows = manageable
```

**3. Sort Operations:**

```sql
-- SELECT * with ORDER BY:
SELECT * 
FROM large_table 
WHERE status = 'ACTIVE'
ORDER BY created_date;

-- Sort buffer must hold ALL 50 columns
-- Memory: 50 columns × row width × number of rows
-- 1KB row × 1M rows = 1GB sort buffer

-- Specific columns:
SELECT id, name, created_date
FROM large_table
WHERE status = 'ACTIVE'
ORDER BY created_date;

-- Sort buffer holds just 3 columns
-- Memory: 100 bytes × 1M rows = 100MB
```

**4. Execution Plan Complexity:**

```sql
-- With SELECT *:
-- - More columns to track
-- - Wider row estimates
-- - Larger intermediate result sets
-- - Higher probability of underestimating memory

-- Example query plan memory:
SELECT * FROM large_table WHERE id IN (SELECT id FROM temp_table);

-- Plan execution:
-- 1. Scan temp_table
-- 2. Build hash table: id + ALL 50 columns (unnecessary!)
-- 3. Memory grant: overestimated or underestimated
```

**5. Blocking Operators:**

**Blocking operators** (must consume all input before producing output):
- Sort
- Hash aggregate
- Hash join build phase

```sql
-- SELECT * makes all blocking operators heavier:

-- Sort with SELECT *:
SELECT * FROM large_table ORDER BY status, created_date;
-- Sorts all 50 columns (even if only ordering by 2)
-- Memory: Entire row width in sort buffer

-- Hash aggregate with SELECT *:
SELECT * FROM large_table GROUP BY status;
-- ERROR! Can't GROUP BY without aggregating
-- But concept: grouping wide rows is expensive

-- Better example:
SELECT status, COUNT(*), AVG(price), MAX(quantity)
FROM large_table
GROUP BY status;
-- Hash aggregate only stores: status + 3 running totals
-- Memory: Minimal per group
```

**6. Query Execution Buffers:**

```sql
-- SQL Server memory architecture:
-- - Buffer pool (data pages)
-- - Plan cache
-- - Memory grants (sort, hash)
-- - Work tables (tempdb)

-- SELECT * increases ALL of these:

-- Buffer pool:
-- Must load ALL columns into memory
-- 50 columns vs 3 columns = 16x more pages

-- Plan cache:
-- Larger execution plans
-- More column metadata

-- Memory grants:
-- Wider rows = larger grant requests
-- Can cause memory waits

-- Work tables (tempdb):
-- If sort/hash spills: writes ALL columns to tempdb
-- 50 columns × spill = massive I/O
```

**Memory Impact Summary:**

| Operation | SELECT * (50 cols) | SELECT 3 cols | Savings |
|-----------|-------------------|---------------|---------|
| Hash Join | 1GB | 100MB | 90% |
| Sort Buffer | 1GB | 100MB | 90% |
| Memory Grant | 500MB | 50MB | 90% |
| Buffer Pool | 800MB | 100MB | 87.5% |
| Tempdb Spill | 2GB | 200MB | 90% |

**Real-World Scenario:**

```sql
-- Application fetches user data (users table: 50 columns)
-- Only displays: username, email, last_login

-- BAD (fetches all 50 columns):
SELECT * FROM users WHERE status = 'ACTIVE';
-- Network: 50 columns × 100KB row × 10K users = 500MB transferred
-- Memory: Large memory grant, potential spill
-- Execution: 2 seconds

-- GOOD:
SELECT username, email, last_login 
FROM users 
WHERE status = 'ACTIVE';
-- Network: 3 columns × 1KB row × 10K users = 10MB transferred  
-- Memory: Small memory grant, no spill
-- Execution: 0.1 seconds
```

**Exceptions (when SELECT * is okay):**

```sql
-- 1. Development/debugging:
SELECT * FROM small_table WHERE id = 123;
-- Okay for quick checks

-- 2. Small tables (<100 rows, <10 columns):
SELECT * FROM config_settings;

-- 3. Actually need all columns:
-- (But be explicit anyway for maintenance)

-- 4. Covering index query:
-- If index covers all columns, SELECT * from index is efficient
-- (But still bad practice)
```

**Best Practice:**

```sql
-- ALWAYS specify columns explicitly:
SELECT 
    id,
    username,
    email,
    created_date
FROM users
WHERE status = 'ACTIVE';

-- Benefits:
-- 1. Reduced memory usage across ALL operations
-- 2. Faster execution (less data movement)
-- 3. Better execution plans
-- 4. More accurate cardinality estimates
-- 5. Self-documenting code
-- 6. Index covering opportunities
-- 7. Future-proof (new columns don't break code)
```

---

### **Q17: DISTINCT vs GROUP BY**

**Answer:**

```sql
-- Version A: DISTINCT
SELECT DISTINCT customer_id FROM orders;

-- Version B: GROUP BY
SELECT customer_id FROM orders GROUP BY customer_id;
```

**Memory Usage - They're Often Identical!**

**SQL Server:**
- Both use **hash aggregate** or **stream aggregate**
- Identical execution plans in most cases
- Same memory usage

```sql
-- Execution plan (both versions):
-- 1. Scan orders table
-- 2. Hash Aggregate on customer_id
-- 3. Remove duplicates
-- Memory: Hash table with distinct customer_ids
```

**PostgreSQL:**
- DISTINCT → HashAggregate or Unique operator
- GROUP BY → HashAggregate or GroupAggregate
- Usually identical plans

**MySQL:**
- Historically, DISTINCT could be faster
- Modern MySQL (8.0+): similar optimization

**When They Differ:**

**1. Multiple Columns:**

```sql
-- DISTINCT on multiple columns:
SELECT DISTINCT customer_id, order_date 
FROM orders;
-- Memory: Hash table with (customer_id, order_date) pairs

-- GROUP BY equivalent:
SELECT customer_id, order_date 
FROM orders 
GROUP BY customer_id, order_date;
-- Memory: Identical hash table
```

**2. With Aggregates (ONLY GROUP BY):**

```sql
-- GROUP BY with aggregate:
SELECT customer_id, COUNT(*) as order_count
FROM orders
GROUP BY customer_id;
-- Memory: Hash table with customer_id + running count

-- DISTINCT doesn't support this:
SELECT DISTINCT customer_id, COUNT(*)  -- ERROR!
FROM orders;
```

**3. Sorting Behavior:**

```sql
-- DISTINCT might use sort:
SELECT DISTINCT customer_id 
FROM orders
ORDER BY customer_id;
-- Plan: Sort → Remove duplicates (stream aggregate)
-- Memory: Sort buffer

-- GROUP BY with sort:
SELECT customer_id
FROM orders
GROUP BY customer_id
ORDER BY customer_id;
-- Plan: Hash aggregate → Sort
-- Or: Sort → Stream aggregate
-- Memory: Similar to DISTINCT
```

**Performance Comparison (10M orders, 1M distinct customers):**

```sql
-- Both versions:
-- 1. Scan 10M rows
-- 2. Build hash table of 1M distinct values
-- 3. Memory: ~8MB (assuming 8 bytes per customer_id)
-- 4. Return 1M rows

-- Identical performance and memory!
```

**When GROUP BY Uses More Memory:**

```sql
-- GROUP BY with multiple aggregates:
SELECT 
    customer_id,
    COUNT(*) as order_count,
    SUM(amount) as total_amount,
    AVG(amount) as avg_amount,
    MIN(order_date) as first_order,
    MAX(order_date) as last_order
FROM orders
GROUP BY customer_id;

-- Memory: Hash table with:
-- - customer_id (8 bytes)
-- - count accumulator (4 bytes)
-- - sum accumulator (8 bytes)
-- - avg accumulator (8 bytes)
-- - min value (8 bytes)
-- - max value (8 bytes)
-- Total: ~44 bytes × 1M customers = 44MB
```

**When DISTINCT Uses More Memory (Rare):**

```sql
-- Some databases materialize DISTINCT before ORDER BY:
SELECT DISTINCT customer_id
FROM orders
ORDER BY customer_id DESC;

-- Possible plan:
-- 1. Hash aggregate (8MB)
-- 2. Sort distinct values (additional memory)
-- vs GROUP BY might combine operations
```

**Execution Plan Comparison:**

```sql
-- SQL Server - CHECK THE PLAN:
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

-- Version A:
SELECT DISTINCT customer_id FROM orders;
-- Plan: Hash Match (Aggregate)
-- Memory Grant: 8MB

-- Version B:
SELECT customer_id FROM orders GROUP BY customer_id;
-- Plan: Hash Match (Aggregate)
-- Memory Grant: 8MB

-- IDENTICAL!
```

**Always Equivalent?**

Almost always, but edge cases:

```sql
-- 1. DISTINCT with ORDER BY non-selected column (PostgreSQL):
SELECT DISTINCT customer_id
FROM orders
ORDER BY order_date;  -- ERROR in some databases!
-- order_date not in SELECT list

SELECT customer_id
FROM orders
GROUP BY customer_id
ORDER BY MAX(order_date);  -- Works!

-- 2. DISTINCT with LIMIT/TOP:
SELECT DISTINCT TOP 100 customer_id FROM orders;
-- Might stop early (after 100 distinct found)

SELECT TOP 100 customer_id FROM orders GROUP BY customer_id;
-- Same behavior

-- 3. DISTINCT on expressions:
SELECT DISTINCT UPPER(customer_name) FROM customers;
-- Hash aggregate on computed value

SELECT UPPER(customer_name) FROM customers GROUP BY UPPER(customer_name);
-- Identical
```

**Memory Savings Strategies:**

**Neither DISTINCT nor GROUP BY - Use EXISTS:**

```sql
-- Find customers who have placed orders:

-- SLOW (materializes all customer_ids):
SELECT DISTINCT customer_id FROM orders;
-- Memory: 8MB for 1M distinct IDs

-- FAST (no materialization):
SELECT c.id
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
-- Memory: Minimal (streaming semi-join)
-- Stops at first match per customer
```

**Pre-aggregate to reduce memory:**

```sql
-- Bad: Distinct on large result set
SELECT DISTINCT product_id
FROM order_items
WHERE order_date >= '2024-01-01';
-- Scans millions of rows

-- Better: Filter first
SELECT DISTINCT product_id
FROM order_items
WHERE order_date >= '2024-01-01'
  AND quantity > 0;
-- Fewer rows to deduplicate

-- Best: Use appropriate indexes
CREATE INDEX ix_date_product ON order_items(order_date, product_id);
-- Index-only scan, very efficient
```

**Best Practices:**

```sql
-- Use DISTINCT when:
-- - Simple duplicate removal
-- - No aggregates needed
-- - Clearer intent

SELECT DISTINCT status FROM orders;

-- Use GROUP BY when:
-- - Need aggregates
-- - Complex grouping
-- - Multiple grouping sets

SELECT status, COUNT(*), AVG(amount)
FROM orders
GROUP BY status;

-- Avoid DISTINCT when:
-- - Data should already be unique (fix data quality)
-- - Can use JOIN/EXISTS instead
-- - Primary key is included (redundant)

-- BAD:
SELECT DISTINCT id, name FROM users;  -- id is PK!

-- GOOD:
SELECT id, name FROM users;  -- Already unique
```

**Memory Efficiency Winner:**

In modern databases: **TIE** 
- Same algorithm (hash aggregate)
- Same memory usage
- Choose based on readability and whether you need aggregates

**Exception:** Use **EXISTS** when checking for existence rather than materializing values - much more memory efficient.

---

### **Q18: The Complete Execution Puzzle**

**Question:**
```sql
WITH ranked_employees AS (
    SELECT *, 
           ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as rn
    FROM employees
)
SELECT r.department, r.name, r.salary, d.budget
FROM ranked_employees r
INNER JOIN departments d ON r.department = d.dept_name
WHERE r.rn <= 3
  AND d.budget > 100000
HAVING AVG(r.salary) > 60000
ORDER BY r.department, r.rn;
```

**Answer: ERROR - HAVING without GROUP BY!**

**The Problem:**
```sql
HAVING AVG(r.salary) > 60000
-- HAVING requires GROUP BY
-- This query has no GROUP BY clause
```

**Complete Execution Order Explanation:**

**Correct Execution Order:**
1. **WITH (CTE)** - Define ranked_employees
2. **FROM** - ranked_employees r JOIN departments d
3. **ON** - r.department = d.dept_name
4. **WHERE** - r.rn <= 3 AND d.budget > 100000
5. **GROUP BY** - (MISSING! Causes error)
6. **HAVING** - AVG(r.salary) > 60000 (ERROR: no GROUP BY)
7. **SELECT** - Choose columns
8. **Window Functions** - (Already executed in CTE)
9. **DISTINCT** - (Not applicable)
10. **ORDER BY** - r.department, r.rn
11. **LIMIT/OFFSET** - (Not applicable)

**Step-by-Step Execution:**

**Step 1: CTE Evaluation**
```sql
-- CTE: ranked_employees
SELECT *, ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as rn
FROM employees;

-- Result:
-- Dept    Name    Salary   rn
-- IT      Alice   100000   1
-- IT      Bob     90000    2
-- IT      Carol   85000    3
-- Sales   Dave    95000    1
-- Sales   Eve     88000    2
-- Sales   Frank   82000    3
```

**Step 2: FROM Clause**
```sql
FROM ranked_employees r
```
Result: All rows from CTE

**Step 3: JOIN**
```sql
INNER JOIN departments d ON r.department = d.dept_name
```
Result: Only departments that exist in both tables

**Step 4: WHERE Clause**
```sql
WHERE r.rn <= 3 AND d.budget > 100000
```
Filters to: top 3 per department, only well-funded departments

**Step 5: GROUP BY**
```sql
-- MISSING! This is the problem
-- HAVING requires GROUP BY to exist
```

**Step 6: HAVING**
```sql
HAVING AVG(r.salary) > 60000
-- ERROR: Cannot use HAVING without GROUP BY
```

**Corrected Version (Option 1: Add GROUP BY):**

```sql
WITH ranked_employees AS (
    SELECT *, 
           ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as rn
    FROM employees
)
SELECT r.department, d.budget, AVG(r.salary) as avg_salary, COUNT(*) as emp_count
FROM ranked_employees r
INNER JOIN departments d ON r.department = d.dept_name
WHERE r.rn <= 3
  AND d.budget > 100000
GROUP BY r.department, d.budget
HAVING AVG(r.salary) > 60000
ORDER BY r.department;
```

**Execution:**
1. CTE: Rank employees within departments
2. JOIN: Match with departments
3. WHERE: Filter to top 3 in well-funded departments
4. GROUP BY: Group by department
5. HAVING: Keep only departments with avg salary > 60000
6. SELECT: Show department, budget, averages
7. ORDER BY: Sort by department

**Corrected Version (Option 2: Remove HAVING, use WHERE in outer query):**

```sql
WITH ranked_employees AS (
    SELECT *, 
           ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as rn
    FROM employees
),
dept_summary AS (
    SELECT 
        r.department, 
        r.name,
        r.salary,
        d.budget,
        r.rn,
        AVG(r.salary) OVER (PARTITION BY r.department) as dept_avg
    FROM ranked_employees r
    INNER JOIN departments d ON r.department = d.dept_name
    WHERE r.rn <= 3 AND d.budget > 100000
)
SELECT department, name, salary, budget, dept_avg
FROM dept_summary
WHERE dept_avg > 60000
ORDER BY department, rn;
```

**Complete Execution Flow Diagram:**

```
Original Query (WRONG):
CTE Definition
    ↓
FROM ranked_employees
    ↓
JOIN departments
    ↓
WHERE (filter rows)
    ↓
GROUP BY (MISSING!)
    ↓
HAVING (ERROR! needs GROUP BY)
    ↓
SELECT
    ↓
ORDER BY

Corrected Query (Option 1):
CTE Definition
    ↓
FROM ranked_employees
    ↓
JOIN departments
    ↓
WHERE r.rn <= 3 AND d.budget > 100000
    ↓
GROUP BY r.department, d.budget
    ↓
HAVING AVG(r.salary) > 60000
    ↓
SELECT department, budget, AVG(salary)
    ↓
ORDER BY department
```

**Key Lessons:**

1. **HAVING requires GROUP BY** (except for treating entire table as one group)
2. **Window functions execute in CTE SELECT phase**, before main query
3. **Execution order matters**: WHERE → GROUP BY → HAVING → SELECT → ORDER BY
4. **You can't reference SELECT aliases in HAVING** (in most databases)
5. **CTEs execute first**, their window functions are already computed

**Real-World Corrected Query:**

```sql
-- Find departments where top 3 earners average > 60k,
-- and department budget > 100k

WITH ranked_employees AS (
    SELECT 
        department,
        name,
        salary,
        ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as rn
    FROM employees
)
SELECT 
    r.department,
    d.budget,
    COUNT(*) as top_earner_count,
    AVG(r.salary) as avg_top_salary,
    MIN(r.salary) as min_top_salary,
    MAX(r.salary) as max_top_salary
FROM ranked_employees r
INNER JOIN departments d ON r.department = d.dept_name
WHERE r.rn <= 3
  AND d.budget > 100000
GROUP BY r.department, d.budget
HAVING AVG(r.salary) > 60000
ORDER BY avg_top_salary DESC;
```

---

### **Q19: Memory-Optimized Ranking**

**Question:**
```sql
-- Version A
SELECT * FROM (
    SELECT *, DENSE_RANK() OVER (ORDER BY salary DESC) as rnk
    FROM employees
) WHERE rnk <= 5;

-- Version B  
SELECT TOP 5 salary FROM employees ORDER BY salary DESC;
```

**Answer: Version B is MUCH more memory-efficient!**

**Memory Analysis:**

**Version A: DENSE_RANK() over 10M rows**

```sql
SELECT * FROM (
    SELECT *, DENSE_RANK() OVER (ORDER BY salary DESC) as rnk
    FROM employees  -- 10M rows
) WHERE rnk <= 5;
```

**Execution:**
1. **Scan entire table** (10M rows)
2. **Sort by salary DESC** (requires sort buffer)
   - Memory: 10M rows × row size
   - If row = 200 bytes: 2GB sort buffer needed!
   - If doesn't fit: spills to tempdb
3. **Compute DENSE_RANK** for ALL 10M rows
4. **Filter** to rnk <= 5 (finally get 5 rows)
5. **Return** ~50-100 rows (employees with top 5 salaries)

**Memory Used:**
- Sort buffer: 2GB (or tempdb spill)
- Intermediate result: 10M rows with rank
- Final result: Small

**Version B: TOP 5 with ORDER BY**

```sql
SELECT TOP 5 salary 
FROM employees 
ORDER BY salary DESC;
```

**Execution:**
1. **Top-N sort algorithm**
   - Maintains heap of top 5 salaries only
   - Memory: 5 rows × row size
   - If row = 200 bytes: **1KB memory!**
2. **Scan table**, updating heap as better salaries found
3. **Return** 5 rows

**Memory Used:**
- Top-N heap: 1KB
- No full sort needed
- No intermediate materialization

**Memory Comparison:**

| Aspect | Version A (DENSE_RANK) | Version B (TOP 5) |
|--------|------------------------|-------------------|
| Sort buffer | 2GB | 1KB |
| Tempdb spill | Possible | No |
| Intermediate results | 10M rows | None |
| Memory grant | 2GB | Minimal |
| Execution time | ~10 seconds | ~0.5 seconds |

**The Catch with Version B:**

**1. Only returns salary values, not full rows:**
```sql
-- Version B: Only salary column
SELECT TOP 5 salary FROM employees ORDER BY salary DESC;
-- Result: 100000, 95000, 90000, 85000, 80000

-- If you need employee names, departments, etc:
SELECT TOP 5 * FROM employees ORDER BY salary DESC;
-- Still efficient! Top-N algorithm works with full rows
```

**2. Duplicate salaries handling:**
```sql
-- Data: 10 employees at 100k, 5 at 95k, 3 at 90k

-- Version B: TOP 5
SELECT TOP 5 salary FROM employees ORDER BY salary DESC;
-- Result: 100000, 100000, 100000, 100000, 100000
-- Only 1 distinct salary value (not what you might want!)

-- Version A: DENSE_RANK <= 5  
SELECT DISTINCT salary FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) as rnk
    FROM employees
) WHERE rnk <= 5;
-- Result: 100000, 95000, 90000, 85000, 80000
-- 5 distinct salary levels (probably what you want!)
```

**3. TOP with DISTINCT:**
```sql
-- Get top 5 DISTINCT salaries efficiently:
SELECT DISTINCT TOP 5 salary
FROM employees
ORDER BY salary DESC;

-- Execution:
-- 1. Scan table
-- 2. Remove duplicates (hash aggregate)
-- 3. Top-N sort on distinct values
-- Memory: Much less than DENSE_RANK approach
```

**Execution Plan Comparison:**

**Version A:**
```
Compute Scalar (add DENSE_RANK)
    ↓
Segment (partition data)
    ↓
Sort (ORDER BY salary DESC) ← 2GB memory grant!
    ↓
Table Scan (10M rows)
```

**Version B:**
```
Top (5 rows) ← Top-N algorithm, minimal memory
    ↓
Table Scan (10M rows, but with early exit optimization)
```

**Real-World Example:**

```sql
-- Find top 5 highest salaries with employee details

-- INEFFICIENT:
SELECT * FROM (
    SELECT *, DENSE_RANK() OVER (ORDER BY salary DESC) as rnk
    FROM employees
) WHERE rnk <= 5;
-- Memory: 2GB+ for 10M rows

-- EFFICIENT:
SELECT TOP 5 *
FROM employees
ORDER BY salary DESC;
-- Memory: Minimal (top-N heap)

-- If you need "top 5 salary LEVELS" (handle ties):
WITH top_salaries AS (
    SELECT DISTINCT TOP 5 salary
    FROM employees
    ORDER BY salary DESC
)
SELECT e.*
FROM employees e
WHERE e.salary IN (SELECT salary FROM top_salaries)
ORDER BY e.salary DESC, e.name;
-- Memory: Much better than full DENSE_RANK
```

**When to Use Each:**

| Use Case | Best Approach |
|----------|---------------|
| Top N rows | `SELECT TOP N` or `LIMIT N` |
| Top N distinct values | `SELECT DISTINCT TOP N` |
| Top N per group | DENSE_RANK with PARTITION BY |
| Rank all rows | ROW_NUMBER/RANK/DENSE_RANK |
| Top N with complex tie-breaking | Window functions |

**Additional Memory Optimization:**

```sql
-- If you have index on salary:
CREATE INDEX ix_salary ON employees(salary DESC);

-- Version B becomes even better:
SELECT TOP 5 salary FROM employees ORDER BY salary DESC;
-- Execution: Index scan (already sorted!)
-- Memory: Almost none
-- No sort needed at all!

-- Version A still needs to scan entire index:
SELECT * FROM (
    SELECT *, DENSE_RANK() OVER (ORDER BY salary DESC) as rnk
    FROM employees
) WHERE rnk <= 5;
-- Still computes rank for all rows
```

**Summary:**

**Version B (TOP 5) is VASTLY more memory-efficient:**
- Uses Top-N algorithm: constant memory
- No full sort required
- No intermediate materialization
- 2000x less memory (1KB vs 2GB)

**The catch:**
- Handles duplicates differently (returns rows, not distinct values)
- If you need "top N distinct values," use `DISTINCT TOP N`
- If you need "top N per group," must use window functions

**Best Practice:**
- Use `TOP N` / `LIMIT N` whenever possible
- Only use window functions when you actually need ranking/partitioning
- Add indexes to avoid sorts altogether

---

### **Q20: Recursive CTE Memory Explosion**

**Question:**
```sql
WITH RECURSIVE org_chart AS (
    SELECT id, name, manager_id, 1 as level
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.id, e.name, e.manager_id, o.level + 1
    FROM employees e
    JOIN org_chart o ON e.manager_id = o.id
)
SELECT * FROM org_chart;
```

**Answer: Multiple memory explosion risks!**

**Memory Issues:**

**1. Depth Explosion (Deep Hierarchies)**

```sql
-- If org chart is 100 levels deep:
-- Level 1: 1 CEO
-- Level 2: 10 direct reports
-- Level 3: 100 employees
-- ...
-- Level 10: Millions of rows in intermediate results!

-- Memory grows with each iteration:
-- Iteration 1: 1 row in working table
-- Iteration 2: 10 rows
-- Iteration 3: 100 rows
-- Iteration 10: 10,000,000 rows in memory!
```

**2. Circular Reference (Infinite Loop)**

```sql
-- Data error:
-- Alice (id=1) reports to Bob (id=2)
-- Bob (id=2) reports to Alice (id=1)

-- Execution:
-- Iteration 1: Alice (level 1)
-- Iteration 2: Bob (level 2)
-- Iteration 3: Alice (level 3) ← Already processed!
-- Iteration 4: Bob (level 4)
-- Iteration 5: Alice (level 5)
-- ...
-- INFINITE LOOP! Memory fills until crash
```

**3. Cartesian Product (Multiple Paths)**

```sql
-- Employee with multiple managers (bad data):
-- Carol reports to both Alice AND Bob

-- Execution:
-- Iteration 1: CEO (1 row)
-- Iteration 2: Alice, Bob (2 rows)
-- Iteration 3: Carol appears twice! (1 for each manager)
-- Memory: Exponential growth
```

**Preventing Memory Exhaustion:**

**Solution 1: MAXRECURSION Limit (SQL Server)**

```sql
-- Limit recursion depth:
WITH org_chart AS (
    SELECT id, name, manager_id, 1 as level
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.id, e.name, e.manager_id, o.level + 1
    FROM employees e
    JOIN org_chart o ON e.manager_id = o.id
    WHERE o.level < 100  -- Prevent runaway recursion
)
SELECT * FROM org_chart
OPTION (MAXRECURSION 100);  -- SQL Server specific

-- If recursion exceeds limit: error thrown, query stops
-- Better than memory exhaustion!
```

**Solution 2: Circular Reference Detection**

```sql
-- Track visited nodes:
WITH RECURSIVE org_chart AS (
    SELECT 
        id, 
        name, 
        manager_id, 
        1 as level,
        CAST(id AS VARCHAR(1000)) as path,  -- Track path
        ARRAY[id] as visited  -- PostgreSQL array
    FROM employees 
    WHERE manager_id IS NULL
    
    UNION ALL
    
    SELECT 
        e.id, 
        e.name, 
        e.manager_id, 
        o.level + 1,
        o.path || '/' || CAST(e.id AS VARCHAR),
        o.visited || e.id
    FROM employees e
    JOIN org_chart o ON e.manager_id = o.id
    WHERE NOT (e.id = ANY(o.visited))  -- Prevent cycles!
      AND o.level < 100  -- Safety limit
)
SELECT * FROM org_chart;
```

**Solution 3: Materialized Approach (Better for Large Hierarchies)**

```sql
-- Instead of recursive CTE, use iterative approach with temp table:

CREATE TABLE #org_hierarchy (
    id INT,
    name VARCHAR(100),
    manager_id INT,
    level INT,
    PRIMARY KEY (id)
);

-- Seed with top-level:
INSERT INTO #org_hierarchy
SELECT id, name, manager_id, 1
FROM employees
WHERE manager_id IS NULL;

-- Iterate level by level:
DECLARE @level INT = 1;
DECLARE @rows_inserted INT = 1;

WHILE @rows_inserted > 0 AND @level < 100
BEGIN
    INSERT INTO #org_hierarchy (id, name, manager_id, level)
    SELECT e.id, e.name, e.manager_id, @level + 1
    FROM employees e
    WHERE e.manager_id IN (SELECT id FROM #org_hierarchy WHERE level = @level)
      AND NOT EXISTS (SELECT 1 FROM #org_hierarchy h WHERE h.id = e.id);
    
    SET @rows_inserted = @@ROWCOUNT;
    SET @level = @level + 1;
END

SELECT * FROM #org_hierarchy;

-- Benefits:
-- 1. Control over each iteration
-- 2. Can monitor memory usage
-- 3. Can detect circular references
-- 4. Can add indexes between iterations
```

**Memory Usage Patterns:**

**Normal Hierarchy (balanced tree):**
```
Level 1: 1 row (CEO)
Level 2: 10 rows (10 MB memory)
Level 3: 100 rows (100 MB)
Level 4: 1,000 rows (1 GB)
Level 5: 10,000 rows (10 GB)

-- Manageable for typical corporate hierarchies (3-7 levels)
```

**Problematic Hierarchy (deep or circular):**
```
Level 1: 1 row
Level 2: 1 row
...
Level 100: 1 row
Level 101: 1 row (circular!)
Level 102: 1 row (circular!)
...
CRASH at Level 500 when memory exhausted
```

**Real-World Example with Safety:**

```sql
-- Safe organization chart query:
WITH RECURSIVE org_chart AS (
    -- Anchor: Top-level employees
    SELECT 
        id,
        name,
        manager_id,
        1 as level,
        name as path,
        ARRAY[id] as path_ids  -- Circular detection
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive: Add next level
    SELECT 
        e.id,
        e.name,
        e.manager_id,
        o.level + 1,
        o.path || ' > ' || e.name,
        o.path_ids || e.id
    FROM employees e
    INNER JOIN org_chart o ON e.manager_id = o.id
    WHERE o.level < 20  -- Hard limit on depth
      AND NOT (e.id = ANY(o.path_ids))  -- Prevent circular references
)
SELECT 
    id,
    name,
    level,
    path,
    manager_id
FROM org_chart
ORDER BY path;

-- Memory: Bounded by depth limit (level < 20)
-- Safety: Circular reference detection
-- Performance: Early termination on cycles
```

**Monitoring Recursive CTE Execution:**

```sql
-- SQL Server - monitor tempdb usage:
SELECT 
    session_id,
    database_id,
    internal_objects_alloc_page_count,
    internal_objects_dealloc_page_count
FROM sys.dm_db_task_space_usage
WHERE session_id = @@SPID;

-- PostgreSQL - use EXPLAIN ANALYZE:
EXPLAIN ANALYZE
WITH RECURSIVE org_chart AS (...)
SELECT * FROM org_chart;
```

**Best Practices:**

1. **Always set MAXRECURSION** (SQL Server) or depth limit
2. **Detect circular references** using path tracking
3. **Use materialized approach** for very large hierarchies
4. **Monitor memory** during development
5. **Add early termination** conditions
6. **Test with production data** volumes
7. **Consider non-recursive alternatives** (nested sets, adjacency list with multiple queries)

**Alternative: Adjacency List with Multiple Queries**

```sql
-- Instead of one recursive CTE, query level by level:

-- Level 1:
SELECT id, name, 1 as level FROM employees WHERE manager_id IS NULL;

-- Level 2:
SELECT e.id, e.name, 2 as level
FROM employees e
WHERE manager_id IN (SELECT id FROM employees WHERE manager_id IS NULL);

-- Level 3:
-- etc...

-- More memory-efficient for very deep hierarchies
-- Can pause/resume between levels
-- No risk of infinite recursion
```

**Summary:**

Recursive CTEs can cause memory explosion through:
1. **Deep hierarchies** (many levels)
2. **Circular references** (infinite loops)
3. **Multiple paths** (Cartesian products)

**Prevent with:**
- Depth limits (MAXRECURSION)
- Circular reference detection
- Path tracking
- Materialized iterative approach
- Monitoring and testing

Always assume your hierarchical data may have errors or unexpected depth!