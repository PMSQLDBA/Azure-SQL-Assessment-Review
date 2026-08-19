# Azure SQL FinOps Automation

## Automated Azure SQL Cost Assessment, Optimization, and Savings Tracking

This solution provides a reusable framework for assessing Azure SQL
costs, identifying optimization opportunities, tracking recommendations,
and measuring the savings actually realized after changes are
implemented.

The main goal is simple:

> **Find where Azure SQL money is being spent, identify safe
> opportunities to reduce the cost, get the right approvals, implement
> the change, and prove that the Azure bill actually went down.**

------------------------------------------------------------------------

## 1. Why This Solution Exists

Azure SQL environments can grow over time. Databases may be created for
applications, development, testing, reporting, migrations, or temporary
projects.

Common cost issues include:

-   Databases running on more vCores than they need.
-   Development and test databases running continuously.
-   Databases using an expensive service tier without needing its
    capabilities.
-   Many small databases using dedicated compute instead of sharing an
    Elastic Pool.
-   Intermittent databases using provisioned compute instead of
    Serverless.
-   Eligible workloads not using Azure Hybrid Benefit.
-   Stable workloads not taking advantage of reservation opportunities.
-   Storage growing because old or unnecessary data is retained.
-   Poor queries consuming CPU and IO and making the database appear
    undersized.
-   Azure SQL costs increasing without clear ownership or investigation.
-   Recommendations being identified but never implemented.
-   Expected savings being reported without verifying actual savings.

This framework addresses these problems through continuous data
collection, automated analysis, governance, and savings tracking.

------------------------------------------------------------------------

## 2. High-Level Architecture

``` text
                    Azure Subscriptions
                           |
            +--------------+--------------+
            |              |              |
            v              v              v
    Resource Graph    Azure Monitor    Cost Management
      Inventory          Metrics           Export
            |              |              |
            +-------+------+--------------+
                    |
                    v
             Data Collection
                    |
                    +------------------+
                    |                  |
                    v                  v
             Azure Advisor      SQL Cost Rules
                    |                  |
                    +--------+---------+
                             |
                             v
                    Recommendation Engine
                             |
                             v
                   Recommendation Repository
                             |
                  +----------+-----------+
                  |                      |
                  v                      v
          Dashboard / Reporting      Approval Workflow
                                         |
                                         v
                                 Remediation Process
                                         |
                                         v
                                 Post-Change Monitoring
                                         |
                                         v
                                   Realized Savings
```

------------------------------------------------------------------------

## 3. Main Solution Components

  -----------------------------------------------------------------------
  Component                           Purpose
  ----------------------------------- -----------------------------------
  Azure Resource Graph                Discover Azure SQL resources across
                                      subscriptions

  Azure Monitor                       Collect database utilization and
                                      performance metrics

  Azure Cost Management               Provide actual or amortized Azure
                                      billing information

  Azure Advisor                       Import Microsoft's existing
                                      optimization recommendations

  Azure Function / Automation         Run collection and assessment logic

  Recommendation Engine               Apply SQL-specific cost
                                      optimization rules

  Azure SQL / Data Store              Store findings, status, ownership,
                                      and savings

  Logic App / Workflow                Route recommendations for review
                                      and approval

  Power BI / Excel / Workbook         Present technical and executive
                                      dashboards

  Azure Automation / DevOps           Implement approved remediation

  Managed Identity                    Secure service-to-service
                                      authentication

  Key Vault                           Store secrets only where
                                      identity-based access cannot be
                                      used
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 4. Solution Principles

### 4.1 Read-only first

The assessment process should begin with read-only access.

The assessment identity discovers resources, reads monitoring
information, reads cost information, and retrieves recommendations.

It should **not** initially have permission to resize or modify
production databases.

### 4.2 Separate assessment from remediation

Use separate identities:

