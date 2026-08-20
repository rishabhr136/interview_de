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


# 3. Databricks Delta Lake Question

## Interview Question

Ask the candidate:

> **"Why would you use Delta instead of plain Parquet?"**

### Look For

A strong candidate should mention several of the following:

* ✅ **ACID transactions**
* ✅ **Transaction log**
* ✅ **Schema enforcement**
* ✅ **Schema evolution**
* ✅ **Time travel**
* ✅ **MERGE**
* ✅ **DELETE / UPDATE**
* ✅ **Reliable concurrent writes**
* ✅ **Operational capabilities**

---

## 🔥 Follow-Up Question: What Happens During an UPDATE?

Ask:

> **"What physically happens when I UPDATE a Delta table?"**

### ❌ Weak Answer

> "It changes the Parquet file."

This answer is **incomplete**.

The candidate should understand that Delta Lake does not simply modify Parquet bytes in place.

### ✅ Strong Answer

A stronger candidate should explain that Delta maintains **transaction-log metadata** and writes new or rewritten data files as part of the transaction rather than simply modifying the existing Parquet file in place.

Conceptually:

```text
UPDATE Delta Table
       │
       ▼
Identify affected data files
       │
       ▼
Rewrite affected data
       │
       ▼
Write new Parquet files
       │
       ▼
Commit transaction
       │
       ▼
Update Delta transaction log
```

### 🎯 Key Concept

The important distinction is:

> **Delta Lake = Parquet data files + transaction log + transactional metadata**

So the candidate should be able to explain both:

1. **What happens to the data files**
2. **What happens to the transaction log**

---

## 🔎 Follow-Up Question: Previous Versions

Ask:

> **"How would you find previous versions of a Delta table?"**

### Expected Answer

The candidate should know about:

* **Delta table history**
* **Time travel**
* **Version numbers**
* **Timestamps**

For example:

```sql
DESCRIBE HISTORY my_table;
```

And they should understand that previous table versions can be queried using **Delta time travel**.

---

## 🚨 Red Flags

Be cautious if the candidate says:

* ❌ "Delta is just Parquet."
* ❌ "UPDATE directly modifies the Parquet bytes."
* ❌ "Delta doesn't need a transaction log."
* ❌ "Time travel means taking a backup every time."
* ❌ "MERGE is only a SQL join."
* ❌ They cannot explain what happens to the underlying files during an update.
* ❌ They know Delta terminology but cannot explain **why the transaction log is necessary**.

---

## ⭐ Strong Candidate Follow-Up

Ask one final question:

> **"If Delta uses Parquet files underneath, why can't I get the same ACID behavior by simply writing Parquet carefully?"**

### What You Want to Hear

The candidate should explain that **the transaction log provides the coordination and atomic commit mechanism** needed to maintain a consistent table state across data-file changes and concurrent operations.

They should be able to connect:

```text
Parquet
   +
Transaction Log
   +
Atomic Commits
   +
Schema Management
   +
Versioning
   ↓
Delta Lake
```

### 🎯 What This Question Tests

| Concept               | What You Are Testing                                                 |
| --------------------- | -------------------------------------------------------------------- |
| **Delta vs Parquet**  | Does the candidate understand the actual value of Delta?             |
| **Transaction Log**   | Do they understand how table state is tracked?                       |
| **ACID**              | Can they explain transactional guarantees?                           |
| **UPDATE**            | Do they understand file rewriting rather than in-place modification? |
| **MERGE**             | Do they understand transactional upserts?                            |
| **Time Travel**       | Do they understand version-based table access?                       |
| **Concurrency**       | Do they understand reliable concurrent writes?                       |
| **Schema Management** | Do they understand enforcement vs evolution?                         |

> **Interviewer's goal:** Don't accept a candidate who can only list Delta features. Ask **what happens physically and why**. That distinction quickly separates memorized answers from genuine Delta Lake understanding.



