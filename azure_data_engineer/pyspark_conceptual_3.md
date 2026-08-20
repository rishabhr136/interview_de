# 16. Databricks Delta — Internals & Time Travel

## Interview Question

Ask the candidate:

> **"Why would you use Delta instead of plain Parquet?"**

---

# 🎯 What to Look For

A strong candidate should mention several of these:

* ✅ **ACID transactions**
* ✅ **Transaction log**
* ✅ **Schema enforcement**
* ✅ **Schema evolution**
* ✅ **Time travel**
* ✅ **MERGE**
* ✅ **DELETE / UPDATE**
* ✅ **Reliable concurrent writes**
* ✅ **Operational capabilities**

But don't stop at feature memorization.

The real test is whether the candidate understands **what Delta adds on top of Parquet**.

---

# 🔥 Follow-Up: What Physically Happens During an UPDATE?

Ask:

> **"What physically happens when I UPDATE a Delta table?"**

### ❌ Weak Candidate

> "It changes the Parquet file."

This is **incomplete**.

The candidate should understand that Delta does not simply modify the existing Parquet bytes in place.

---

## ✅ Strong Candidate Answer

A stronger candidate should explain:

> **"Delta maintains transaction-log metadata and writes new or rewritten data files as part of the transaction rather than simply modifying Parquet bytes in place."**

Conceptually:

```text id="v3j9qk"
UPDATE Delta Table
       │
       ▼
Identify affected data files
       │
       ▼
Rewrite affected records
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

The key distinction is:

```text id="9c4w8p"
Parquet files
      +
Delta transaction log
      +
Transactional commits
      ↓
   Delta Table
```

---

# 🧠 Why Is the Transaction Log Important?

Ask:

> **"If Delta still stores data as Parquet, what does the transaction log actually give you?"**

A strong candidate should connect the transaction log with:

* Table versions
* Atomic commits
* Consistent table state
* Tracking data-file additions/removals
* Concurrent operations
* Recovery and transactional behavior

The candidate should understand that the transaction log is **not simply an audit log**.

It is fundamental to determining the table's state at a particular version.

---

# 🔎 Follow-Up: Previous Versions

Ask:

> **"How would you find previous versions of a Delta table?"**

### Expected Answer

The candidate should know about:

* **Delta table history**
* **Time travel**
* **Version numbers**
* **Timestamps**

For example:

```sql id="s5p4wd"
DESCRIBE HISTORY my_table;
```

They should understand that Delta can expose the table's historical versions and allow supported time-travel queries against previous states.

---

# 🔥 Make It Harder

Ask:

> **"If I update a row today, does Delta physically keep the old Parquet data?"**

A strong candidate should understand that old data files may remain as part of the table's historical state until they are eventually cleaned up according to the table's retention/maintenance process.

This is important for understanding why **time travel is possible**.

Conceptually:

```text id="n2k7vy"
Version 1
   │
   ├── old-file-1.parquet
   └── old-file-2.parquet

UPDATE
   │
   ▼

Version 2
   │
   ├── old-file-1.parquet
   └── new-file-3.parquet
```

The transaction log determines which files belong to each table version.

---

# 🚨 Red Flags

Be cautious if the candidate says:

* ❌ "Delta is just a different Parquet format."
* ❌ "UPDATE modifies the Parquet file in place."
* ❌ "The transaction log is only for auditing."
* ❌ "Time travel means Delta creates a complete backup for every version."
* ❌ They cannot explain why the transaction log is required.
* ❌ They know `DESCRIBE HISTORY` but cannot explain what it represents.
* ❌ They cannot connect old data files with time travel.

---

# ⭐ Excellent Candidate Answer

A very strong candidate should be able to say:

> **"Delta uses Parquet for the underlying data, but adds a transaction log that tracks table state and enables transactional commits. An UPDATE doesn't simply modify Parquet bytes in place. Delta identifies the affected files, writes new or rewritten data files, and commits the change through the transaction log. Because table versions are represented through the transaction log and historical files can remain available, Delta can provide time travel. I can inspect the table history using `DESCRIBE HISTORY` and query previous versions using Delta's time-travel capabilities."**

---

# 🎯 What This Question Tests

| Concept               | What You're Testing                                         |
| --------------------- | ----------------------------------------------------------- |
| **Delta vs Parquet**  | Do they understand the actual value Delta adds?             |
| **Transaction Log**   | Do they understand how table state is tracked?              |
| **ACID**              | Do they understand transactional behavior?                  |
| **UPDATE**            | Do they understand data-file rewriting?                     |
| **Time Travel**       | Do they understand versioned table access?                  |
| **Concurrency**       | Do they understand reliable concurrent writes?              |
| **File Lifecycle**    | Do they understand why historical versions remain possible? |
| **MERGE**             | Do they understand transactional upserts?                   |
| **Schema Management** | Do they understand enforcement vs evolution?                |

> **Interviewer's goal:** Don't accept a candidate who can merely list **ACID, time travel, MERGE, and schema evolution**. Ask **what physically changes in the data files and transaction log during an UPDATE**, then connect that explanation to **table history and time travel**. This separates memorized Delta terminology from genuine understanding.





# 28. Driver vs Executor Scenario

## Interview Question

Give the candidate:

```python
df.groupBy("customer_id").count().collect()
```

Then ask:

> **"Where does the `groupBy` execute?"**

### Expected Answer

**On the executors.**

The aggregation is distributed across the Spark cluster.

Conceptually:

```text
Input Data
    │
    ▼
