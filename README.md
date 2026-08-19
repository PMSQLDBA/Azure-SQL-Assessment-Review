# Azure SQL FinOps Automation

## Purpose

Automate Azure SQL cost assessment to identify **safe cost-saving opportunities**, track implementation, and validate **actual realized savings**.

## Architecture

```text
Azure Subscriptions
        ↓
Resource Graph ── Azure SQL Inventory
        ↓
Azure Monitor ─── 30/60/90-Day Performance
        ↓
Cost Management ─ Actual / Amortized Cost
        ↓
Azure Advisor ─── Microsoft Recommendations
        ↓
Unified Assessment Dataset
        ↓
Recommendation Engine
        ↓
Dashboard + Approval
        ↓
Controlled Remediation
        ↓
Post-Change Validation
        ↓
Realized Savings
```

## Optimization Rules

| Rule         | Focus                     |
| ------------ | ------------------------- |
| SQL-COST-001 | Compute Rightsizing       |
| SQL-COST-002 | Idle Databases            |
| SQL-COST-003 | Serverless Candidates     |
| SQL-COST-004 | Elastic Pool Candidates   |
| SQL-COST-005 | Business Critical Review  |
| SQL-COST-006 | Reservation Opportunities |
| SQL-COST-007 | Azure Hybrid Benefit      |
| SQL-COST-008 | Storage Optimization      |
| SQL-COST-009 | Query/Performance Cost    |
| SQL-COST-010 | Cost Spikes               |

## Recommendation Output

```text
Database
Current SKU
30-Day Utilization
Current Cost
Recommended Action
Target SKU
Monthly / Annual Savings
Confidence
Risk
Priority
Owner
Status
```

## Governance

```text
Identified
   ↓
Technical Review
   ↓
Owner Approval
   ↓
Implemented
   ↓
7/30-Day Validation
   ↓
Savings Confirmed
```

Use separate **read-only assessment** and **remediation** managed identities. Never automatically resize production databases without validation and approval.

## Dashboard

Track:

```text
Current Spend
Identified Savings
Approved Savings
Implemented Savings
Realized Savings
Savings Realization %
```

## Goal

> **Understand Azure SQL spend → identify safe optimizations → implement approved changes → prove the Azure bill actually decreased.**
