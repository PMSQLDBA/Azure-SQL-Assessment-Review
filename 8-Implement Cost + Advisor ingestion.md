The next action is to implement **Cost + Advisor ingestion**. That completes the inputs needed before building the dashboard.

### Step 1 — Cost Management ingestion

Configure a daily Cost Management Export:

```text
Azure Cost Management
        ↓
Daily Export
        ↓
Storage Account
        ↓
Azure Function
        ↓
Filter Azure SQL resources
        ↓
Aggregate cost by ResourceId
        ↓
Recommendation Repository
```

Cost Management supports scheduled exports to storage, which is a good fit for this automation. ([Microsoft Learn][1])

Store these fields:

```text
ResourceId
DatabaseName
Date
ActualCost
AmortizedCost
MeterCategory
MeterName
Currency
SubscriptionId
ResourceGroup
```

Then calculate:

```text
Last 7-day cost
Last 30-day cost
Previous 30-day cost
Month-to-date cost
Projected monthly cost
Cost growth %
```

### Step 2 — Import Azure Advisor

Call Advisor at subscription level and filter for:

```text
Category = Cost
```

The current Advisor `2025-01-01` API supports filtering recommendations by category, resource group, resource ID and other properties. ([Microsoft Learn][2])

Flow:

```text
Subscription
     ↓
Azure Advisor API
     ↓
Cost Recommendations
     ↓
Match ResourceId
     ↓
Azure SQL resources
     ↓
Recommendation Repository
```

Don't replace Advisor recommendations with our custom rules. Keep both:

```text
SOURCE
-----------------
CUSTOM
AZURE_ADVISOR
```

That lets the customer see:

> "Microsoft recommends this"

versus

> "Our Azure SQL assessment engine identified this."

### Step 3 — Combine the three datasets

This becomes the core assessment dataset:

```text
               SQL Inventory
                    │
                    ▼
             ┌──────────────┐
Monitor ────►│ Azure SQL DB │◄──── Cost
Metrics      └──────┬───────┘
                    │
                 Advisor
                    │
                    ▼
          Recommendation Engine
```

For every database we should now have:

```text
Database
Current SKU
vCores

30-Day Avg CPU
30-Day P95 CPU
30-Day P95 Data IO
30-Day P95 Log IO

30-Day Actual Cost
Previous 30-Day Cost
Cost Growth %

Advisor Recommendation

Custom Recommendation
Recommended SKU

Expected Monthly Saving
Expected Annual Saving

Confidence
Risk
Owner
Status
```

### Step 4 — Produce the first real customer report

For example:

```text
Customer Azure SQL Assessment
=================================

Databases                 214

Current Monthly Cost     $286K

Optimization Candidates
---------------------------------
Rightsizing                38
Idle DB                    11
Serverless                 16
Elastic Pool               24
Reservation                31
AHB                        18
Storage                     7
Advisor                    22

Potential Saving
---------------------------------
Monthly                   $61K
Annual                   $732K

High Confidence           $34K
Medium Confidence         $21K
Needs Review               $6K
```

### Step 5 — Then build the dashboard

At that point, we stop adding collectors.

The next layer becomes:

**Power BI Executive Dashboard → DBA Dashboard → Recommendation Detail → Owner Approval → Implementation → Realized Savings.**

I would specifically make the executive dashboard show **Current Spend, Identified Savings, Approved Savings, Implemented Savings and Realized Savings**. Cost Management itself is built around analysis, monitoring, optimization, alerts and exported cost data, so this model fits naturally with Azure's FinOps capabilities. ([Microsoft Learn][1])

**So our immediate coding milestone is now: `Cost Management Export Reader + Azure Advisor Collector + ResourceId Join`.** Once that works, we have all the data needed for the first complete Azure SQL cost assessment engine.

[1]: https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/overview-cost-management "Overview of Cost Management - Microsoft Cost Management | Microsoft Learn"
[2]: https://learn.microsoft.com/en-us/rest/api/advisor/recommendations/list "Recommendations - List - REST API (Azure Advisor) | Microsoft Learn"