``` text
Assessment Managed Identity
        |
        +-- Resource inventory
        +-- Monitoring
        +-- Cost
        +-- Advisor

Remediation Managed Identity
        |
        +-- Approved Azure SQL changes only
```

This reduces risk and supports enterprise change-control requirements.

### 4.3 Recommendations are not automatic production changes

Low CPU does not automatically mean a database should be resized.

Before changing production capacity, validate:

-   Month-end processing.
-   Quarter-end processing.
-   Overnight batch jobs.
-   Application latency requirements.
-   SLA requirements.
-   Worker and session limits.
-   IO requirements.
-   Memory requirements.
-   HA/DR requirements.
-   Zone redundancy.
-   Read replicas.
-   Application owner approval.
-   Rollback configuration.

------------------------------------------------------------------------

# 5. Phase 1 --- Azure SQL Inventory

The first step is discovering the complete Azure SQL estate.

Azure Resource Graph is used to find resources across subscriptions.

Example inventory query:

``` kusto
Resources
| where type =~ "microsoft.sql/servers/databases"
| where name !~ "master"
| extend
    serverName = tostring(split(id, "/")[8]),
    databaseName = name,
    skuName = tostring(sku.name),
    skuTier = tostring(sku.tier),
    skuCapacity = toint(sku.capacity),
    maxSizeBytes = tolong(properties.maxSizeBytes),
    elasticPoolId = tostring(properties.elasticPoolId)
| project
    subscriptionId,
    resourceGroup,
    serverName,
    databaseName,
    location,
    skuName,
    skuTier,
    skuCapacity,
    maxSizeBytes,
    elasticPoolId,
    tags,
    resourceId = id
```

The inventory should contain at least:

``` text
Subscription
Resource Group
SQL Server
Database
Resource ID
Region
Environment
Application
Owner
Service Tier
SKU
vCores
Storage
Elastic Pool
Zone Redundancy
Tags
```

The full Azure Resource ID should be used as the primary identifier.

Do not join data using only database name because database names can be
repeated across subscriptions and servers.

------------------------------------------------------------------------

# 6. Phase 2 --- Performance Metrics

For every Azure SQL Database, collect performance information from Azure
Monitor.

Recommended metrics include:

``` text
CPU %
Data IO %
Log IO %
Sessions %
Workers %
Storage %
```

Depending on the SQL resource type, additional metrics may be available.

Managed Instance should be handled separately because its metric set
differs from Azure SQL Database.

## Observation windows

Recommended periods:

``` text
7 days
14 days
30 days
60 days
90 days
```

For cost rightsizing, 30 days is a useful initial minimum, but longer
periods should be used when workloads have monthly or seasonal peaks.

Calculate:

``` text
Average
Maximum
P50
P90
P95
P99
```

P95 is particularly useful because averages can hide short but important
workload peaks.

Example:

  Database    SKU            Avg CPU   P95 CPU   P95 Data IO   P95 Log IO
  ----------- ------------ --------- --------- ------------- ------------
  OrdersDB    GP 8 vCore         13%       32%           27%          19%
  BillingDB   GP 8 vCore         39%       78%           61%          49%
  HRDB        GP 4 vCore          3%        8%            6%           5%

------------------------------------------------------------------------

# 7. Phase 3 --- Azure Cost Management

Performance tells us whether a database may be oversized.

Cost Management tells us whether optimizing it is financially
meaningful.

Use Azure Cost Management recurring exports as the primary billing feed.

Recommended flow:

``` text
Azure Cost Management
        |
        v
Scheduled Export
        |
        v
Storage Account / ADLS
        |
        v
Cost Reader
        |
        v
Normalize Resource ID
        |
        v
Aggregate Cost by Resource
```

Store information such as:

``` text
ResourceId
Date
ActualCost
AmortizedCost
Currency
SubscriptionId
ResourceGroup
MeterCategory
MeterName
```

## Actual vs. amortized cost

Choose one standard with the customer's FinOps team.

