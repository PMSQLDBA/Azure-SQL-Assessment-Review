Lets build this as a small **Azure SQL Cost Optimization Platform** rather than a one-time assessment. 

The goal is to automatically discover Azure SQL resources, collect utilization and billing, generate savings recommendations, track approvals, and then measure whether the customer actually saved money.

Microsoft already provides the core data sources needed for this: 
Azure Resource Graph can inventory Azure SQL resources across subscriptions; 
Azure Monitor exposes SQL Database performance metrics; 
Cost Management exports can regularly push detailed cost data into storage; and 
Azure Advisor exposes recommendations programmatically through its REST API. ([Microsoft Learn][1])

## 1. Recommended architecture

```text
                    CUSTOMER AZURE TENANT
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
 Azure Resource Graph   Azure Monitor   Cost Management
     Inventory             Metrics          Export
          │                │                │
          │                │                ▼
          │                │          Storage Account
          │                │
          └────────────┬───┴────────────────┘
                       │
                       ▼
              Data Collection Layer
          Azure Function / Automation
                       │
                       ▼
              Optimization Engine
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
   Rightsizing      FinOps Rules    Azure Advisor
        │              │              │
        └──────────────┼──────────────┘
                       ▼
             Recommendation Store
             Azure SQL / Cosmos DB
                       │
              ┌────────┴────────┐
              ▼                 ▼
       Power BI / Workbook   Alerts
                             Teams /
                             Email /
                             ServiceNow
                                  │
                                  ▼
                              Approval
                                  │
                                  ▼
                       Remediation Pipeline
                       Azure Automation /
                       Function / DevOps
                                  │
                                  ▼
                           Azure SQL Change
                                  │
                                  ▼
                          Post-change Monitor
                                  │
                                  ▼
                         Realized Savings
```

Azure Monitor is particularly important because you should not right-size using CPU alone. 

Microsoft exposes metrics including CPU, Data IO, Log IO and other database resource indicators that can be used to detect whether a database is approaching resource limits. ([Microsoft Learn][2])

---

# 2. Components I would use

| Component                  | Purpose                                             |
| -------------------------- | --------------------------------------------------- |
| Azure Resource Graph       | Find every Azure SQL DB/server across subscriptions |
| Azure Monitor Metrics API  | Collect CPU, IO, sessions, workers, storage         |
| Cost Management Export     | Daily actual/amortized billing data                 |
| Azure Advisor API          | Import Microsoft's existing recommendations         |
| Azure Function             | Main recommendation engine                          |
| Storage Account / ADLS     | Raw cost and metrics history                        |
| Azure SQL DB / Cosmos DB   | Recommendation tracking                             |
| Logic App                  | Approval / notification workflow                    |
| Power BI or Azure Workbook | FinOps dashboard                                    |
| Azure Automation / DevOps  | Approved remediation                                |
| Managed Identity           | Authentication                                      |
| Key Vault                  | Secrets if anything cannot use MI                   |

Cost Management exports can run automatically on a recurring schedule and write the resulting cost data into Azure Storage, which makes them useful as the billing feed for this solution. ([Microsoft Learn][3])

---

# 3. Step 1 — Inventory every Azure SQL database

Azure Resource Graph should be the starting point.

Example KQL:

```kusto
Resources
| where type =~ "microsoft.sql/servers/databases"
| extend
    serverName = split(id, "/")[8],
    databaseName = name,
    skuName = tostring(sku.name),
    skuTier = tostring(sku.tier),
    skuCapacity = tostring(sku.capacity),
    location = location
| project
    subscriptionId,
    resourceGroup,
    serverName,
    databaseName,
    location,
    skuName,
    skuTier,
    skuCapacity,
    id
| order by subscriptionId, serverName, databaseName
```

Microsoft publishes Azure SQL-specific Resource Graph examples, so ARG is a good fit for the inventory layer. ([Microsoft Learn][1])

Our inventory table could contain:

```text
Subscription
Resource Group
SQL Server
Database
Region
Environment
Application
Owner
Service Tier
SKU
vCores
Max Storage
Elastic Pool
Zone Redundancy
HA configuration
Creation Date
Tags
```

