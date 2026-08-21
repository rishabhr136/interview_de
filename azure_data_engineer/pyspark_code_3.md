Accidental Cartesian join
df1.join(df2)
Ask

What happens here if there is no join condition?

Strong candidate should immediately identify a potential cross/cartesian join depending on the API/context and explain that it can explode the result size.

Then ask:

Suppose df1 has 10 million rows and df2 has 5 million rows. Why is this dangerous?

Potential output:

10,000,000 × 5,000,000
= 50,000,000,000,000

That's 50 trillion combinations.


Broadcast join debugging
result = large_df.join(
    small_df,
    large_df.customer_id == small_df.customer_id
)
Ask

small_df contains only 5,000 rows. How could you potentially optimize this?

Expected:

from pyspark.sql.functions import broadcast


result = large_df.join(
    broadcast(small_df),
    "customer_id"
)
Killer follow-up

Should you broadcast every DataFrame containing fewer rows?

Strong candidate should say no.

They should discuss:

size in bytes, not only row count
executor memory
broadcast threshold
serialization
whether the table is actually small enough




filter() After select() — Column Availability Trap
🎯 Interview Scenario

Give the candidate:

result = (
    df
    .select("customer_id", "amount")
    .filter(df.country == "US")
)

Then ask:

"Will this work? Why or why not?"

❌ Expected Answer

No.

The problem is that:

.select("customer_id", "amount")

creates a DataFrame containing only:

customer_id
amount

The country column is no longer available in that DataFrame.

But the next operation tries to reference:

df.country

which was a column from the original DataFrame.

The candidate should understand the distinction between the input DataFrame and the DataFrame produced after select().

🧠 What's Happening?

Original DataFrame:

df
├── customer_id
├── amount
├── country
└── ...

After:

df.select("customer_id", "amount")

the resulting DataFrame is:

result
├── customer_id
└── amount

There is no:

country

available for the subsequent filter.

❌ Incorrect Code
result = (
    df
    .select("customer_id", "amount")
    .filter(df.country == "US")
)

The logical problem is:

select()
  ↓
customer_id
amount
  ↓
filter(country)
  ↓
country is not part of this DataFrame
✅ Correct Approach

Apply the filter before selecting the columns:

result = (
    df
    .filter(df.country == "US")
    .select("customer_id", "amount")
)

Or, preferably:

from pyspark.sql import functions as F


result = (
    df
    .filter(F.col("country") == "US")
    .select("customer_id", "amount")
)
🔥 Killer Follow-Up

Ask:

"Could Spark's optimizer sometimes push the filter down even if I write select() first?"

⭐ Strong Answer

Yes, Spark's optimizer can often rearrange compatible operations in the logical/physical plan.

Catalyst may recognize that:

select(customer_id, amount)

doesn't actually need to eliminate country before the filter is evaluated.

Conceptually, Spark may optimize:

Logical plan:


Select
  ↓
Filter country = US

into something closer to:

Filter country = US
  ↓
Select customer_id, amount

This can allow the filter to be applied earlier.

⚠️ But Here's the Important Distinction

Ask:

"If Spark can optimize the plan, does that mean the original code is logically valid?"

Expected Answer

No.

This is the key interview concept.

There are two different questions:

1. Logical correctness

Does the expression reference a column that exists at that point in the DataFrame transformation?

2. Physical optimization

Can Spark's optimizer rearrange valid logical operations to execute them more efficiently?

The optimizer cannot generally rescue an expression that is already invalid because the referenced column isn't available in the logical plan.

So candidates should not say:

❌ "It will work because Catalyst will push the filter down."

🧠 Excellent Distinction

The candidate should understand:

Column Availability
        ↓
Logical Plan Validity
        ↓
Catalyst Optimization
        ↓
Physical Execution

Catalyst optimizes a valid logical plan.

It doesn't mean you can reference arbitrary columns after removing them and rely on Catalyst to fix the code.

🔎 Follow-Up: What About This?

Give:

result = (
    df
    .select("customer_id", "amount", "country")
    .filter(F.col("country") == "US")
    .select("customer_id", "amount")
)

Ask:

"Is this valid?"

Expected Answer

Yes.