**Actual cost** is useful when reporting closely against invoiced
charges.

**Amortized cost** is useful for FinOps analysis because reservation
purchases can be spread across the resources benefiting from them.

Do not mix the two methods in the same savings KPI without clearly
identifying the basis.

------------------------------------------------------------------------

# 8. Phase 4 --- Azure Advisor

Azure Advisor should be integrated rather than duplicated.

The framework imports Advisor cost recommendations and stores them
alongside custom Azure SQL recommendations.

Recommendation source should be identified as:

``` text
CUSTOM
AZURE_ADVISOR
```

This allows the customer to distinguish Microsoft's Advisor
recommendation from the SQL-specific assessment engine.

The primary join key remains:

``` text
Azure Resource ID
```

------------------------------------------------------------------------

# 9. Unified Assessment Dataset

Inventory, metrics, cost, and Advisor information are joined together.

``` text
SQL Inventory
     +
Azure Monitor Metrics
     +
Cost Management
     +
Azure Advisor
     |
     v
Unified Assessment Dataset
```

Example fields:

``` text
Subscription
Resource Group
Server
Database
ResourceId

Current SKU
Current Tier
vCores

30-Day Avg CPU
30-Day P95 CPU
30-Day P95 Data IO
30-Day P95 Log IO

Current Cost
Previous Period Cost
Cost Growth %

Advisor Recommendation

Application
Owner
Environment
```

This dataset becomes the input to the Recommendation Engine.

------------------------------------------------------------------------

# 10. Recommendation Engine

The Recommendation Engine applies configurable policies to the unified
dataset.

Thresholds should be stored in configuration rather than hard-coded into
application logic.

Example:

``` json
{
  "observation_days_min": 30,
  "rightsizing": {
    "high_confidence": {
      "p95_cpu_max": 30,
      "p95_data_io_max": 30,
      "p95_log_io_max": 30
    },
    "medium_confidence": {
      "p95_cpu_max": 50,
      "p95_data_io_max": 50,
      "p95_log_io_max": 50
    }
  }
}
```

These are example organizational policy thresholds, not guaranteed
Microsoft sizing limits.

------------------------------------------------------------------------

# 11. Optimization Rules

## SQL-COST-001 --- Compute Rightsizing

Purpose:

Identify databases that may have more compute than required.

Example high-confidence rule:

``` text
Observation >= 30 days

AND

P95 CPU < 30%
P95 Data IO < 30%
P95 Log IO < 30%
```

Recommendation:

``` text
Evaluate the next smaller compute SKU.
```

Example:

``` text
Current:
General Purpose 8 vCore

30-Day Metrics:
P95 CPU       31%
P95 Data IO   27%
P95 Log IO    19%

Recommendation:
Evaluate General Purpose 4 vCore
```

The recommendation should still go through technical validation.

------------------------------------------------------------------------

## SQL-COST-002 --- Idle Database

Purpose:

Find databases that may no longer require dedicated compute.

Example signals:

``` text
Average CPU < 5%
P95 CPU < 10%
Very low connection count
Observation >= 30 days
```

Possible recommendations:

``` text
Retire database
Lower SKU
Move to Serverless
Consolidate
```

Always validate ownership and dependencies before retirement.

------------------------------------------------------------------------

## SQL-COST-003 --- Serverless Candidate

Serverless can be useful for intermittent workloads.

Typical candidates include:

``` text
Development
QA
Sandbox
Training
Intermittent applications
```

Look for:

``` text
Low average CPU
Long idle periods
Few overnight connections
Short active workload windows
```

Example:

``` text
Database: DevCustomerPortal

Active:
4 hours/day

Mostly idle:
20 hours/day

Recommendation:
Evaluate Azure SQL Serverless
```

------------------------------------------------------------------------

## SQL-COST-004 --- Elastic Pool Candidate

Elastic Pools should be evaluated when multiple databases can share
capacity efficiently.