---

# 4. Step 2 — Collect utilization

For every database, collect at least:

```text
CPU %
Data IO %
Log IO %
Sessions %
Workers %
Storage %
Connections
Deadlocks
DTU % if DTU model
```

I would store:

```text
Average
Maximum
P50
P90
P95
P99
```

for:

```text
7 days
14 days
30 days
60 days
90 days
```

Example data:

| DB        | SKU        | Avg CPU | P95 CPU | P95 Data IO | P95 Log IO |
| --------- | ---------- | ------: | ------: | ----------: | ---------: |
| OrdersDB  | GP 8 vCore |     13% |     34% |         27% |        21% |
| BillingDB | GP 8 vCore |     39% |     78% |         61% |        49% |
| HRDB      | GP 4 vCore |      3% |      8% |          6% |         5% |

---

# 5. Recommendation engine

This is where your IP/value sits.

We would implement recommendations using rule IDs.

For example:

```text
SQL-COST-001
SQL-COST-002
SQL-COST-003
...
```

### SQL-COST-001 — Underutilized compute

```text
IF

P95_CPU < 40%
AND
P95_DataIO < 40%
AND
P95_LogIO < 40%
AND
ObservationPeriod >= 30 days

THEN

Recommend testing next smaller compute SKU
```

Example:

```text
Current:
General Purpose 8 vCore

P95 CPU:
31%

P95 Data IO:
22%

P95 Log IO:
19%

Recommendation:
Evaluate General Purpose 4 vCore
```

Not:

```text
Automatically change to 4 vCore
```

The distinction matters.

---

# 6. Add safety headroom

We would use 3 classifications.

### Strong candidate

```text
P95 CPU < 30%
P95 IO < 30%
```

Confidence:

```text
HIGH
```

### Review candidate

```text
P95 CPU 30–50%
P95 IO 30–50%
```

Confidence:

```text
MEDIUM
```

### Do not downsize

```text
P95 CPU > 60%
OR
P95 IO > 60%
```

Confidence:

```text
LOW
```

These percentages should be treated as **your policy thresholds**, not Microsoft guarantees.

For critical systems, we would make them more conservative.

---

# 7. SQL-COST-002 — Idle database detection

Example rule:

```text
Average CPU < 5%
AND
P95 CPU < 10%
AND
very low connection count
AND
30 days observation
```

Recommendation:

```text
Possible unused / idle database.

Validate:
Application dependency
Backup requirements
Compliance requirements
Business owner
```

Possible action:

```text
Retire
Consolidate
Serverless
Lower SKU
```

---

# 8. SQL-COST-003 — Serverless candidate

This is particularly useful for:

```text
Development
QA
Sandbox
Training
Intermittent workloads
```

Look for:

```text
Long periods with no connections
Very low overnight utilization
Short daytime workload
Large idle windows
```

Azure SQL Database serverless supports automatic compute scaling and, depending on configuration/workload, auto-pause behavior, so intermittent workloads can be good candidates. 
([Microsoft Learn][4])

Our recommendation could be:

```text
Database:
DevCustomerPortal

Provisioned runtime:
24 hours/day

Active workload:
3.8 hours/day

Recommendation:
Evaluate Azure SQL Serverless

Reason:
Database is idle approximately 80% of the day.
```

---

# 9. SQL-COST-004 — Elastic Pool opportunity

Imagine the customer has:

```text
DB-A peak = 8 AM
DB-B peak = 11 AM
DB-C peak = 4 PM
DB-D peak = 8 PM
```

They don't peak simultaneously.

Instead of:

```text
4 dedicated databases
4 separate compute allocations
```

evaluate:

```text
Elastic Pool
      │
 ┌────┼────┬────┐
 DB-A DB-B DB-C DB-D
```

Your engine should detect:

```text
Same server/region
Multiple databases
Low average utilization
Different peak windows
```

and flag them as:

```text
Elastic Pool Candidate
```

---

# 10. SQL-COST-005 — Business Critical → General Purpose