country is still available when the filter is evaluated.

Then the final select() removes it:

Initial
customer_id
amount
country
    ↓
Filter country = US
    ↓
Final select
customer_id
amount

Spark may optimize the physical execution and avoid unnecessary work where possible.

🚨 Red Flags

Be cautious if the candidate says:

❌ "Yes, Spark automatically knows country exists."
❌ "Catalyst will always fix it."
❌ "Select only controls the final output."
❌ They don't understand that select() changes the DataFrame schema.
❌ They confuse logical column availability with physical optimization.
⭐ Excellent Candidate Answer

"No, not in the way it's written. After select('customer_id', 'amount'), the resulting DataFrame no longer exposes country, so the subsequent filter cannot logically reference that column. The safe expression is to filter first and then select the required columns. Spark's Catalyst optimizer can sometimes reorder a valid filter and projection in the execution plan, but that is different from allowing an invalid column reference. Optimization happens after Spark has a valid logical expression to optimize."

🎯 What This Question Tests
Concept	What You're Testing
select()	Do they understand how it changes the schema?
filter()	Do they understand column availability?
Column Resolution	Can they reason about Spark expressions?
Catalyst Optimizer	Do they understand logical vs physical optimization?
Predicate Pushdown	Do they understand filters can sometimes be moved closer to the source?
Projection Pruning	Do they understand unnecessary columns can be removed?
Debugging	Can they distinguish a coding error from an optimization?

Interviewer's Goal: The real test is whether the candidate understands that Catalyst optimization does not mean Spark can make an invalid logical expression valid. Ask them to distinguish column availability → logical plan → optimization → physical execution.



The Best Live-Debugging Question — End-to-End Spark Performance
🎯 Interview Scenario

Give the candidate:

from pyspark.sql import functions as F


result = (
    orders
    .join(customers, "customer_id")
    .filter(F.col("country") == "US")
    .groupBy("customer_id")
    .sum("amount")
    .orderBy("sum(amount)", ascending=False)
)


result.show()

Then tell them:

"This job takes 45 minutes in production. You are not allowed to simply add more executors. Debug it."

🚨 Important

Don't give them hints initially.

Let the candidate drive the investigation.

🔥 What a Strong Candidate Should Investigate

A genuinely strong candidate should start asking questions rather than immediately changing Spark configurations.

1. Input Size

Ask:

"How much data are we actually processing?"

Look for:

Total input size
Number of records
Historical vs current data
Input growth compared with previous runs
Yesterday → 200 GB
Today     → 2 TB

If the input increased 10×, the first debugging step is understanding why.

2. File Sizes and File Count

Ask:

"How is that data physically stored?"

Look for:

Number of files
Average file size
Tiny files
Extremely large files
Compression
File format

Example:

Yesterday:
20,000 files × 10 MB


Today:
5,000,000 files × 40 KB

Same logical data volume can have very different processing characteristics.

3. Number of Partitions

Ask:

"How many partitions are involved at each important stage?"

Look for:

Input partitions
Shuffle partitions
Output partitions
Partition-size distribution

The candidate should understand that:

Partition count and partition-size distribution are different problems.

4. Join Cardinality

This should be one of the first major investigations.

The code contains:

orders.join(customers, "customer_id")

Ask:

"Is customer_id unique in customers?"

If not, the join could multiply rows.

For example:

orders:
customer 101 → 100 rows


customers:
customer 101 → 20 rows

Potential output:

100 × 20 = 2,000 rows

A strong candidate should investigate table grain and join cardinality.

5. Broadcast Possibility

Ask:

"How large is the customer table?"

If customers is sufficiently small, Spark might use a broadcast join.

The candidate should inspect whether the physical plan contains something like:

BroadcastHashJoin

rather than blindly writing:

F.broadcast(customers)

They should consider:

Actual size
Serialized size
Executor memory
Growth of the dimension
Broadcast overhead
6. Data Skew

Ask:

"Could customer_id be heavily skewed?"

For example:

customer_id = 999
→ 70% of all orders

That could produce a few very large shuffle partitions.

The candidate should look for:

Normal tasks → 30 seconds
Skewed task → 25 minutes

