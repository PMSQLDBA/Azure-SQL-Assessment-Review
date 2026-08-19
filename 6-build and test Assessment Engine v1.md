The **next action should be to build and test Assessment Engine v1 against one non-production Azure subscription**. Don’t start with automatic remediation yet.

### Recommended next steps

1. **Deploy the starter infrastructure**

   * Function App / Automation
   * Managed Identity
   * Storage
   * Recommendation database
   * Read-only RBAC

2. **Connect one pilot subscription**

   * Start with Dev/Test.
   * Inventory all Azure SQL Databases.
   * Confirm SKU, tier, vCores, elastic-pool membership, region, tags, and owner.

3. **Collect 30 days of performance**
   Capture CPU, Data IO, Log IO, storage, workers and sessions. Microsoft recommends monitoring CPU and IO relative to the selected Azure SQL performance level; importantly, high utilization can also indicate query problems rather than simply insufficient compute. ([Microsoft Learn][1])

4. **Connect actual billing**
   Configure a Cost Management Export into the Storage Account. Microsoft specifically recommends exports for recurring cost-data retrieval and larger datasets. ([Microsoft Learn][2])

5. **Bring in Azure Advisor**
   Import Advisor findings into the same recommendation repository instead of duplicating Microsoft's recommendations. Advisor analyzes configuration and usage telemetry for cost, performance, reliability, and other optimization opportunities. ([Microsoft Learn][3])

6. **Run the first four custom rules**

   * `SQL-COST-001` — Rightsizing
   * `SQL-COST-002` — Idle database
   * `SQL-COST-003` — Serverless candidate
   * `SQL-COST-004` — Elastic Pool candidate

The first pilot output should look like:

```text
Customer SQL Estate
--------------------------------
Azure SQL DBs              186
Monthly SQL Spend      $142,500

Rightsize Candidates        31
Idle DBs                     9
Serverless Candidates       14
Elastic Pool Candidates     22

Potential Savings       $28,400/month
Potential Annual        $340,800/year

High Confidence         $17,200/month
Medium Confidence        $8,900/month
Needs Review             $2,300/month
```

Then manually validate the **top 10 savings opportunities** with the DBA/application teams. This is important because Microsoft also cautions that optimal metric thresholds vary by workload. ([Microsoft Learn][1])

Only after those recommendations prove accurate should we move to:

**v2: Dashboard → owner approval → change ticket → automated resize → rollback → 30-day realized-savings measurement.**

So the immediate technical target is very specific: **get one customer/pilot subscription producing a trustworthy CSV/database table containing `Database → Current SKU → 30-day utilization → Actual Cost → Recommendation → Target SKU → Expected Saving → Confidence → Risk`.**

That gives us a working MVP before adding more automation.

[1]: https://learn.microsoft.com/en-us/azure/azure-sql/database/monitoring-metrics-alerts "Monitoring Azure SQL Database with metrics and alerts - Azure SQL Database | Microsoft Learn"
[2]: https://learn.microsoft.com/en-us/azure/azure-sql/database/cost-management "Plan and Manage Costs - Azure SQL Database | Microsoft Learn"
[3]: https://learn.microsoft.com/en-us/azure/advisor/advisor-overview "Introduction to Azure Advisor - Azure Advisor | Microsoft Learn"
