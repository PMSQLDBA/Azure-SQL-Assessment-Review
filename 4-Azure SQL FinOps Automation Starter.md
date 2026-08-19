# Azure SQL FinOps Automation Starter

This starter kit is a reference implementation for automating Azure SQL cost assessment and savings tracking.

[Azure-SQL-FinOps-Starter.zip](https://github.com/PMSQLDBA/Azure-SQL-Assessment-Review/blob/main/azure-sql-finops-starter.zip)

<img width="718" height="266" alt="Finops" src="https://github.com/user-attachments/assets/beffe98a-ae31-45ff-9098-7b0f9d86f707" />

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