This is much stronger evidence than simply saying:

"There might be skew."

7. Shuffle Size

The query potentially introduces shuffle during:

JOIN
  ↓
GROUP BY
  ↓
ORDER BY

The candidate should inspect:

Shuffle read
Shuffle write
Number of shuffle partitions
Shuffle partition-size distribution

Conceptually:

Orders
   ↓
JOIN
   ↓
Shuffle
   ↓
GROUP BY
   ↓
Shuffle
   ↓
ORDER BY
8. Filter Selectivity

The query contains:

.filter(F.col("country") == "US")

Ask:

"What percentage of the data is actually US?"

For example:

10% US

versus:

99% US

makes a significant difference.

A selective filter can dramatically reduce downstream processing.

9. Predicate Pushdown

Ask:

"Can the country filter be applied before expensive downstream operations?"

The candidate should consider:

Predicate pushdown
Filter placement
Partition pruning
Data skipping
Source capabilities

However, they should not blindly assume that moving the filter syntactically guarantees a particular physical plan.

10. Physical Plan

This is the killer part.

Ask:

"What does Spark actually plan to execute?"

The first thing they should run is:

result.explain("formatted")

Look for operations such as:

BroadcastHashJoin
SortMergeJoin
Exchange
HashAggregate
Sort
11. orderBy() — Hidden Global Sort

The query ends with:

.orderBy("sum(amount)", ascending=False)

Ask:

"What does this operation potentially cost?"

A global orderBy() can require significant distributed sorting.

Conceptually:

Aggregated Results
       ↓
     Shuffle
       ↓
Global Sort
       ↓
Ordered Result

The candidate should recognize that this can become expensive when the aggregated result is large.

12. Spark UI — Stages and Tasks

After examining the physical plan, the candidate should inspect the Spark UI.

Look for:

Longest stage
Number of tasks
Task duration distribution
Shuffle read/write
Input/output size
Spill
Executor failures
GC time
Straggler tasks

Example:

Stage 1 → 2 minutes
Stage 2 → 3 minutes
Stage 3 → 39 minutes  ← investigate this

Then determine why Stage 3 is slow.

13. Spill to Disk

Ask:

"Are tasks spilling to disk?"

Look for:

Memory Spill
Disk Spill

Large aggregation/sort operations can spill when data doesn't fit comfortably in memory.

The candidate should correlate spill with:

Partition size
Aggregation
Sorting
Executor memory
Data skew
14. Executor Memory / GC

Ask:

"What are the executors doing while the job is slow?"

Look for:

Executor memory pressure
GC time
Executor OOM
Executor lost events
CPU utilization
Disk usage

A job spending significant time in GC suggests a very different problem from one waiting on storage or network I/O.

15. Output Partitions

Ask:

"How much data is actually produced after the aggregation?"

The candidate should understand that:

2 TB input

does not necessarily mean:

2 TB final result

For example:

2 TB orders
     ↓
GROUP BY customer_id
     ↓
50 million customers

The final dataset may still be large enough for:

ORDER BY

to become expensive.

🔥 Killer Question

After allowing them to discuss the possible causes, ask:

"Show me the first thing you would run in PySpark before changing the code."

⭐ Best Answer
result.explain("formatted")

Then:

"What would you do next?"

Expected:

Inspect the Spark UI and correlate the physical plan with actual stage/task behavior.

🧠 The Ideal Debugging Process

A strong candidate should effectively follow:

                45-minute runtime
                       │
                       ▼
              Inspect physical plan
                       │
                       ▼
              Identify expensive stage
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Shuffle        Join        Sort
          │            │            │
          ▼            ▼            ▼
        Skew?       Cardinality?  Large result?
          │            │            │
          └────────────┼────────────┘
                       ▼
                Inspect Spark UI
                       │
                       ▼
                 Collect evidence
                       │
                       ▼
                Identify root cause
                       │
                       ▼
                  Apply fix
🚨 Red Flags

Be cautious if the candidate immediately says:

❌ "Add more executors."
❌ "Increase spark.sql.shuffle.partitions."
❌ "Use broadcast join."
❌ "Cache everything."
❌ "Increase executor memory."
❌ "Use more partitions."
❌ "Remove orderBy()."

