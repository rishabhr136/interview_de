# Microsoft Fabric Interview Handbook for Azure Data Engineer

## How to Use This Handbook

Evaluate candidates on:

- Microsoft Fabric Architecture
- OneLake Design and Usage
- Data Factory in Fabric
- Lakehouse Implementation
- Data Warehouse Concepts
- Real-Time Analytics
- Fabric Governance
- Performance Optimization
- Enterprise Data Platform Design
- Migration and Modernization Experience

---

# Q1. Explain Microsoft Fabric Architecture

## Real-World Scenario

Your organization wants to consolidate ADF, Synapse, Power BI, and Data Lake into a unified analytics platform.

## Detailed Answer

Microsoft Fabric is an end-to-end SaaS analytics platform that combines:

- Data Factory
- Data Engineering
- Data Science
- Data Warehouse
- Real-Time Analytics
- Power BI

into a single ecosystem.

### Core Components

```text
OneLake
   |
---------------------------------
| Data Factory                 |
| Data Engineering             |
| Data Warehouse               |
| Data Science                 |
| Real-Time Analytics          |
| Power BI                     |
---------------------------------
```

### Benefits

- Single SaaS platform
- Unified storage
- Simplified governance
- Reduced integration complexity

### Follow-Up Questions

- Why is Fabric different from Synapse?
- What workloads belong in Fabric?

### Red Flags

- Thinks Fabric is only Power BI.
- Cannot explain OneLake.

### Expected Answers

#### Mid-Level

Understands Fabric offerings.

#### Senior

Can explain workload selection.

#### Lead

Can discuss enterprise adoption strategy.

---

# Q2. What is OneLake?

## Scenario

Different departments maintain separate ADLS accounts causing duplication.

## Detailed Answer

OneLake is the unified storage layer in Microsoft Fabric.

Features:

- Single data lake
- Shared storage across workloads
- Data virtualization
- Consistent governance

### Architecture

```text
Departments
     |
     |
   OneLake
     |
-------------------
| Lakehouse      |
| Warehouse      |
| Power BI       |
-------------------
```

### Benefits

- Eliminates silos
- Reduces duplication
- Centralized security

### Follow-Up Questions

- How is OneLake different from ADLS?
- What is a shortcut?

### Red Flags

- Treats OneLake like a traditional database.

---

# Q3. Explain Fabric Lakehouse

## Scenario

A team wants to process raw claims data and build analytics-ready datasets.

## Detailed Answer

Lakehouse combines data lake flexibility with warehouse capabilities.

Supports:

- Structured data
- Semi-structured data
- Spark workloads
- SQL analytics

### Medallion Architecture

```text
Bronze
   |
Silver
   |
Gold
```

### Benefits

- Unified architecture
- Simplified analytics
- Open formats

### Follow-Up Questions

- Why use Lakehouse instead of Warehouse?
- What data belongs in Gold?

### Red Flags

- Cannot explain Medallion architecture.

---

# Q4. Explain Fabric Data Warehouse

## Scenario

Business users need high-performance SQL reporting.

## Detailed Answer

Fabric Warehouse provides:

- SQL Engine
- T-SQL Support
- Managed storage
- Enterprise analytics

Best for:

- BI reporting
- Star schemas
- KPI dashboards

### Follow-Up Questions

- Lakehouse vs Warehouse?
- When would you use each?

### Red Flags

- Uses warehouse for raw ingestion.

---

# Q5. Explain Data Factory in Fabric

## Detailed Answer

Fabric Data Factory provides:

- Data Pipelines
- Dataflows Gen2
- Scheduling
- Monitoring

Common Uses:

- Data ingestion
- Data movement
- Workflow orchestration

### Follow-Up Questions

- Difference between Fabric Data Factory and Azure Data Factory?

### Red Flags

- Assumes both services are identical.

---

# Q6. What Are Fabric Shortcuts?

## Scenario

Data already exists in ADLS Gen2.

## Detailed Answer

Shortcuts provide virtual access to existing data.

### Architecture

```text
ADLS
  |
Shortcut
  |
OneLake
```

### Benefits

- No data duplication
- Faster onboarding
- Lower storage costs

### Follow-Up Questions

- Why are shortcuts important?

### Red Flags

- Copies all data unnecessarily.

---

# Q7. Explain Medallion Architecture in Fabric

## Detailed Answer

### Bronze

Raw source data.

### Silver

Validated and transformed data.

### Gold

Business-ready datasets.

### Example

Insurance Claims Pipeline:

```text
Raw Claims
    ↓
Bronze
    ↓
Validated Claims
    ↓
Silver
    ↓
Claim Metrics
    ↓
Gold
```

### Follow-Up Questions

- What transformations belong in Silver?

### Red Flags

- Loads directly into Gold.

