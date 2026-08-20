# PySpark Interview Handbook (Azure Data Engineer)

## How to Use This Handbook
For each question assess:
- Practical implementation experience
- Performance tuning knowledge
- Troubleshooting approach
- Production support experience
- Ability to justify design decisions

---

# Q1. Explain Spark Architecture and Execution Flow
### Real Scenario
A daily 2 TB claims ingestion job takes 3 hours. Explain how Spark executes the workload.

### Detailed Answer
Spark consists of:
1. Driver
2. Cluster Manager
3. Executors
4. Tasks
5. Partitions

Execution Flow:
- Driver creates logical plan.
- Catalyst optimizer creates optimized plan.
- DAG split into stages.
- Stages split into tasks.
- Tasks execute on executors.

Example:
Reading 1 TB parquet with 500 partitions creates hundreds of tasks which run in parallel.

### Follow-up Questions
- What creates a new stage?
- Difference between task and stage?
- How do you view execution plan?

### Red Flags
- Cannot explain Driver vs Executor.
- Thinks Spark processes data row-by-row.

### Expected Answers
Mid: Basic architecture.
Senior: Explain DAG and stages.
Lead: Explain execution tuning, resource allocation and production troubleshooting.

---

# Q2. Explain Data Skew
### Scenario
One join stage runs 45 minutes while others finish in seconds.

### Detailed Answer
Data skew occurs when one partition holds significantly more records.

Example:
Customer A = 80 million rows.
Remaining customers = 20 million rows.

Solutions:
- Salting
- Broadcast join
- AQE
- Better partition keys

### Troubleshooting
Check Spark UI.
Look for one long-running task.

### Follow-up
How have you fixed skew in production?

### Red Flags
Never used Spark UI.

### Expected Levels
Mid: Define skew.
Senior: Discuss solutions.
Lead: Quantify impact and explain tradeoffs.

---

# Q3. Repartition vs Coalesce
### Detailed Answer
Repartition:
- Full shuffle
- Increase/decrease partitions

Coalesce:
- Usually decrease partitions
- Minimal shuffle

Example:
1000 small output files reduced to 20 files before ADLS write.

### Follow-up
Why is repartition expensive?

### Red Flags
Uses repartition everywhere.

---

# Q4. Broadcast Join
### Scenario
500 GB fact table joins 20 MB product dimension.

### Detailed Answer
Broadcast sends small table to executors.
Avoids large shuffle.

Example:
Use broadcast(product_dim).

Benefits:
- Faster execution
- Reduced network traffic

### Follow-up
When should you NOT broadcast?

---

# Q5. Troubleshooting Slow Jobs
### Detailed Answer
Review:
- Spark UI
- Shuffle size
- Skew
- Storage latency
- Executor utilization
- Join strategy

Production Method:
1. Identify slow stage.
2. Find root cause.
3. Tune and benchmark.

### Red Flags
Immediately increases cluster size without investigation.

---

# Q6. What Causes Shuffle?
### Detailed Answer
Operations:
- groupBy
- join
- orderBy
- distinct
- repartition

Impacts:
- Disk spill
- Network transfer
- Longer execution times

### Follow-up
Which transformation causes largest shuffle in your project?

---

# Q7. Cache vs Persist
### Detailed Answer
cache() = MEMORY_ONLY
persist() allows MEMORY_AND_DISK.

Project Example:
Feature engineering dataset reused across multiple ML stages.

### Decision Question
When would cache worsen performance?

---

# Q8. Partition Strategy Design
### Scenario
10 TB policy data stored in Delta.

### Detailed Answer
Good partition keys:
- Business date
- Year/month

Bad partition keys:
- Customer ID with millions of values.

Measure:
- Query performance
- Small-file generation

---

# Q9. Adaptive Query Execution
### Detailed Answer
AQE dynamically changes execution plan.

Benefits:
- Better joins
- Skew mitigation
- Dynamic partition sizing

Production impact often reduces runtime significantly without code changes.

---

# Q10. Delta Lake vs Parquet
### Detailed Answer
Delta Adds:
- ACID
- Time Travel
- Merge
- Schema Enforcement
- Schema Evolution

Insurance Example:
Daily policy updates via merge instead of full reload.

---

# Q11. Incremental Processing Design
### Scenario
Loading new policy transactions daily.

### Detailed Answer
Use:
- Watermarks
- CDC
- Delta Merge

Benefits:
- Faster execution
- Lower compute cost

---

# Q12. How Do You Handle Duplicate Records?
### Detailed Answer
Methods:
- dropDuplicates
- Window functions
- Merge logic

Production Example:
Keep latest transaction by transaction timestamp.

---

# Q13. Out Of Memory Investigation
### Detailed Answer
Causes:
- collect()
- Large shuffle
- Excess caching
- Skew

Investigation Steps:
- Executor logs
- Spark UI
- Partition counts

---

# Q14. Small File Problem
### Detailed Answer
Millions of files cause metadata overhead.

Solutions:
- OPTIMIZE
- Coalesce
- Proper write partitioning

Insurance Example:
Daily ingestion producing thousands of tiny files.

---

# Q15. Design a Production Grade PySpark Framework
### Detailed Answer
Components:
- Metadata driven ingestion
- Audit framework
- Logging
- Watermarks
- Retry handling
- Monitoring dashboards
- Data quality rules

A Lead Engineer should discuss:
- CI/CD
- Unit testing
- Recovery strategy
- Cost optimization
- Security controls

---

# Practical Scenarios for Deep-Dive Discussion
1. 2 TB join suddenly doubled in runtime.
2. Delta table grew to 50 million files.
3. Executor memory failures during month-end runs.
4. CDC feed delivered duplicate transactions.
5. Daily SLA missed after schema changes.

# Interview Scoring Rubric
Mid-Level (3-5 Years)
- Understands Spark fundamentals.
- Can build ETL pipelines.

Senior (5-8 Years)
- Can tune performance.
- Can troubleshoot production incidents.
- Understands Delta architecture.

Lead (8+ Years)
- Designs enterprise frameworks.
- Optimizes cost and scalability.
- Leads production recovery efforts.
- Mentors teams and defines standards.