These might eventually be valid solutions, but none should be recommended before establishing evidence.

⭐ Excellent Candidate Answer

A very strong candidate might say:

"I wouldn't change the cluster first. I'd start with result.explain("formatted") to understand the physical plan. Then I'd use the Spark UI to identify which stage is consuming most of the 45 minutes and examine task-duration distribution, shuffle read/write, spill, input/output sizes, and executor metrics. For this query I'd specifically investigate join cardinality and whether the customer table can be broadcast, key skew on customer_id, filter selectivity and whether the country predicate is being pushed down, and the cost of the final global sort. I'd compare partition-size distribution and determine whether there are straggler tasks. Only after identifying the actual bottleneck would I change the code or configuration."

🎯 What This Question Tests
Concept	What You're Testing
Physical Plan	Can they understand what Spark will actually execute?
Spark UI	Can they debug using execution evidence?
Joins	Do they understand join strategies and cardinality?
Broadcast	Do they know when broadcasting is appropriate?
Shuffle	Do they understand distributed data movement?
Data Skew	Can they identify straggler tasks?
Aggregation	Do they understand groupBy() cost?
Global Sort	Do they recognize orderBy() as potentially expensive?
Predicate Pushdown	Do they understand source-level filtering?
Partitioning	Can they reason about partition distribution?
Spill / Memory	Can they distinguish memory pressure from other bottlenecks?
Debugging Methodology	Do they collect evidence before changing configuration?

Interviewer's Goal: This is not primarily a syntax question. The real test is whether the candidate understands how Spark executes a DataFrame and can move from code → physical plan → Spark UI → measurements → root cause → solution. A candidate who immediately says "increase executors" is guessing; a strong Data Engineer first asks "Which stage is actually taking the 45 minutes, and why?"


# Window Function Without `partitionBy()` — Accidentally Global


## 🎯 Scenario


```python
from pyspark.sql import Window
from pyspark.sql import functions as F


df = spark.createDataFrame(
    [
        ("A", 1, 100),
        ("A", 2, 150),
        ("B", 1, 200),
        ("B", 2, 50)
    ],
    ["region", "seq", "sales"]
)


w = Window.orderBy("seq")


result = df.withColumn(
    "running_total",
    F.sum("sales").over(w)
)


result.show()
🎯 Ask

"This is supposed to calculate a running total of sales per region. Predict the running_total for the region='B', seq=1 row."

✅ Expected Answer

The window does not contain:

partitionBy("region")

Therefore, the window is global rather than independently calculated for each region.

The candidate should identify that the intended window should be:

w = (
    Window
    .partitionBy("region")
    .orderBy("seq")
)

This would calculate the running total independently for each region.

🔍 Depth Probe

Ask:

"What happens when there are ties in the ordering column?"

This is an advanced question.

The candidate should understand the difference between:

RANGE

and:

ROWS

and recognize that tied ordering values can affect window-frame behavior.

For deterministic business logic, use an appropriate ordering/tie-breaker and explicitly define the frame when necessary.




# Schema Inference — Numeric-Looking Strings


## 🎯 Scenario


Assume two CSV files:


### `file1.csv`


```text
id,zip_code
1,02134
file2.csv
id,zip_code
2,90210

The code is:

df = (
    spark.read
    .option("header", "true")
    .option("inferSchema", "true")
    .csv("/data/zips/*.csv")
)


df.show()
🎯 Ask

"What value do you expect for zip_code where id=1?"

✅ Expected Answer

The danger is that Spark can infer:

zip_code → integer

because the values look numeric.

That can transform:

02134

into:

2134

The job succeeds, but the data is semantically corrupted.

🧠 Root Cause

The problem is relying on:

.option("inferSchema", "true")

for a production field whose semantic meaning requires leading zeros.

Examples of fields that may look numeric but should often be strings:

ZIP/postal codes
Phone numbers
Product codes
Customer codes
Account numbers
Other fixed-format identifiers
🔍 Depth Probe

Ask:

"How would you prevent this class of bug systematically?"

💡 Strong Answer

Define an explicit schema:

from pyspark.sql.types import (
    StructType,
    StructField,
    IntegerType,
    StringType
)


