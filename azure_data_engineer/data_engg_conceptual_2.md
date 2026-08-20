# Ultimate Fundamental Data Engineering Debugging Question

## 🎯 Scenario

Give the candidate:

```text
Yesterday:
100 GB input
30 minutes

Today:
100 GB input
4 hours

Same code
```

Then ask:

> **"What could have changed?"**

---

# Expected Answer

A strong candidate should **not immediately propose a solution**.

They should first generate hypotheses about what could have changed even though the input volume and code appear the same.

Potential areas include:

```text id="j4z8sp"
Data Distribution
       ↓
Data Skew
       ↓
File Count
       ↓
File Sizes
       ↓
Source Data Quality
       ↓
Join Cardinality
       ↓
Duplicate Records
       ↓
Schema Changes
       ↓
Cluster Health
       ↓
Network / Storage
       ↓
Partition Distribution
```

The important point is:

> **100 GB of data does not necessarily mean the same amount of work.**

---

# 🔥 Follow-Up: Don't Give Me Solutions

Ask:

> **"Don't give me solutions. Give me the measurements you would collect."**

This is the **key part of the interview question**.

A strong data engineer should respond with **evidence they would collect before changing anything**.

---

# 1. Data Distribution

Ask:

> **"What would you measure about the data itself?"**

Look for:

* Record count
* Records per partition
* Distribution of important join/grouping keys
* Null rates
* Duplicate rates
* Cardinality
* Data types
* Min/max values where relevant

For example:

```text id="j8s5xc"
Yesterday:
customer_id distribution → relatively uniform

Today:
customer_id = C001 → 70% of records
```

That could indicate a skew problem even though total input remains 100 GB.

---

# 2. File Count and File Sizes

Ask:

> **"What would you check about the input files?"**

Look for:

* Total number of files
* Average file size
* Minimum file size
* Maximum file size
* Number of tiny files
* Number of unusually large files
* File format
* Compression

For example:

```text id="f2c8na"
Yesterday:

10,000 files × ~10 MB

Today:

5,000,000 files × ~20 KB
```

Both datasets could represent approximately the same logical volume, but the processing characteristics can be dramatically different.

---

# 3. Join Cardinality

Ask:

> **"What measurements would you collect around joins?"**

Look for:

* Number of rows before the join
* Number of rows after the join
* Join-key cardinality
* Match rate
* Number of unmatched records
* Duplicate keys
* Many-to-many relationships

Example:

```text id="m0b5zx"
Yesterday:

1 input row → ~1 output row

Today:

1 input row → 50 matching rows
```

The output can explode even though the input volume remains 100 GB.

---

# 4. Duplicate Records

Ask:

> **"How would you check whether today's data contains unexpected duplicates?"**

Look for:

```python id="2k7m4p"
df.groupBy("business_key") \
  .count() \
  .filter("count > 1")
```

The candidate should explain that they would compare duplicate rates between yesterday and today.

For example:

```text id="3c5v7n"
Yesterday duplicate rate → 0.1%

Today duplicate rate → 35%
```

That could explain significantly increased processing.

---

# 5. Schema Changes

Ask:

> **"What would you compare in the schemas?"**

Look for:

* Column count
* Column names
* Data types
* Nullability
* Nested structure
* New columns
* Removed columns
* Changed data types

For example:

```text id="x8p3qm"
Yesterday:
amount → DECIMAL

Today:
amount → STRING
```

A schema change could affect parsing, execution, joins, filtering, or downstream processing.

---

# 6. Spark Execution Metrics

This is where a strong Spark candidate should become very specific.

Ask:

> **"What would you inspect in the Spark UI?"**

Look for:

* Stage duration
* Task duration
* Number of tasks
* Task duration distribution
* Input size
* Output size
* Shuffle read
* Shuffle write
* Spill to memory
* Spill to disk
* Executor failures
* GC time
* Scheduler delay

For example:

```text id="h5j2kw"
Yesterday:

Task 1 → 25 sec
Task 2 → 27 sec
Task 3 → 24 sec
Task 4 → 26 sec

Today:

Task 1 → 20 sec
Task 2 → 21 sec
Task 3 → 22 sec
Task 4 → 3 hours
```

