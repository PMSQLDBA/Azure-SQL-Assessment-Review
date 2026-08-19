The next action is **not more architecture**. It is to run a small **MVP data collection test** against one Azure subscription.

### Action 1 — Validate the collector

Use the package we created and select **one Dev/Test Azure subscription** containing a few Azure SQL databases.

Run the inventory query first and confirm you get:

```text
Subscription
Resource Group
SQL Server
Database
Region
Service Tier
SKU
vCores
Elastic Pool
Resource ID
```

Then run the metric collector for **30 days** and retrieve:

```text
CPU %
Data IO %
Log IO %
Sessions %
Workers %
Storage %

Average
P95
Maximum
```

Azure Monitor's current Metrics API supports specifying the time range, interval, metric names, and aggregation, so this collection can be fully automated. ([Microsoft Learn][1])

### Action 2 — Produce the first raw assessment file

Our immediate success criterion should be a file like:

| Database  | SKU  | Avg CPU | P95 CPU | P95 Data IO | P95 Log IO | Monthly Cost |
| --------- | ---- | ------: | ------: | ----------: | ---------: | -----------: |
| OrdersDB  | GP 8 |     14% |     32% |         27% |        19% |       $1,834 |
| HRDB      | GP 4 |      3% |      9% |          7% |         5% |         $620 |
| BillingDB | GP 8 |     38% |     76% |         63% |        51% |       $2,100 |

For larger estates, Azure Monitor also provides batch/subscription-level approaches, which we can use later to reduce API calls and throttling. ([Microsoft Learn][2])

### Action 3 — Run recommendations

Once that data is coming through correctly:

```text
Inventory
    ↓
30-Day Metrics
    ↓
Actual Cost
    ↓
Recommendation Engine
    ↓
Candidate Savings
```

For example:

```text
OrdersDB
GP 8 vCore

P95 CPU       32%
P95 Data IO   27%
P95 Log IO    19%

        ↓

SQL-COST-001
Rightsizing Candidate

        ↓

Evaluate GP 4 vCore

Expected saving:
$804/month
$9,648/year

Confidence: HIGH
Risk: MEDIUM
```

One improvement we should make while testing: **discover the available metric definitions instead of hard-coding every metric name**. Azure exposes metric definitions programmatically, which makes the collector safer across SQL resource types. ([Microsoft Learn][3])

Also, Azure SQL Managed Instance has a different metric set—for example `avg_cpu_percent`, `io_bytes_read`, `io_bytes_written`, `reserved_storage_mb`, and `virtual_core_count`—so we should build a separate MI collector rather than assuming SQL Database metrics apply to MI. ([Microsoft Learn][4])

**So the immediate milestone is:**

> **One Dev/Test subscription → inventory + 30-day metrics + cost → CSV → automated recommendations.**

Once that works correctly, the next build is **Cost Management ingestion + Advisor integration + dashboard**.

[1]: https://learn.microsoft.com/en-us/rest/api/monitor/metrics/list "Metrics - List - REST API (Azure Monitor) | Microsoft Learn"
[2]: https://learn.microsoft.com/en-us/azure/azure-monitor/metrics/data-platform-metrics "Metrics in Azure Monitor - Azure Monitor | Microsoft Learn"
[3]: https://learn.microsoft.com/en-us/rest/api/sql/databases/list-metric-definitions?view=rest-sql-2014-04-01&utm_source=chatgpt.com "Databases - List Metric Definitions - REST API (Azure SQL Database) | Microsoft Learn"
[4]: https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-metrics/microsoft-sql-managedinstances-metrics?utm_source=chatgpt.com "Supported metrics - Microsoft.Sql/managedInstances - Azure Monitor | Microsoft Learn"