schema = StructType([
    StructField("id", IntegerType(), True),
    StructField("zip_code", StringType(), True)
])


df = (
    spark.read
    .option("header", "true")
    .schema(schema)
    .csv("/data/zips/*.csv")
)

The candidate should understand that schema inference is convenient for exploration but should not blindly be relied upon for production ingestion, especially when the semantic type of a field differs from its apparent physical representation.






# Multiple Actions — Re-reading an External JDBC Source


## 🎯 Scenario


```python
df = (
    spark.read
    .format("jdbc")
    .option("url", "jdbc:postgresql://db-host/prod")
    .option("dbtable", "orders")
    .option("user", "svc_account")
    .option("password", "***")
    .load()
)


active_count = (
    df.filter(F.col("status") == "active")
      .count()
)


total_count = df.count()


sample = (
    df.filter(F.col("status") == "active")
      .limit(10)
      .collect()
)
🎯 Ask

"How many times can this code access the live PostgreSQL source, and why?"

✅ Expected Answer

DataFrame transformations are lazy.

Each action can trigger execution.

Here there are three actions:

active_count
total_count
sample

Because the JDBC DataFrame is not persisted, the source extraction can be executed again for each action.

This can therefore result in multiple reads against the live database.

🚨 Why Is This More Than a Spark Performance Problem?

The external database is potentially a production OLTP system.

Repeated reads can create:

Database load
Network load
Connection pressure
Lock/resource contention, depending on the query
Impact on application workloads

The candidate should think about the blast radius outside Spark.

🔥 Depth Probe

Ask:

"Beyond simply adding cache(), what would you control?"

Look for:

JDBC partitioning
numPartitions
fetchsize
Connection pool limits
Reading from a read replica
Reducing source queries
Extracting once and reusing the result
Monitoring source-system load

For example:

df.persist()

may help avoid repeated reads when the same extracted DataFrame is reused, but the candidate should understand that persistence itself requires an action to materialize the cache.

🚨 Red Flags

Be cautious if the candidate says:

❌ "Spark automatically caches DataFrames."
❌ "The JDBC query only runs once because df is defined once."
❌ "Add more executors."
❌ They don't recognize the production database impact.
❌ They only discuss Spark performance and ignore the external system.
🎯 Scoring Guide
Question	Primary Skill Tested
union()	Schema alignment and silent corruption
orderBy() + repartition()	Ordering guarantees
persist()	Persistence semantics
Malformed JSON	Data quality and corrupt records
Window without partitionBy()	Window semantics
toPandas()	Driver memory
Schema inference	Data correctness
JDBC multiple actions	External-system impact
⭐ What Makes This Set Difficult

These questions are deliberately designed so that the code often runs successfully.

The candidate therefore cannot rely only on:

"Find the error and fix the syntax."

They must identify silent correctness problems, distributed execution behavior, and production consequences.

The strongest candidates should consistently follow:

Read the code
     ↓
Predict behavior
     ↓
Identify the hidden problem
     ↓
Explain WHY
     ↓
Measure / verify
     ↓
Fix
     ↓
Handle the "change one thing" follow-up

That is a much stronger test of real PySpark experience than asking candidates to write generic groupBy, join, or filter syntax.





# `groupBy().first()` / `last()` — No Deterministic Ordering

## 🎯 Scenario

```python
from pyspark.sql import functions as F