Example:

``` text
SQL Server
   |
   +-- DB-A peaks at 08:00
   +-- DB-B peaks at 11:00
   +-- DB-C peaks at 16:00
   +-- DB-D peaks at 20:00
```

Instead of four independently sized compute allocations, an Elastic Pool
may be more efficient.

Detection should consider:

``` text
Multiple databases
Same logical server
Low average utilization
Different workload peaks
Not already pooled
```

Peak concurrency must be validated before recommending consolidation.

------------------------------------------------------------------------

## SQL-COST-005 --- Business Critical Review

Purpose:

Find databases using Business Critical where the architecture may no
longer require it.

Do not automatically recommend General Purpose based only on CPU.

Validate:

``` text
Latency requirements
HA requirements
Storage/IO requirements
Read-scale requirements
Application architecture
Business SLA
```

Output:

``` text
Architecture Review Required
```

------------------------------------------------------------------------

## SQL-COST-006 --- Azure Reservation Candidate

Look for workloads that are:

``` text
Stable
Running continuously
Predictable
Expected to remain for the long term
```

Recommendation:

``` text
Evaluate reservation coverage with FinOps.
```

Reservations should be validated at the customer's billing and
commitment scope.

------------------------------------------------------------------------

## SQL-COST-007 --- Azure Hybrid Benefit

Check whether the customer has eligible SQL licensing and whether Azure
Hybrid Benefit is already enabled.

Example:

``` text
AHB Eligible = Yes
AHB Enabled = No
```

Recommendation:

``` text
Validate licensing and enable Azure Hybrid Benefit if permitted.
```

The licensing team should confirm eligibility.

------------------------------------------------------------------------

## SQL-COST-008 --- Storage Optimization

Track storage growth.

Example:

``` text
90 days ago: 1.5 TB
Current:     2.4 TB

Growth: 60%
```

Investigate:

``` text
Large tables
Historical data
Audit data
Staging data
Duplicate indexes
Unused indexes
Retention policy
Backup retention
```

------------------------------------------------------------------------

## SQL-COST-009 --- Query-Driven Compute Cost

Not every compute problem should be solved by buying more compute.

Example:

``` text
Top 5 queries consume 65% of CPU.
```

Instead of:

``` text
8 vCores -> 16 vCores
```

first investigate:

``` text
Execution plans
Missing indexes
Duplicate indexes
Blocking
Expensive scans
High-frequency queries
Poor SQL patterns
```

After query optimization, the database may require less compute.

This is why performance tuning is part of the FinOps assessment.

------------------------------------------------------------------------

## SQL-COST-010 --- Cost Spike

Compare cost periods.

Example:

``` text
Previous 30 days: $10,000
Current 30 days:  $13,000

Increase: 30%
```

If the increase exceeds the configured threshold, create an
investigation recommendation.

Possible causes include:

``` text
SKU increase
Storage growth
New database
Higher workload
Additional replicas
Backup/retention changes
Reservation changes
Configuration changes
```

------------------------------------------------------------------------

# 12. Recommendation Output

Every recommendation should contain enough technical and financial
context to make a decision.

Example:

``` text
Database:
OrdersDB

Current SKU:
General Purpose 8 vCore

Observation:
30 days

Avg CPU:
13%

P95 CPU:
32%

P95 Data IO:
27%

P95 Log IO:
19%

Current Monthly Cost:
$1,834

Recommended:
Evaluate GP 4 vCore

Projected Cost:
$1,030/month

Potential Monthly Saving:
$804

Potential Annual Saving:
$9,648

Confidence:
HIGH

Risk:
MEDIUM

Priority:
HIGH

Source:
CUSTOM

Owner:
Orders Application Team

Status:
Identified
```

------------------------------------------------------------------------

# 13. Savings Calculation

For each recommendation:

``` text
Monthly Saving
=
Current Monthly Cost
-
Projected Monthly Cost
```

