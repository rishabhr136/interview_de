# Azure Data Engineer Master Interview Playbook

## Version

Enterprise Edition

## Target Audience

- Interviewers
- Technical Leads
- Engineering Managers
- Solution Architects
- Hiring Panels

---

# Table of Contents

1. Interview Framework
2. Candidate Evaluation Matrix
3. PySpark Handbook
4. Azure Data Factory Handbook
5. Databricks Handbook
6. SQL Handbook
7. Microsoft Fabric Handbook
8. Python Handbook
9. Azure Blob Storage Handbook
10. Architecture Design Rounds
11. Production Troubleshooting Rounds
12. Practical Coding Assessments
13. Hiring Scorecards
14. Interview Debrief Templates

---

# Section 1: Interview Framework

## Candidate Evaluation Matrix

| Skill Area | Weight |
|------------|---------|
| SQL | 20% |
| PySpark | 20% |
| Databricks | 15% |
| Azure Data Factory | 15% |
| Microsoft Fabric | 10% |
| Python | 10% |
| Azure Storage | 10% |

---

## Recommended Interview Flow

### Round 1

SQL + Python

Duration:

90 Minutes

---

### Round 2

PySpark + Databricks

Duration:

90 Minutes

---

### Round 3

ADF + Azure Storage

Duration:

60 Minutes

---

### Round 4

Architecture Design

Duration:

90 Minutes

---

### Round 5

Leadership and Stakeholder Management

Duration:

60 Minutes

---

# Section 2: Experience Expectations

## Mid-Level Data Engineer (3-5 Years)

Expected Skills

- SQL development
- ETL implementation
- Basic Spark
- Pipeline support
- Production monitoring

Red Flags

- No debugging experience
- No cloud exposure

---

## Senior Data Engineer (5-8 Years)

Expected Skills

- End-to-end platform ownership
- Performance tuning
- CI/CD
- Security implementation

Red Flags

- Pure development experience
- No production ownership

---

## Lead Data Engineer (8+ Years)

Expected Skills

- Enterprise architecture
- Governance
- Cost optimization
- Incident leadership

Red Flags

- No architecture ownership
- No stakeholder engagement

---

# Universal Question Template

Every technical question in the handbook should use the following format.

---

## Question #

### Topic

### Difficulty

- Intermediate
- Advanced
- Expert

### Business Scenario

Explain real-world project background.

### Primary Question

Question asked to candidate.

### Why We Ask This

Evaluation objective.

### Detailed Expected Answer

Comprehensive answer.

### Practical Example

Industry example.

### Sample Code

SQL, Python, PySpark etc.

### Architecture Diagram

ASCII representation.

### Performance Considerations

What to optimize.

### Cost Considerations

How cloud costs are impacted.

### Security Considerations

Data protection concerns.

### Troubleshooting Discussion

Expected debugging approach.

### Follow-Up Questions

3-10 probing questions.

### Red Flags

Common weak answers.

### Mid-Level Expectation

Minimum acceptable answer.

### Senior-Level Expectation

Strong answer.

### Lead-Level Expectation

Architecture-level answer.

### Score

1-5 Rating Guide

---

# Section 3: Architecture Design Round

## Case Study 1

Marketing Analytics Platform

Requirements

- Google Ads API
- CRM Data
- Daily Refresh
- Reporting

Expected Architecture

Google Ads
      |
ADF
      |
ADLS
      |
Databricks
      |
Gold Layer
      |
Power BI

Evaluation Areas

- Scalability
- Security
- Cost
- Reliability

---

## Case Study 2

Insurance Claim Analytics Platform

Requirements

- 5 Billion Claims
- Historical Retention
- Near Real-Time Updates

Expected Topics

- Delta Lake
- CDC
- Medallion
- Data Governance
- Storage Design

---

## Case Study 3

Enterprise Microsoft Fabric Migration

Requirements

Current Platform:

- ADF
- Databricks
- SQL DW
- Power BI

Target:

Microsoft Fabric

Expected Discussion

- OneLake
- Lakehouse
- Warehouse
- Shortcuts
- Capacity Planning

---

# Section 4: Production Support Round

## Scenario 1

ETL Runtime Jumped

From:

2 Hours

To:

8 Hours

Expected Investigation

- Data Volume
- Query Plan
- Data Skew
- Cluster Changes

---

## Scenario 2

Pipeline Success But Data Wrong

Expected Investigation

- Watermarks
- Reconciliation
- Audit Tables

---

## Scenario 3

Storage Costs Increased

Expected Investigation

- Lifecycle Policy
- Retention
- Data Duplication

---

## Scenario 4

Databricks Out Of Memory

Expected Investigation

- Skew
- Cache Usage
- Partitions

---

# Section 5: Practical Coding Assessments

## SQL Assessment

Latest Customer Policy

Topics

- Window Functions
- Ranking
- Optimization

---

## Python Assessment

API Ingestion Framework

Topics

- Pagination
- Retry Logic
- Logging

---

## PySpark Assessment

Deduplication Framework

Topics

- Window Functions
- Delta Merge
- Optimization

---

## Databricks Assessment

Bronze To Silver Pipeline

Topics

- Delta Lake
- Auto Loader
- Quality Checks

---

# Section 6: Hiring Decision Matrix

## Scorecard

| Area | Weight |
|--------|--------|
| SQL | 20 |
| PySpark | 20 |
| Databricks | 15 |
| ADF | 15 |
| Fabric | 10 |
| Python | 10 |
| Azure Storage | 10 |

---

## Strong Hire

Score

85+

Characteristics

- Architecture ownership
- Strong troubleshooting
- Production experience
- Stakeholder management

---

## Hire

Score

70-84

Characteristics

- Solid implementation knowledge
- Moderate guidance required

---

## Borderline

Score

60-69

Characteristics

- Limited architecture exposure

---

## No Hire

Score

Below 60

Characteristics

- Poor debugging
- Weak fundamentals
- No production ownership

---

# Handbook Completion Checklist

For Every Topic Ensure:

✅ 15 Core Questions

✅ Detailed Answers

✅ Code Examples

✅ Real World Scenarios

✅ Production Troubleshooting Examples

✅ Follow-up Questions

✅ Red Flags

✅ Mid-Level Expectations

✅ Senior-Level Expectations

✅ Lead-Level Expectations

✅ Performance Optimization Notes

✅ Cost Optimization Notes

✅ Security Considerations

✅ Scoring Rubric

---

# Final Target State

Topics

1. SQL (15 Questions)
2. PySpark (15 Questions)
3. Databricks (15 Questions)
4. Azure Data Factory (15 Questions)
5. Microsoft Fabric (15 Questions)
6. Python (15 Questions)
7. Azure Blob Storage (15 Questions)

Total

105 Enterprise Interview Questions

Supporting Material

- Architecture Scenarios
- Production Outages
- Hands-On Exercises
- Hiring Rubrics
- Evaluation Templates

This document serves as the Azure Data Engineer Interviewer's Playbook and hiring standard.
