# 2. Question: Explain This PySpark Code Without Running It

### Interview Question

Give the candidate the following code:

```python
df = spark.read.parquet("/data/sales")

df1 = df.filter("year = 2026")
df2 = df1.select("customer_id", "amount")

df3 = df2.groupBy("customer_id").sum("amount")

df4 = df3.filter("sum(amount) > 10000")

df4.count()
```

### Ask the Candidate

> **"Tell me exactly when Spark reads the data, when computation happens, and where the shuffle occurs."**

---

## Expected Answer

### 1. Spark Transformations Are Lazy

The candidate should understand that Spark does **not immediately execute** most DataFrame operations.

These are **transformations**:

```python
filter()
select()
groupBy()
filter()
```

They build a **logical plan**, which Spark later optimizes into a physical execution plan.

**Nothing is actually executed until an action is called.**

In this example:

```python
df4.count()
```

is the **action** that triggers execution.

---

## 2. Execution Flow

The candidate should be able to explain the execution approximately as:

```text
                 Parquet Files
                      │
                      ▼
             Filter: year = 2026
                      │
                      ▼
             Projection / Select
          customer_id, amount
                      │
                      ▼
                  SHUFFLE
                      │
                      ▼
          Group By customer_id
                      │
                      ▼
        SUM(amount) per customer
                      │
                      ▼
       Filter: sum(amount) > 10000
                      │
                      ▼
                    Count
```

### 🔥 Key Point: Shuffle

The important operation is:

```python
groupBy("customer_id")
```

A `groupBy` normally requires Spark to bring records with the same `customer_id` together.

That requires data to move between partitions, which is a **shuffle**.

> **`groupBy()` → data redistribution across partitions → shuffle**

The candidate should be able to explain **why** the shuffle occurs, rather than simply saying:

> "groupBy causes a shuffle."

---

## 3. Predicate Pushdown and Projection Pushdown

A strong candidate should also mention **Parquet optimization**.

The original code reads:

```python
df = spark.read.parquet("/data/sales")
```

but the downstream logic only needs:

```text
year
customer_id
amount
```

Spark's optimizer and Parquet reader can potentially use:

* **Predicate pushdown** → avoid reading rows that don't satisfy `year = 2026`
* **Projection pushdown** → avoid reading unnecessary columns

So Spark may avoid scanning unnecessary data from the Parquet files.

### Strong Candidate Should Say

> "Because the source is Parquet, Spark can push applicable filters and column projections closer to the data source, reducing the amount of data that needs to be read."

---

## 4. What Actually Triggers Execution?

This line:

```python
df4.count()
```

is an **action**.

It causes Spark to construct/optimize the plan and execute the required stages.

Therefore:

```python
df.filter(...)
df.select(...)
df.groupBy(...)
df.filter(...)
```

do **not** independently execute the data processing.

They construct the computation that Spark eventually executes when an action is invoked.

---

# Follow-Up Question: Expose Memorization

Ask the candidate:

> **"If I remove `count()` and replace the last line with `df4.cache()`, has the computation happened?"**

### Strong Answer

**No.**

Calling:

```python
df4.cache()
```

does **not** immediately execute and populate the cache.

It marks the DataFrame for caching.

An **action** is still required to materialize the cached data.

For example:

```python
df4.cache()

df4.count()
```

or:

```python
df4.cache()

df4.show()
```

or:

```python
df4.cache()

df4.write.parquet("/output")
```

The first action causes the computation to execute and the cache to be populated.

---

## 🎯 What This Question Tests

| Concept                        | What You Are Testing                                        |
| ------------------------------ | ----------------------------------------------------------- |
| **Lazy evaluation**            | Does the candidate understand when Spark actually executes? |
| **Transformations vs Actions** | Can they distinguish `filter()` from `count()`?             |
| **Shuffle**                    | Do they understand data movement between partitions?        |
| **groupBy**                    | Do they understand why aggregation can require a shuffle?   |
| **Catalyst Optimizer**         | Do they understand plan optimization?                       |
| **Predicate pushdown**         | Do they understand source-level filtering?                  |
| **Projection pushdown**        | Do they understand column pruning?                          |
| **Caching**                    | Do they know that `cache()` itself is lazy?                 |
| **Execution planning**         | Can they reason about Spark without running the code?       |

---

## 🚨 Red Flags

Be cautious if the candidate says:

* ❌ "`filter()` immediately reads the data."
* ❌ "`select()` immediately executes."
* ❌ "`cache()` immediately caches the data."
* ❌ "`groupBy()` always means the entire dataset is shuffled."
* ❌ "Spark executes each line one by one."
* ❌ "The shuffle happens because `count()` is an action."
* ❌ "Parquet always reads the complete file."
* ❌ They can define "lazy evaluation" but cannot explain **where the shuffle occurs and why**.

---

## ⭐ Excellent Candidate

A very strong candidate should naturally discuss:

```text
Lazy evaluation
       ↓
Catalyst optimization
       ↓
Predicate pushdown
       ↓
Projection / column pruning
       ↓
Partitioning
       ↓
Shuffle
       ↓
Aggregation
       ↓
Action
       ↓
Execution
```

They should be able to explain the **reasoning behind the execution**, not just memorize the terms.