Annualized:

``` text
Annual Saving
=
Monthly Saving * 12
```

Percentage:

``` text
Savings %
=
Monthly Saving / Current Monthly Cost * 100
```

Example:

``` text
Current Cost:      $2,400/month
Projected Cost:    $1,450/month

Monthly Saving:      $950
Annual Saving:    $11,400
Savings:             39.6%
```

------------------------------------------------------------------------

# 14. Confidence, Risk, and Priority

These are different concepts.

## Confidence

How strongly the data supports the recommendation.

``` text
HIGH
MEDIUM
LOW
```

## Risk

How risky the change could be to the application.

``` text
HIGH
MEDIUM
LOW
```

For example:

``` text
Recommendation:
Resize 8 -> 4 vCore

Confidence:
HIGH

Risk:
MEDIUM
```

The data may strongly support the recommendation, but production change
risk may still exist.

## Priority

Priority should combine:

``` text
Financial benefit
Confidence
Risk
Business importance
Implementation effort
```

------------------------------------------------------------------------

# 15. Recommendation Workflow

Recommended lifecycle:

``` text
Identified
     |
     v
Technical Review
     |
     v
Application Owner Review
     |
     v
Approved
     |
     v
Scheduled
     |
     v
Implemented
     |
     v
Validation
     |
     v
Savings Confirmed
```

Alternative states:

``` text
Rejected
Deferred
Exception
```

------------------------------------------------------------------------

# 16. Controlled Remediation

Only approved recommendations should enter remediation.

Possible implementation tools:

``` text
Azure Automation
Azure Functions
Azure DevOps
GitHub Actions
Logic Apps
ServiceNow workflow
```

Recommended production process:

``` text
Recommendation
      |
      v
Owner Approval
      |
      v
Pre-Change Validation
      |
      v
Capture Current Configuration
      |
      v
Implement Change
      |
      v
24-Hour Monitoring
      |
      v
7-Day Monitoring
      |
      v
30-Day Cost Validation
```

------------------------------------------------------------------------

# 17. Rollback

Every production change should capture:

``` text
Original SKU
Target SKU
Rollback SKU
Change timestamp
Owner
Change ticket
Approval
```

Example:

``` text
Original:
GP 8 vCore

Target:
GP 4 vCore

Rollback:
GP 8 vCore
```

Possible post-change guardrails:

``` text
P95 CPU > 85%

OR

P95 Data IO > 85%

OR

P95 Log IO > 85%

OR

Application latency exceeds SLA
```

The platform should alert the application/DBA team and support a
controlled rollback.

Automatic rollback can be added later if the customer's change policy
allows it.

------------------------------------------------------------------------

# 18. Realized Savings

Potential savings are not the same as actual savings.

Track four separate stages:

``` text
Identified Savings
Approved Savings
Implemented Savings
Realized Savings
```

Example:

``` text
Current cost:
$1,850/month

Expected after change:
$1,050/month

Expected saving:
$800/month

Actual post-change bill:
$1,110/month

Realized saving:
$740/month
```

Dashboard:

``` text
Identified       $800
Approved         $800
Implemented      $800
Realized         $740
```

This provides a defensible FinOps ROI measurement.

------------------------------------------------------------------------

# 19. Customer Dashboard

The customer dashboard should have four main views.

## Executive Dashboard

Show:

``` text
Current Azure SQL Spend
Identified Savings
Approved Savings
Implemented Savings
Realized Savings
Savings Realization %
```

Example:

``` text
Current SQL Spend       $286K/month
Identified Savings       $74K/month
Approved Savings         $51K/month
Implemented Savings      $43K/month
Realized Savings         $39K/month
```

## Savings by Category

Example:

``` text
Rightsizing             $21K
Reservations            $10K
Elastic Pools            $6K
Serverless               $4K
Storage                  $3K
Azure Hybrid Benefit     $2K
Query Optimization       $2K
```