That is strong evidence of a partition/data-skew problem.

---

# 7. Partition Distribution

Ask:

> **"What would you measure about the partitions?"**

Look for:

* Number of partitions
* Records per partition
* Bytes per partition
* Distribution of partition sizes
* Whether a few partitions are disproportionately large

Conceptually:

```text id="w8q6dy"
Partition 1 → 2 GB
Partition 2 → 2 GB
Partition 3 → 2 GB
Partition 4 → 85 GB   ← suspicious
```

This is much more useful than simply saying:

> "There might be skew."

---

# 8. Cluster Health

Ask:

> **"What would you measure about the cluster?"**

Look for:

* Executor count
* Executor availability
* CPU utilization
* Memory utilization
* GC time
* Executor failures
* Executor lost events
* Disk usage
* Spill
* Autoscaling behavior
* Driver health

The candidate should determine whether the problem is actually **data-related** or **infrastructure-related**.

---

# 9. Network and Storage

Ask:

> **"What infrastructure-level measurements would you check?"**

Look for:

* Storage read latency
* Storage throughput
* Network throughput
* Network latency
* API throttling
* Storage errors
* Retries
* Object-store request behavior
* Source-system latency

For example:

```text id="q8s2pl"
Yesterday:
Storage throughput → 800 MB/s

Today:
Storage throughput → 100 MB/s
```

The same Spark code and same logical input size could therefore take much longer.

---

# 🧠 The Important Debugging Framework

A strong candidate should effectively build an evidence chain:

```text id="u7c3qm"
                4-hour Runtime
                      │
                      ▼
              Which Stage Changed?
                      │
                      ▼
            Which Metric Changed?
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       Shuffle      Tasks       Input
       Metrics      Metrics     Metrics
          │           │           │
          ▼           ▼           ▼
        Skew?       Straggler?   Files?
```

Then continue narrowing down the root cause.

---

# 🚨 The Trap

If the candidate immediately says:

> ❌ **"Increase the number of workers."**

Push back:

> **"Why? What evidence tells you that more workers will solve the problem?"**

A strong candidate should realize that more workers won't necessarily fix:

* Data skew
* A single huge partition
* Tiny-file explosion
* Join cardinality explosion
* Storage throttling
* Network bottlenecks
* Bad source data
* Schema-related processing changes

---

# ⭐ Excellent Candidate Answer

A very strong candidate might say:

> **"I wouldn't change the cluster immediately. First I'd compare yesterday's and today's execution metrics. I'd identify which stage increased from 30 minutes to 4 hours, then compare task-duration distributions, shuffle read/write, spill, input/output records, partition sizes, and executor metrics. I'd also compare the input file count and size distribution, key cardinality and skew, duplicate rates, join match/cardinality behavior, and schema. Finally I'd check cluster, storage, and network health. Once I have evidence showing where the additional time is coming from, I'd decide on the appropriate remediation."**

---

# 🎯 What This Question Tests

| Concept                   | What You're Testing                                     |
| ------------------------- | ------------------------------------------------------- |
| **Debugging Methodology** | Do they investigate before changing things?             |
| **Spark UI**              | Can they use execution evidence?                        |
| **Data Skew**             | Can they identify uneven workloads?                     |
| **Partitioning**          | Can they reason about partition distribution?           |
| **Shuffle**               | Can they identify expensive data movement?              |
| **Join Cardinality**      | Can they detect output explosion?                       |
| **File Layout**           | Do they understand small-file problems?                 |
| **Data Quality**          | Can bad data affect performance?                        |
| **Infrastructure**        | Can they distinguish data vs cluster problems?          |
| **Root Cause Analysis**   | Can they move from hypothesis → measurement → evidence? |

> **Interviewer's goal:** The most important signal is **how the candidate thinks**. A junior candidate often jumps directly to *"increase workers."* A strong data engineer first asks **"What changed?"**, identifies measurable hypotheses, collects evidence, and only then recommends a solution.




