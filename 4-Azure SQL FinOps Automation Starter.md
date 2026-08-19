# Azure SQL FinOps Automation Starter

[Azure-SQL-FinOps-Starter.zip](https://github.com/PMSQLDBA/Azure-SQL-Assessment-Review/blob/main/azure-sql-finops-starter.zip)

<img width="718" height="266" alt="Finops" src="https://github.com/user-attachments/assets/beffe98a-ae31-45ff-9098-7b0f9d86f707" />

Created a reusable **Azure SQL FinOps Automation Starter Kit** that you can use as the base for customer assessments.

[Download the Azure SQL FinOps Starter Kit](sandbox:/mnt/data/azure-sql-finops-starter.zip)

It includes:

* `README.md` — how to use the solution
* `kql/sql_inventory.kql` — discovers Azure SQL databases using Azure Resource Graph
* `powershell/collect_sql_assessment.ps1` — collects inventory and Azure Monitor metrics
* `python/recommendation_engine.py` — automated cost-optimization rules engine
* `config/policy.json` — configurable P95 CPU/IO/idle thresholds
* `sql/recommendation_schema.sql` — database schema for recommendation and savings tracking
* `docs/architecture.md` — target architecture, approval flow, security, and rollout model
* `sample_metrics.csv` — example assessment input
* `sample_recommendations.csv` — sample output produced by the rules engine

The design uses Azure Resource Graph for inventory and Azure Monitor for performance telemetry. Microsoft specifically exposes SQL Database metrics such as CPU, Data IO, Log IO, sessions, workers, and storage through Azure Monitor. ([Microsoft Learn][1])

For billing, the design expects **Cost Management Exports** rather than trying to calculate the customer's real spend from Azure retail pricing. Microsoft recommends recurring Exports as the scalable way to ingest detailed cost data, and the current REST API supports creating, updating, executing, and checking export history. ([Microsoft Learn][2])

I also structured it so Azure Advisor recommendations can be brought into the same recommendation repository. The current Advisor recommendations API supports retrieving cached recommendations at subscription scope. ([Microsoft Learn][3])

The automation flow is essentially:

```text
Azure Subscriptions
        ↓
Azure Resource Graph
        ↓
SQL Inventory
        ↓
Azure Monitor Metrics
        +
Cost Management Export
        +
Azure Advisor
        ↓
Recommendation Engine
        ↓
Rightsize / Idle / Storage / Serverless /
Elastic Pool / Reservation / AHB opportunities
        ↓
Expected Monthly + Annual Savings
        ↓
Technical Review
        ↓
Application Owner Approval
        ↓
Automation / DevOps Change
        ↓
7 / 30 Day Validation
        ↓
Realized Savings
```

The next logical step is to turn this starter into a **deployable Azure solution** using Bicep or Terraform, with Managed Identity, Azure Function/Automation Account, Storage, recommendation database, Logic App approval workflow, and Power BI/Workbook reporting.

[1]: https://learn.microsoft.com/en-us/azure/azure-sql/database/resource-graph-samples "Azure Resource Graph Sample Queries - Azure SQL ..."
[2]: https://learn.microsoft.com/en-us/azure/cost-management-billing/automate/usage-details-best-practices "Cost details best practices - Microsoft Cost Management"
[3]: https://learn.microsoft.com/en-us/rest/api/advisor/recommendations "Recommendations - REST API (Azure Advisor)"

========================================================================================================================================================================
This starter kit is a reference implementation for automating Azure SQL cost assessment and savings tracking.

## What it does

1. Discovers Azure SQL databases across subscriptions using Azure Resource Graph.
2. Collects Azure Monitor metrics for a configurable lookback window.
3. Imports/joins actual billing data exported by Azure Cost Management.
4. Applies cost-optimization rules.
5. Produces recommendations with confidence, risk and savings estimates.
6. Stores recommendations for approval and realized-savings tracking.

## Recommended production flow

Azure Resource Graph
    -> Azure Monitor Metrics
    -> Cost Management Export
    -> Recommendation Engine
    -> Recommendation Repository
    -> Power BI / Workbook / Teams / Email / ServiceNow
    -> Human Approval
    -> Azure Automation / DevOps remediation
    -> 7/30 day post-change validation

## Files

- `kql/sql_inventory.kql` - SQL DB inventory query.
- `powershell/collect_sql_assessment.ps1` - inventory and metrics collection scaffold.
- `python/recommendation_engine.py` - rules engine that reads a normalized CSV and emits recommendations.
- `sql/recommendation_schema.sql` - recommendation tracking schema.
- `config/policy.json` - configurable thresholds.
- `docs/architecture.md` - implementation architecture and rollout guidance.
- `sample_metrics.csv` - sample input for the Python rules engine.

## Prerequisites

PowerShell modules:
- Az.Accounts
- Az.ResourceGraph
- Az.Monitor

Typical read permissions:
- Reader or Resource Graph Reader on target scope
- Monitoring Reader for metrics
- Cost Management Reader for cost data
- Advisor Reader for Advisor recommendations

For remediation, use a separate managed identity with least-privilege write permissions and an approval gate.

## Run the sample recommendation engine

```bash
python python/recommendation_engine.py \
  --input sample_metrics.csv \
  --policy config/policy.json \
  --output recommendations.csv
```

## Important

The included thresholds are example policy values, not Microsoft guarantees. Validate them with the customer's workload, SLA, month-end processing, maintenance windows, HA/DR needs and change-management policy before resizing production databases.