# 4. Delta MERGE — Excellent Interview Question

## Interview Question

Give the candidate:

```sql
MERGE INTO target t
USING source s
ON t.customer_id = s.customer_id

WHEN MATCHED THEN
    UPDATE SET *

WHEN NOT MATCHED THEN
    INSERT *
```

Then ask:

> **"What happens if the source contains two records for the same `customer_id`?"**

---

## 🎯 What You Are Testing

This is a good question because it tests whether the candidate understands **MERGE semantics**, rather than simply knowing the syntax.

A strong candidate should recognize that:

> **If multiple source rows match the same target row, the MERGE can become ambiguous and may fail, depending on the matched conditions and operation.**

For example:

### Target

| customer_id | amount |
| ----------: | -----: |
|         101 |    500 |

### Source

| customer_id | amount | updated_at |
| ----------: | -----: | ---------- |
|         101 |    700 | 2026-08-19 |
|         101 |    900 | 2026-08-20 |

Both source records match:

```text
t.customer_id = s.customer_id
```

So Spark/Delta cannot necessarily determine which source record should be used to update the single target record.

---

# 🔥 Follow-Up Question

Ask:

> **"How would you fix it?"**

The candidate should propose **deduplicating the source before the MERGE**.

A common approach is to keep the most recently updated record.

### PySpark Solution

```python
from pyspark.sql import functions as F
from pyspark.sql.window import Window

w = Window.partitionBy("customer_id").orderBy(
    F.col("updated_at").desc()
)

source = (
    source
    .withColumn("rn", F.row_number().over(w))
    .filter(F.col("rn") == 1)
    .drop("rn")
)
```

Now each `customer_id` has at most **one source record**.

Then perform the MERGE:

```sql
MERGE INTO target t
USING source s

ON t.customer_id = s.customer_id

WHEN MATCHED THEN
    UPDATE SET *

WHEN NOT MATCHED THEN
    INSERT *
```

---

## ⭐ Strong Candidate Should Explain the Logic

The candidate shouldn't just write `row_number()`.

Ask:

> **"Why did you use `row_number()` instead of `dropDuplicates()`?"**

### Strong Answer

`dropDuplicates(["customer_id"])` removes duplicates, but it does **not express which record should win**.

With:

```python
Window.partitionBy("customer_id").orderBy(
    F.col("updated_at").desc()
)
```

we explicitly define the business rule:

> **For each customer, keep the most recently updated record.**

Then:

```python
row_number() == 1
```

selects that record.

---

## 🚨 Red Flags

Be cautious if the candidate says:

* ❌ "Delta automatically picks one of the duplicate source rows."
* ❌ "MERGE will update the target twice."
* ❌ "Just use `dropDuplicates()`" without discussing which record should be retained.
* ❌ They cannot explain why multiple source matches are problematic.
* ❌ They know `row_number()` but cannot explain the `partitionBy()` and `orderBy()` logic.
* ❌ They don't ask what the **business rule** is for choosing the winning record.

---

## 🧠 Advanced Follow-Up

Ask:

> **"What if two records have the exact same `updated_at` timestamp?"**

This is an excellent discriminator.

A very strong candidate should recognize that the ordering is no longer deterministic.

They may suggest adding a **tie-breaker**:

```python
w = Window.partitionBy("customer_id").orderBy(
    F.col("updated_at").desc(),
    F.col("source_sequence").desc()
)
```

Now the selection becomes deterministic:

```text
customer_id
     │
     ▼
Partition records
     │
     ▼
Sort by updated_at DESC
     │
     ▼
Tie-break using source_sequence DESC
     │
     ▼
row_number() = 1
     │
     ▼
MERGE
```

> **Interviewer's goal:** Don't stop when the candidate says "deduplicate the source." Ask **how they decide which duplicate survives**. That reveals whether they understand the production-data problem rather than just memorizing a PySpark pattern.
