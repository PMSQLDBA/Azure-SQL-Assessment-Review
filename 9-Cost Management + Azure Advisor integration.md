Milestone: **Cost Management + Azure Advisor integration**.

[Download the Cost + Advisor integration package](sandbox:/mnt/data/azure-sql-finops-cost-advisor.zip)

[Cost+Advisor Integration Pacakge](https://github.com/PMSQLDBA/Azure-SQL-Assessment-Review/blob/main/azure-sql-finops-cost-advisor.zip)

This package adds:

* `cost_export_reader.py` — reads Cost Management export CSVs and aggregates cost by Azure `ResourceId`
* `advisor_collector.py` — calls Azure Advisor using the current `2025-01-01` API and collects Cost recommendations ([Microsoft Learn][1])
* `join_assessment.py` — joins **SQL inventory/metrics + cost + Advisor** into one assessment dataset
* support for **Actual vs Amortized cost**
* normalized Resource ID matching
* sample input/output data
* SQL schema for assessment snapshots
* production deployment/security notes

Cost Management Exports currently use API version `2025-03-01`, and Microsoft supports Actual Cost and Amortized Cost export datasets. ([Microsoft Learn][2])

The working pipeline is now:

```text
Azure SQL Inventory
        +
30-Day Monitor Metrics
        +
Cost Management Export
        +
Azure Advisor
        ↓
ResourceId Join
        ↓
Unified Assessment Dataset
        ↓
Recommendation Engine
```

So we now have the main **data foundation**.

### Next action

The next useful build is the **final recommendation engine v1**, where the unified dataset automatically generates:

```text
SQL-COST-001  Rightsizing
SQL-COST-002  Idle Database
SQL-COST-003  Serverless Candidate
SQL-COST-004  Elastic Pool Candidate
SQL-COST-005  Business Critical Review
SQL-COST-006  Reservation Candidate
SQL-COST-007  Azure Hybrid Benefit Review
SQL-COST-008  Storage Optimization
SQL-COST-009  Query/Performance Cost
SQL-COST-010  Cost Spike
```

and calculates:

```text
Current Cost
Target Cost
Monthly Savings
Annual Savings
Savings %
Confidence
Risk
Priority
Recommendation Source
Owner
Status
```

That is the step that turns the collected Azure data into a **customer-ready Azure SQL cost assessment**.

[1]: https://learn.microsoft.com/en-us/rest/api/advisor/recommendations/list "Recommendations - List - REST API (Azure Advisor)"
[2]: https://learn.microsoft.com/en-us/rest/api/cost-management/exports/get "Exports - Get - REST API (Azure Cost Management)"