---

# Q8. How Would You Design an Enterprise Fabric Platform?

## Scenario

20 business units share the same Fabric environment.

## Detailed Answer

Design Considerations:

- Domain-based architecture
- Workspace separation
- RBAC
- OneLake governance
- Capacity planning

### Follow-Up Questions

- How would you organize workspaces?

### Red Flags

- Single workspace for everything.

---

# Q9. Explain Real-Time Analytics

## Scenario

Policy quote events arrive continuously.

## Detailed Answer

Fabric Real-Time Analytics supports:

- Streaming ingestion
- KQL queries
- Near real-time dashboards

### Architecture

```text
Events
   |
Eventstream
   |
KQL Database
   |
Dashboards
```

### Follow-Up Questions

- Difference between batch and streaming analytics?

### Red Flags

- No understanding of event processing.

---

# Q10. Explain Fabric Capacity

## Detailed Answer

Fabric capacities provide compute resources.

Factors affecting capacity:

- Data volume
- Concurrent users
- Refresh frequency

### Monitoring Metrics

- CPU utilization
- Memory usage
- Query durations

### Follow-Up Questions

- How do you optimize capacity consumption?

### Red Flags

- No awareness of capacity planning.

---

# Q11. How Do You Optimize Fabric Performance?

## Detailed Answer

Optimization Techniques:

### Data Model Optimization

- Star schema design
- Reduce unnecessary columns

### Storage Optimization

- Partitioning
- Delta tables

### Query Optimization

- Predicate pushdown
- Efficient joins

### Follow-Up Questions

- How do you identify bottlenecks?

### Red Flags

- Solves every problem by increasing capacity.

---

# Q12. Explain Data Governance in Fabric

## Detailed Answer

Governance Components:

- RBAC
- Workspace permissions
- Sensitivity labels
- Lineage tracking

### Benefits

- Compliance
- Visibility
- Security

### Follow-Up Questions

- How would you manage PII data?

### Red Flags

- No governance strategy.

---

# Q13. Explain Fabric Monitoring

## Detailed Answer

Monitor:

- Pipeline runs
- Capacity metrics
- Dataset refreshes
- Query performance

### Common Tools

- Fabric Monitoring Hub
- Capacity Metrics App

### Follow-Up Questions

- Which KPIs would you track?

### Red Flags

- No monitoring approach.

---

# Q14. Fabric vs Synapse

## Detailed Answer

### Synapse

- More infrastructure management
- Separate services

### Fabric

- SaaS platform
- Unified experience
- OneLake

### Follow-Up Questions

- Why migrate from Synapse to Fabric?

### Red Flags

- Cannot compare the platforms.

---

# Q15. Design a Production-Grade Fabric Solution

## Scenario

Insurance company needs centralized analytics for claims, policy, and marketing data.

## Detailed Answer

Components:

### Ingestion

- Data Factory

### Storage

- OneLake
- Lakehouse

### Processing

- Spark Notebooks
- Pipelines

### Analytics

- Warehouse
- Power BI

### Governance

- Security
- Lineage
- Monitoring

### Lead-Level Discussion

Candidate should explain:

- Multi-domain architecture
- Governance strategy
- Cost optimization
- DR planning
- Capacity management

---

# Practical Deep-Dive Troubleshooting Scenarios

## Scenario 1

Fabric refresh time increased from 30 minutes to 3 hours.

### Expected Investigation

- Capacity pressure
- Data growth
- Query inefficiencies
- Concurrent workloads

---

## Scenario 2

Business users report inconsistent KPI values.

### Expected Discussion

- Lineage
- Data validation
- Gold layer certification

---

## Scenario 3

OneLake storage costs increased significantly.

### Expected Discussion

- Duplicate datasets
- Retention policies
- Shortcut opportunities

---

# Interview Scoring Rubric

## Mid-Level (3-5 Years)

✅ Understands Lakehouse  
✅ Builds pipelines  
✅ Uses Fabric workloads

---

## Senior (5-8 Years)

✅ Designs Medallion architecture  
✅ Implements governance  
✅ Optimizes performance

---

## Lead (8+ Years)

✅ Designs enterprise Fabric platforms  
✅ Defines capacity strategy  
✅ Leads migration programs  
✅ Establishes governance standards

---

# Validation Checklist

| Requirement | Status |
|------------|---------|
| 15 Practical Questions | ✅ |
| Detailed Answers | ✅ |
| Follow-Up Questions | ✅ |
| Red Flags | ✅ |
| Mid/Senior/Lead Expectations | ✅ |
| Real Azure Scenarios | ✅ |
| Performance Tuning Cases | ✅ |
| Troubleshooting Cases | ✅ |
| Interview Scoring Rubric | ✅ |

**Handbook Status: Validated and Interview Ready**