Detect databases using:

```text
Business Critical
```

Then check whether the application actually requires capabilities associated with that tier.

Don't automate this change directly.

Generate:

```text
Architecture Review Required
```

Example:

```text
Current Tier:
Business Critical

Monthly Cost:
$4,800

Potential GP cost:
$2,100

Potential Savings:
$2,700/month

Recommendation:
Validate whether BC-specific performance/
availability requirements are still required.
```

---

# 11. SQL-COST-006 — Reservation opportunity

Look for databases that:

```text
Run 24x7
Have stable sizing
Have predictable workload
Are expected to exist long term
```

Flag them as:

```text
Reservation Candidate
```

Azure Advisor can already provide cost-related recommendations, so rather than replace it, We would ingest Advisor findings into the same recommendation store and combine them with your SQL-specific logic.

Advisor recommendations are available programmatically through Microsoft's recommendations API. ([Microsoft Learn][5])

---

# 12. SQL-COST-007 — Azure Hybrid Benefit

Check whether:

```text
SQL licenses available?
Software Assurance?
Eligible subscriptions?
AHB already enabled?
```

Then surface:

```text
Potential Azure Hybrid Benefit opportunity
```

This normally requires the customer's Microsoft licensing/FinOps team to validate eligibility.

---

# 13. SQL-COST-008 — Storage optimization

Track:

```text
Current storage
30-day growth
90-day growth
Growth %
Backup size
Retention
```

Example:

```text
Database:
ReportingDB

Current storage:
2.4 TB

90 days ago:
1.5 TB

Growth:
60%

Projected 12-month storage:
6 TB
```

Then investigate:

```text
Large tables
Unused indexes
Duplicate indexes
Historical data
Audit data
Staging tables
Retention policy
```

---

# 14. Query optimization should also feed FinOps

This is often missed.

Suppose:

```text
Top 5 queries consume 65% CPU.
```

You may initially think:

```text
Need 16 vCores.
```

But after fixing queries:

```text
CPU drops 50%.
```

Now:

```text
8 vCores may be enough.
```

Microsoft also notes that high CPU or IO can indicate either insufficient resources **or queries that should be optimized**, which is exactly why I would include query tuning in the FinOps assessment. ([Microsoft Learn][4])

So your engine should have another category:

```text
Performance-driven Cost Optimization
```

---

# 15. Cost engine

For every recommendation calculate:

```text
Current Cost
Target Cost
Monthly Savings
Annual Savings
Savings %
```

Formula:

```text
Monthly Saving =
Current Monthly Cost
-
Projected Monthly Cost
```

Example:

```text
Current Cost     $2,400/month
Target Cost      $1,450/month
--------------------------------
Monthly Saving     $950
Annual Saving   $11,400
Saving %          39.6%
```

---

# 16. Don't use retail pricing as actual spend

Use:

```text
Cost Management actual/amortized data
```

as the primary financial source.

That captures the customer's real environment better than simply doing:

```text
Azure public list price × resource
```

Cost Management APIs support querying and analyzing cost and usage information, while recurring exports are Microsoft's recommended mechanism for retrieving larger unaggregated cost datasets regularly. ([Microsoft Learn][6])

---

# 17. Recommendation table

I would create something like:

```sql
CREATE TABLE SqlCostRecommendations
(
    RecommendationId       varchar(50),
    SubscriptionId         varchar(100),
    ResourceId             varchar(1000),
    DatabaseName           varchar(256),

    CurrentSku             varchar(100),
    RecommendedSku         varchar(100),

    AvgCpu                 decimal(5,2),
    P95Cpu                 decimal(5,2),
    P95DataIo              decimal(5,2),
    P95LogIo               decimal(5,2),

    CurrentMonthlyCost     decimal(18,2),
    ProjectedMonthlyCost   decimal(18,2),
    MonthlySaving          decimal(18,2),
    AnnualSaving           decimal(18,2),

    RecommendationType     varchar(100),
    Confidence             varchar(20),
    Risk                   varchar(20),

    Status                 varchar(30),
    Owner                   varchar(200),

    IdentifiedDate         datetime2,
    ApprovedDate           datetime2,
    ImplementedDate        datetime2,

    ActualMonthlySaving    decimal(18,2)
);
```