df = spark.createDataFrame(
    [
        (1, "2024-01-01", "pending"),
        (1, "2024-01-03", "shipped"),
        (1, "2024-01-02", "processing"),
        (2, "2024-01-01", "pending")
    ],
    ["order_id", 







# Partition Count After a Filter


## 🎯 Scenario


```python
from pyspark.sql import functions as F


df = spark.range(
    0,
    1_000_000,
    numPartitions=200
)


filtered = df.filter(
    F.col("id") < 10
)


print(
    filtered.rdd.getNumPartitions()
)
🎯 Ask

"The filter leaves only 10 rows. How many partitions does filtered have?"

✅ Expected Answer

Typically:

200

A filter is a narrow transformation and does not automatically reduce the number of partitions.

Therefore, you can have:

200 partitions
       ↓
10 matching rows
       ↓
Most partitions contain no matching rows

The candidate should understand that:

Number of rows and number of partitions are independent concepts.

Filtering data does not automatically mean Spark will reduce the partition count.

🔥 Depth Probe

Ask:

"You're about to write this tiny result. What might you do?"

Potentially:

filtered = filtered.coalesce(1)

or coalesce to another appropriate number of partitions:

filtered = filtered.coalesce(2)

The choice depends on the expected output size and downstream requirements.

⚠️ Important Follow-Up

Ask:

"Why shouldn't we blindly add coalesce() after every filter?"

A strong candidate should explain that reducing partitions has a trade-off.

For example:

Large Dataset
     ↓
Filter
     ↓
Still Millions of Rows
     ↓
coalesce(1)
     ↓
Single Partition
     ↓
Limited Parallelism / Potential Bottleneck

coalesce() is useful when the resulting dataset is sufficiently small, particularly before writing a small output.

But blindly using:

df.coalesce(1)

can create a single-partition bottleneck and reduce parallelism.

🎯 Core Principle

A filter reduces the amount of data, not necessarily the number of partitions.

The candidate should distinguish between:

Data volume → number of rows / bytes
Partition count → number of Spark partitions
Parallelism → how many partitions can be processed concurrently

A strong candidate should also know that partition behavior can change through other operations such as:

repartition()

or:

coalesce()

and through shuffle-producing transformations.





# Case Sensitivity and Whitespace — Silent Join Mismatch


## 🎯 Scenario


```python
from pyspark.sql import SparkSession


left = spark.createDataFrame(
    [
        ("Acme Corp",),
        (" Globex ",),
        ("Initech",)
    ],
    ["company"]
)


right = spark.createDataFrame(
    [
        ("acme corp", 100),
        ("Globex", 200),
        ("Initech", 300)
    ],
    ["company", "revenue"]
)


joined = left.join(
    right,
    "company",
    "left"
)


joined.show()
🎯 Ask

"Which rows successfully match and get a non-null revenue? Predict all three."

✅ Expected Answer

Only:

Initech → 300

matches.

Why?
1. Acme Corp does not match acme corp
"Acme Corp"
      ≠
"acme corp"

Spark string comparison is case-sensitive by default in this context.

2. Globex does not match Globex
" Globex "
      ≠
"Globex"

The first value contains leading/trailing whitespace.

3. Initech matches exactly
"Initech"
    =
"Initech"

Therefore:

company     revenue
----------  -------
Acme Corp   NULL
 Globex     NULL
Initech     300
🔥 Depth Probe

Ask:

"Would you automatically apply lower() and trim() to every join key?"

✅ Strong Answer

No.

Blind normalization can create:

False matches
Hidden upstream data-quality problems
Unexpected business-level matches
Difficulty identifying the original source-system issue

Normalization should be:

Deliberate
Validated
Business-approved
Measured

For example, if the business explicitly defines company names as case-insensitive and whitespace-insensitive, you could normalize the keys:

from pyspark.sql import functions as F


left_normalized = left.withColumn(
    "company_key",
    F.lower(F.trim(F.col("company")))
)


right_normalized = right.withColumn(
    "company_key",
    F.lower(F.trim(F.col("company")))
)


joined = left_normalized.join(
    right_normalized,
    "company_key",
    "left"
)
🧠 Advanced Follow-Up

Ask:

"If you normalize the join key and suddenly the match rate increases from 92% to 99.8%, is that automatically good?"

A strong candidate should say no.

They should ask:

Why were the original 8% unmatched?
Is the normalization business-approved?
Did any different companies collapse to the same normalized key?
Did the number of duplicate matches increase?
What happened to join cardinality?
Should the source data be corrected instead?

The key principle is:

A higher join-match rate does not automatically mean better data quality.

🎯 Core Principle

Never treat string normalization as a generic performance or correctness fix. First establish the business definition of the join key.

The strongest candidates should distinguish between:

Technical normalization
        ↓
Business-approved identity
        ↓
Validated join logic
        ↓
Data-quality monitoring

rather than simply writing:

lower(trim(column))

for every join.