## Recommendation View

Include:

``` text
Database
Recommendation
Priority
Confidence
Risk
Expected Saving
Owner
Status
Age
```

## Database Detail

Include:

``` text
Current SKU
Utilization
Cost
Advisor findings
Custom findings
Target configuration
Expected savings
Implementation status
Realized savings
```

------------------------------------------------------------------------

# 20. Security Model

Use Managed Identity wherever possible.

Recommended separation:

``` text
Assessment Identity
-------------------
Resource inventory
Monitoring Reader
Cost Reader
Advisor Reader


Remediation Identity
--------------------
Only required Azure SQL write actions
Narrow resource scope
Approval-controlled execution
```

Avoid storing credentials in source code or configuration files.

Use Key Vault only when secrets cannot be replaced by identity-based
authentication.

------------------------------------------------------------------------

# 21. Deployment Approach

## Small or Medium Azure Estate

Recommended:

``` text
Azure Function
Storage Account
Azure SQL Database
Logic App
Azure Workbook / Power BI
Managed Identity
```

## Large Enterprise

Consider:

``` text
Azure Data Lake Storage
Azure Functions
Data Factory / Fabric / Synapse
Central SQL/Fabric Warehouse
Power BI
ServiceNow/Jira
Azure DevOps / GitHub Actions
Central FinOps subscription
```

------------------------------------------------------------------------

# 22. Recommended Rollout

Do not deploy everything at once.

## Phase 1 --- Discovery

Implement:

``` text
Resource Graph
Azure Monitor
Inventory
Basic dashboard
```

Goal:

> Where is Azure SQL money being spent?

## Phase 2 --- Cost

Add:

``` text
Cost Management Export
Resource-level cost matching
Actual/amortized reporting
```

Goal:

> How much is each workload costing?

## Phase 3 --- Recommendations

Enable:

``` text
SQL-COST-001 through SQL-COST-010
Azure Advisor integration
Savings calculations
```

Goal:

> Where can the customer reduce cost?

## Phase 4 --- Governance

Add:

``` text
Application owner
Technical review
Approval
ServiceNow/Jira
Teams/email notification
```

Goal:

> Who owns each recommendation and will implement it?

## Phase 5 --- Controlled Remediation

Add:

``` text
Automation
DevOps pipeline
Rollback
Post-change monitoring
```

Goal:

> Safely implement approved optimizations.

## Phase 6 --- Savings Realization

Compare:

``` text
Pre-change baseline
vs.
Post-change actual billing
```

Goal:

> Did the Azure bill actually decrease?

------------------------------------------------------------------------

# 23. Recommended Pilot

Start with **one non-production subscription**.

Recommended pilot steps:

1.  Deploy the read-only assessment identity.
2.  Run Azure SQL inventory.
3.  Collect at least 30 days of metrics.
4.  Import Cost Management data.
5.  Import Azure Advisor recommendations.
6.  Run SQL-COST-001 through SQL-COST-010.
7.  Produce the recommendation report.
8.  Select the top 10 opportunities.
9.  Review them manually with DBA and application teams.
10. Validate expected savings.
11. Implement a small number of approved non-production changes.
12. Measure performance after the change.
13. Measure billing after the change.
14. Tune recommendation thresholds based on the pilot.
15. Expand to additional subscriptions.

------------------------------------------------------------------------

# 24. Success Criteria

The MVP is successful when the platform can automatically produce:

``` text
Database
Current SKU
30-Day Utilization
Current Cost
Advisor Recommendation
Custom Recommendation
Recommended Target
Expected Monthly Saving
Expected Annual Saving
Confidence
Risk
Priority
Owner
Status
```

Example estate-level result:

``` text
Azure SQL Databases          214
Current Monthly Spend      $286K

Rightsizing Candidates        38
Idle Databases                11
Serverless Candidates         16
Elastic Pool Candidates       24
Reservation Candidates        31
AHB Opportunities             18
Storage Findings               7

Potential Monthly Saving     $61K
Potential Annual Saving     $732K

High Confidence              $34K
Medium Confidence            $21K
Needs Review                  $6K
```