Executors
    │
    ▼
Partial Aggregation
    │
    ▼
Shuffle
    │
    ▼
Final Aggregation
```

The `groupBy("customer_id").count()` computation happens on the executors, not on the driver.

---

# Follow-Up: Where Does `collect()` Execute?

Ask:

> **"Where does `collect()` execute?"**

### Expected Answer

The distributed computation still runs on the executors, but `collect()` **brings the final result back to the driver process**.

Conceptually:

```text
Executors
  │
  │ Final result partitions
  ▼
Driver
```

The driver gathers all result partitions into the driver's memory and returns them to the application.

---

# Make It Difficult

Ask:

> **"What if the grouped result contains 5 million rows?"**

### Strong Answer

The danger is **driver memory pressure or an Out-Of-Memory (OOM) error**.

Even though the aggregation itself was successfully distributed across the executors, `collect()` attempts to materialize the entire result on the driver.

```text
Executors
   │
   │ 5 million rows
   ▼
Driver Memory
   │
   ▼
Potential OOM
```

The bottleneck has moved from the executors to the driver.

---

# Important Distinction

A common misconception is:

> ❌ "`collect()` is always bad."

That's not correct.

The problem is not `collect()` itself.

The problem is **collecting a result that is too large for the driver's memory**.

---

# Follow-Up

Ask:

> **"What if the grouped result is only 100 rows?"**

### Expected Answer

Then `collect()` may be perfectly reasonable.

For example, if the result is being used to:

* Build a small lookup dictionary
* Perform a quick validation
* Return a compact summary
* Drive a small control-flow decision

then collecting 100 rows is often completely acceptable.

```text
Result Size
    │
    ├── 100 rows
    │      ↓
    │   collect() is reasonable
    │
    └── 5 million rows
           ↓
       Risky for driver
```

---

# The Correct Principle

The correct answer depends on **result size**, not on a blanket rule.

| Result Size       | `collect()`                            |
| ----------------- | -------------------------------------- |
| A few rows        | Usually fine                           |
| Hundreds of rows  | Often fine                             |
| Thousands of rows | Depends on row width and driver memory |
| Millions of rows  | Usually dangerous                      |
| Huge datasets     | Avoid collecting to the driver         |

The number of rows is only one factor.

A candidate may also mention that **row width** matters.

For example:

```text
100,000 narrow rows
```

may consume less memory than:

```text
10,000 extremely wide rows
```

---

# Advanced Follow-Up

Ask:

> **"What would you use instead of `collect()` if you only wanted to inspect the result?"**

A strong candidate might suggest:

```python
df.show()
```

or:

```python
df.limit(100).collect()
```

depending on the requirement.

If the goal is to write the result somewhere, they should use a distributed write rather than collecting it to the driver.

---

# Another Follow-Up

Ask:

> **"Does `collect()` cause the `groupBy` to run on the driver?"**

### Expected Answer

**No.**

This is an important distinction.

`collect()` is an **action** that triggers the distributed computation.

The `groupBy` and aggregation still execute on the executors.

Only the **final result** is transferred to the driver.

```text
Driver
  │
  │ Trigger action
  ▼
Executors
  │
  │ Execute groupBy/count
  ▼
Driver
  │
  │ Receive final result
```

---

# Red Flags

Be cautious if the candidate says:

* ❌ "`groupBy` runs on the driver."
* ❌ "`collect()` executes the aggregation on the driver."
* ❌ "`collect()` is always bad."
* ❌ "Spark automatically spills the collected result to disk on the driver."
* ❌ They cannot distinguish computation location from result location.
* ❌ They think executors return only metadata to the driver.
* ❌ They don't mention driver memory as the limiting factor.

---

# Excellent Candidate Answer

A very strong candidate should say:

> **"`groupBy("customer_id").count()` is executed in a distributed manner on the executors, including the shuffle and aggregation. `collect()` is the action that triggers the job and then brings all of the final result rows back to the driver. If that result contains 5 million rows, the main risk is exhausting the driver's memory, even though the distributed computation succeeded. If the result is only around 100 rows, collecting it can be perfectly reasonable. The decision depends on the size of the result and the available driver memory, not on a rule that `collect()` is always bad."**

---

# What This Question Tests

| Concept                   | What You're Testing                                    |
| ------------------------- | ------------------------------------------------------ |
| **Driver**                | Does the candidate know its role?                      |
| **Executors**             | Do they know where transformations execute?            |
| **Actions**               | Do they understand what `collect()` triggers?          |
| **Result Transfer**       | Do they know what moves to the driver?                 |
| **Driver OOM**            | Can they identify the real risk?                       |
| **Result Size**           | Can they reason about when `collect()` is appropriate? |
| **Distributed Execution** | Can they separate computation from result collection?  |

> **Interviewer's goal:** This question is deceptively simple. Many candidates know the terms "driver" and "executor" but still confuse **where the computation runs** with **where the final result is materialized**. The strongest candidates make that distinction clearly and avoid the oversimplified statement that **"`collect()` is always bad."**