---

# 18. Status workflow

This is important.

Don't just have:

```text
Recommendation = Open
```

Use:

```text
Identified
    ↓
Technical Review
    ↓
Owner Review
    ↓
Approved
    ↓
Scheduled
    ↓
Implemented
    ↓
Validation
    ↓
Savings Confirmed
```

Or:

```text
Rejected
Deferred
Exception
```

---

# 19. Example recommendation record

```json
{
  "database": "CustomerOrdersDB",
  "currentSku": "GP_Gen5_8",
  "recommendedSku": "GP_Gen5_4",

  "observationDays": 30,

  "avgCpu": 13.4,
  "p95Cpu": 31.8,
  "p95DataIo": 27.2,
  "p95LogIo": 19.1,

  "currentMonthlyCost": 1834,
  "projectedMonthlyCost": 1030,

  "monthlySaving": 804,
  "annualSaving": 9648,

  "recommendation": "RIGHTSIZE",
  "confidence": "HIGH",
  "risk": "LOW",

  "approvalRequired": true
}
```

---

# 20. PowerShell collection example

For example, your collection runbook could use Resource Graph:

```powershell
$Query = @"
Resources
| where type =~ 'microsoft.sql/servers/databases'
| where name !~ 'master'
| project
    subscriptionId,
    resourceGroup,
    databaseName=name,
    location,
    skuName=sku.name,
    skuTier=sku.tier,
    resourceId=id
"@

$databases = Search-AzGraph `
    -Query $Query `
    -First 5000

foreach ($db in $databases) {

    Write-Output "Processing $($db.databaseName)"

    # Call Azure Monitor Metrics API
    # Retrieve CPU, Data IO, Log IO etc.
    # Calculate P95
    # Read Cost Management data
    # Run FinOps recommendation rules
    # Save recommendation
}
```

---

# 21. Simplified recommendation logic

Pseudo-code:

```python
if (
    p95_cpu < 30
    and p95_data_io < 30
    and p95_log_io < 30
    and observation_days >= 30
):
    recommendation = "Strong rightsizing candidate"
    confidence = "HIGH"

elif (
    p95_cpu < 50
    and p95_data_io < 50
    and p95_log_io < 50
):
    recommendation = "Review for rightsizing"
    confidence = "MEDIUM"

else:
    recommendation = "No rightsizing recommendation"
```

Then run additional checks:

```python
check_serverless()
check_elastic_pool()
check_reservation()
check_hybrid_benefit()
check_storage_growth()
check_idle_database()
check_business_critical()
check_query_efficiency()
```

---

# 22. Power BI dashboard

I would make the first page executive-level:

```text
╔═════════════════════════════════════════════════╗
║          AZURE SQL COST OPTIMIZATION            ║
╠═════════════════════════════════════════════════╣
║ Current Spend       $286K/month                 ║
║                                                 ║
║ Identified Savings   $74K/month                 ║
║ Approved Savings     $51K/month                 ║
║ Implemented Savings  $43K/month                 ║
║ Realized Savings     $39K/month                 ║
╚═════════════════════════════════════════════════╝
```

Then:

```text
Savings by Category

Rightsizing            $21K
Reserved Capacity      $10K
Elastic Pool            $6K
Serverless              $4K
Storage                 $3K
AHB                      $2K
Query Optimization      $2K
```

And another page:

```text
Top 20 Cost Optimization Opportunities
```

---

# 23. Track realized savings

This is possibly the most important FinOps feature.

Before:

```text
OrdersDB
$1,850/month
```

Recommendation:

```text
Reduce 8 → 4 vCore
```

Expected:

```text
$1,050/month
```

After implementation:

```text
Actual bill
$1,110/month
```

Therefore:

```text
Expected Saving
$800

Actual Saving
$740
```

Dashboard:

```text
Savings Realization:

Identified     $800
Approved       $800
Implemented    $800
Realized       $740
```