------------------------------------------------------------------------

# 25. Repository Structure

A recommended repository structure is:

``` text
azure-sql-finops/
|
+-- infrastructure/
|   +-- bicep/
|   +-- parameters/
|
+-- collectors/
|   +-- resource_graph/
|   +-- azure_monitor/
|   +-- cost_management/
|   +-- advisor/
|
+-- recommendation-engine/
|   +-- rules/
|   +-- policy.json
|
+-- database/
|   +-- schema.sql
|
+-- dashboard/
|   +-- powerbi/
|   +-- workbook/
|
+-- remediation/
|   +-- automation/
|   +-- pipelines/
|
+-- docs/
|   +-- architecture.md
|   +-- operations.md
|   +-- security.md
|
+-- tests/
|
+-- README.md
```

------------------------------------------------------------------------

# 26. Files Created During This Design

The solution was broken into several starter packages during
development:

### Azure SQL FinOps Starter

Contains:

``` text
Azure Resource Graph inventory
PowerShell metric collection
Recommendation schema
Policy configuration
Initial recommendation engine
Architecture documentation
```

### Deployable Azure Package

Adds:

``` text
Bicep infrastructure
Managed Identity
Function/Automation foundation
Storage
Recommendation database
Logic App approval foundation
Security guidance
```

### Cost + Advisor Integration

Adds:

``` text
Cost Management export reader
Actual/amortized cost support
Azure Advisor collector
ResourceId normalization
Unified assessment dataset
```

### Recommendation Engine v1

Adds:

``` text
SQL-COST-001 through SQL-COST-010
Azure Advisor findings
Confidence
Risk
Priority
Savings calculation
Recommendation status
```

### Customer Dashboard

Adds:

``` text
Executive KPIs
Recommendation tracking
Database detail
Savings tracking
Realized savings
Recommendation status workflow
```

Together these components form the complete Azure SQL FinOps assessment
framework.

------------------------------------------------------------------------

# 27. Final End-to-End Flow

``` text
Azure Subscriptions
        |
        v
Azure Resource Graph
        |
        v
Azure SQL Inventory
        |
        +--------------------+
        |                    |
        v                    v
Azure Monitor         Cost Management
Metrics                  Export
        |                    |
        +---------+----------+
                  |
                  v
             Azure Advisor
                  |
                  v
        Unified Assessment Data
                  |
                  v
        Recommendation Engine
                  |
        +---------+---------+
        |                   |
        v                   v
Custom SQL Rules      Advisor Findings
        |                   |
        +---------+---------+
                  |
                  v
          Savings Calculation
                  |
                  v
          Customer Dashboard
                  |
                  v
          Technical Review
                  |
                  v
        Application Owner Review
                  |
                  v
               Approval
                  |
                  v
        Controlled Remediation
                  |
                  v
          Post-Change Monitoring
                  |
                  v
         Actual Billing Validation
                  |
                  v
            Realized Savings
```

------------------------------------------------------------------------

# 28. Final Recommendation

Treat this platform as a **FinOps decision-support and governance
system**, not simply a resizing script.

The most valuable capability is not detecting low CPU.

The value comes from combining:

``` text
Resource configuration
+
Performance behavior
+
Actual Azure cost
+
Microsoft Advisor findings
+
SQL-specific optimization rules
+
Application ownership
+
Change governance
+
Realized savings
```

That gives the customer a repeatable process for continuously reducing
Azure SQL cost without sacrificing application reliability.

The end result should answer five questions clearly:

1.  **What are we spending?**
2.  **Why are we spending it?**
3.  **Where can we safely reduce it?**
4.  **Who owns the action?**
5.  **Did the change actually reduce the Azure bill?**