Now the customer can actually prove ROI.

---

# 24. Production safety

For production, we would recommend:

```text
Detection
    ↓
Recommendation
    ↓
Human Approval
    ↓
Pre-change validation
    ↓
Resize
    ↓
24-hour monitoring
    ↓
7-day monitoring
    ↓
30-day cost validation
```

Do **not** start with:

```text
Low CPU
   ↓
Automatically resize production DB
```

A database could have low monthly averages but a critical month-end or overnight processing window.

---

# 25. Rollback automation

Every approved resize should contain:

```text
Original configuration
Target configuration
Change timestamp
Owner
Change ticket
Rollback SKU
```

Example:

```text
Original:
GP 8 vCore

Target:
GP 4 vCore

Rollback:
GP 8 vCore
```

If:

```text
CPU P95 > 85%
OR
IO P95 > 85%
OR
application latency breaches threshold
```

during the validation window:

```text
Alert owner
+
Recommend rollback
```

We would still keep automatic rollback optional depending on the customer's change-control policy.

---

# 26. Suggested implementation phases

We wouldn't build everything at once.

### Phase 1 — Discovery

Build:

```text
Resource Graph inventory
Azure Monitor utilization
Cost Management data
Power BI dashboard
```

Outcome:

> "Where is the Azure SQL money going?"

---

### Phase 2 — Recommendations

Add:

```text
Rightsizing
Idle DB detection
Serverless candidates
Elastic Pool candidates
Storage opportunities
Advisor recommendations
```

Outcome:

> "Where can we save money?"

---

### Phase 3 — FinOps workflow

Add:

```text
Owner
Approval
Jira/ServiceNow ticket
Teams/email alert
Expected savings
Implementation status
```

Outcome:

> "Who will actually execute the saving?"

---

### Phase 4 — Remediation

Add:

```text
Azure Automation
Azure Functions
DevOps pipeline
Rollback
Post-change monitoring
```

Outcome:

> "Automate approved changes."

---

### Phase 5 — Savings realization

Compare:

```text
Baseline Cost

versus

Post-change Actual Cost
```

Outcome:

> "Did the customer's Azure bill actually decrease?"

---

## My preferred final design

For a reusable customer solution, I would use:

```text
Azure Resource Graph
        +
Azure Monitor
        +
Cost Management Export
        +
Azure Advisor API
        ↓
Azure Function
Cost Optimization Engine
        ↓
Azure SQL Repository
        ↓
Power BI
        +
Logic App
        ↓
Approval
        ↓
Azure Automation / DevOps
        ↓
30-Day Savings Validation
```

This keeps the solution relatively simple while still giving you **inventory + telemetry + actual billing + automated recommendations + governance + realized-savings tracking**. 

Azure Resource Graph, Monitor, Cost Management exports, and Advisor all have supported programmatic interfaces, so this architecture can be automated instead of relying on manual portal checks. ([Microsoft Learn][1])

For your Azure SQL Assessment offering, I would make **rightsizing + idle-resource detection + elastic-pool/serverless candidates + reservation/AHB opportunities + realized-savings tracking** the first version. 

Those areas will make the customer-facing report much stronger than a basic Azure Advisor export.

[1]: https://learn.microsoft.com/en-us/azure/azure-sql/database/resource-graph-samples "Azure Resource Graph Sample Queries - Azure SQL ..."
[2]: https://learn.microsoft.com/en-us/azure/azure-sql/database/monitoring-metrics-alerts "Monitoring Azure SQL Database with metrics and alerts"
[3]: https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/tutorial-improved-exports "Tutorial - Create and manage Cost Management exports"
[4]: https://learn.microsoft.com/en-us/azure/azure-sql/database/monitor-tune-overview "Monitoring and performance tuning - Azure SQL Database ..."
[5]: https://learn.microsoft.com/en-us/rest/api/advisor/recommendations "Recommendations - REST API (Azure Advisor)"
[6]: https://learn.microsoft.com/en-us/rest/api/cost-management/ "Microsoft Cost Management REST APIs"
