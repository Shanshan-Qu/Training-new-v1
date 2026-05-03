# Step 16 — Log Analytics & KQL for storage and VMs

_The "I can find any log in 30 seconds" lab._ 🔎 Phase 4 opens with the single most-used skill of your operations role: writing KQL queries against the Log Analytics workspace that ingests every signal from DSR.

> [!NOTE]
> **Trainee duration:** 120 minutes
> **Instructor EDE:** 4.0 hours (1h prep + 2h delivery + 1h Q&A buffer)
> **Lab cost:** under NZD $1 — small workspace, free-tier ingestion volume.
> **Prerequisites:** Steps 01–15 complete (this lab pulls together signals from every prior step).
> **Pairs with:** Module 4 of the DIA training plan (Observability & Reporting).

---

## 📖 Session overview

In Phase 1–3 you saw KQL queries scattered across labs — Resource Graph in Step 02, AzureDiagnostics in Step 13, Heartbeat in Step 12. This lab makes that fluency explicit. You'll create a small workspace, send VM and storage diagnostic data into it, and build the queries you'll actually run during incidents and reporting cycles. By the end you can find any signal across the DSR estate without copy-pasting somebody else's query.

**What you'll learn**
- Log Analytics workspace structure: tables, retention, ingestion rate, daily caps.
- The five tables that matter most for our role: `Heartbeat`, `AzureDiagnostics`, `AzureMetrics`, `Syslog`, `StorageBlobLogs`.
- KQL fundamentals: `where`, `project`, `summarize`, `extend`, `bin`, `join`, `render`.
- How to **save queries** as functions and pin them to a dashboard.
- How to **read existing DSR queries** in the runbooks without retyping them.
- The cost model — what makes a query expensive and how to scope it.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **Log Analytics workspace** | The container for log data. Tables are inside it. |
| **Table** | Schema-on-write store, e.g. `Heartbeat`. Each ingest creates rows. |
| **Diagnostic Setting** | The pipe from an Azure resource to a workspace. Usually one per resource. |
| **AzureDiagnostics** | The catch-all table for diagnostic categories that don't have a dedicated table. |
| **AzureMetrics** | Numeric metrics emitted by resources (CPU, IOPS, etc.). |
| **Syslog** | Standard Linux log facility — VMs running the Azure Monitor Agent forward Syslog here. |
| **Retention** | How long the table keeps data. Default 30 days; can extend per table. |
| **Ingestion cap** | A hard daily ceiling to stop runaway costs. |
| **KQL** | Kusto Query Language — read-only, Splunk-like, very fast for time-bounded queries. |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Introduction to Kusto Query Language](https://learn.microsoft.com/training/modules/intro-to-kusto/) | The core mental model — same dialect as Resource Graph and App Insights. |
| [Analyze your Azure infrastructure with Log Analytics](https://learn.microsoft.com/training/modules/analyze-infrastructure-log-data/) | Practical query patterns for ops. |
| [Write efficient KQL queries](https://learn.microsoft.com/azure/data-explorer/kusto/query/best-practices) | Cost and performance shaping — read this once. |

About **2.5 hours** of optional pre-reading.

## 🧱 Foundational primer

- **A workspace is per-region, per-subscription** in concept. DSR runs one workspace per environment (`law-anl-nzn-prd`, `law-anl-nzn-tst`, etc.).
- **Diagnostic Settings are how data gets in.** No setting = no data. Audit early.
- **AzureDiagnostics is a wide table** — different resource types put different columns. Always filter by `ResourceType` and `Category`.
- **Always scope by time first.** A query without `TimeGenerated > ago(...)` reads the whole retention window.
- **Project early.** `| project a, b, c` after a `where` cuts data volume passed downstream.
- **Functions vs saved queries** — functions are reusable templates, saved queries are one-shot.
- **Workbooks (Step 17) consume KQL** — every query you save here becomes a dashboard tile later.

## ⌨️ Activity 1 — Create a tiny lab workspace

1. Portal → **Log Analytics workspaces → + Create**.
2. RG: `rg-labs-foundations-<your-initials>`. Name: `law-lab-<initials>`. Region: Australia East.
3. Pricing tier: **Pay-as-you-go (Per GB 2018)**.
4. Daily cap: set 1 GB to be safe.
5. Create. Wait ~1 min.

## ⌨️ Activity 2 — Wire a VM into the workspace

Use the lab VM from Step 15 (or any small VM you have).

1. VM → **Insights → Enable**. Pick your lab workspace.
2. Wait ~5 min. Heartbeat starts flowing.
3. Workspace → **Logs**. Run:

```kql
Heartbeat
| where TimeGenerated > ago(15m)
| summarize lastSeen = max(TimeGenerated) by Computer, OSType
```

You should see your VM with a recent timestamp.

## ⌨️ Activity 3 — Wire a storage account into the workspace

Use the storage account from Step 05.

1. Storage account → **Diagnostic settings → + Add diagnostic setting**.
2. Categories: pick **StorageRead**, **StorageWrite**, **StorageDelete** (under blob).
3. Destination: Send to Log Analytics → your workspace.
4. Save.
5. Generate some traffic (upload/download a blob). Wait ~5 min, then run:

```kql
StorageBlobLogs
| where TimeGenerated > ago(30m)
| project TimeGenerated, OperationName, StatusText, CallerIpAddress, Uri, DurationMs
| order by TimeGenerated desc
| take 50
```

You see every API call against your storage account, with status, caller IP, and latency.

## ⌨️ Activity 4 — KQL fundamentals

Run each query and read the result before moving on.

```kql
// 1. Filter and project
Heartbeat
| where TimeGenerated > ago(1h)
| project TimeGenerated, Computer, OSType, ResourceGroup
| take 20

// 2. Summarize
Heartbeat
| where TimeGenerated > ago(1h)
| summarize beats = count() by Computer
| order by beats desc

// 3. Bin time series
Heartbeat
| where TimeGenerated > ago(1h)
| summarize beats = count() by bin(TimeGenerated, 5m), Computer
| render timechart

// 4. Extend (add computed columns)
StorageBlobLogs
| where TimeGenerated > ago(1h)
| extend latencyMs = DurationMs, sizeKB = ResponseBodySize / 1024
| project TimeGenerated, OperationName, latencyMs, sizeKB

// 5. Join (cross-table correlation)
StorageBlobLogs
| where TimeGenerated > ago(1h)
| where StatusCode >= 400
| join kind=leftouter (
    AzureMetrics
    | where TimeGenerated > ago(1h)
    | where MetricName == "Transactions"
    | summarize count() by Resource
  ) on $left.AccountName == $right.Resource
```

## ⌨️ Activity 5 — Save a query as a function

Functions let you call your query like a table.

1. In Logs view, write your favourite query.
2. **Save → Save as function**.
3. Name: `OurFailedBlobOps_v1`. Category: `anl`.
4. Save. Now you can call:

```kql
OurFailedBlobOps_v1
| where TimeGenerated > ago(1h)
```

DSR's runbooks reference saved functions like `AnlBackupHealth`, `AnlStorageThrottle` — every operator can call them without retyping.

## ⌨️ Activity 6 — The five queries you'll run weekly

```kql
// 1. VM heartbeat health
Heartbeat
| where TimeGenerated > ago(15m)
| summarize lastBeat = max(TimeGenerated) by Computer
| extend ageMin = datetime_diff('minute', now(), lastBeat)
| where ageMin > 5
| order by ageMin desc

// 2. Storage 5xx errors
StorageBlobLogs
| where TimeGenerated > ago(24h)
| where StatusCode >= 500
| summarize errors = count() by AccountName, OperationName, bin(TimeGenerated, 1h)
| render timechart

// 3. AGW backend unhealthy
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS"
| where Category == "ApplicationGatewayAccessLog"
| where TimeGenerated > ago(24h)
| where backendStatusCode_d >= 500
| summarize count() by backendPoolName_s, bin(TimeGenerated, 5m)
| render timechart

// 4. Syslog ERROR/WARNING from Rosetta tier
Syslog
| where TimeGenerated > ago(24h)
| where Computer startswith "vm-rosi"
| where SeverityLevel in ("err", "warning", "crit")
| summarize count() by Computer, SeverityLevel, bin(TimeGenerated, 1h)

// 5. Storage transaction volume by tier (cost driver)
AzureMetrics
| where TimeGenerated > ago(7d)
| where MetricName == "Transactions"
| where Resource startswith "STANLNZN"
| summarize total = sum(Total) by Resource, bin(TimeGenerated, 1d)
| render columnchart
```

These five are your daily/weekly "is the estate healthy?" check.

## ⌨️ Activity 7 — Cost-aware queries

```kql
// Show how much data each table ingested in the last 7 days
Usage
| where TimeGenerated > ago(7d)
| where IsBillable == true
| summarize GBs = sum(Quantity) / 1024 by DataType
| order by GBs desc
```

Use this to spot a runaway ingest source. Common culprits: a VM with verbose diagnostic logs, a misconfigured Diagnostic Setting sending Audit data unintentionally.

## 🦾 Now your turn!

1. Write the KQL for "show every storage account in the workspace, with its 95th-percentile blob op latency over the last 24 hours, sorted descending".
2. Save your "five weekly queries" as functions named `Anl1HeartbeatGaps`, `Anl2StorageErrors`, etc. (Use a consistent prefix.)
3. Find the **Log Analytics ingestion alerts** feature — set up an alert when daily ingestion exceeds 5 GB.
4. Read the existing DSR runbook section for "AGW backend unhealthy" — what's the saved function name? Run it for the last hour.

## ✅ Success checklist

- [ ] You can name the five most-used tables and what each holds.
- [ ] You can write a KQL query with `where`, `project`, `summarize`, `bin`, and `render`.
- [ ] You've wired a VM and a storage account into your workspace.
- [ ] You can save and reuse a function.
- [ ] You can spot a query that's expensive (no time filter, scans wide tables).
- [ ] You've **deleted** the lab workspace if it isn't needed beyond the lab.

## 📚 Self-serve refresher

- [KQL quick reference](https://learn.microsoft.com/azure/data-explorer/kql-quick-reference) — every operator on one page.
- [Diagnostic Settings reference](https://learn.microsoft.com/azure/azure-monitor/essentials/diagnostic-settings) — what categories ship to which table.

## 💰 Cost note

- Workspace itself: free.
- Per GB ingestion: ~NZD $4.30/GB after free 5 GB/month tier. Lab volume <100 MB: $0.
- Retention beyond 31 days: per-GB-month fee. Default 30 days = free.

```bash
# Cleanup (only if you don't want to keep the workspace)
az monitor log-analytics workspace delete -g rg-labs-foundations-<your-initials> -n law-lab-<initials> --yes
```

---

⬅️ **Previous:** [Step 15 — WOD container operations](step-15-wod-container-ops.md)
➡️ **Next:** [Step 17 — Azure Monitor Workbooks](step-17-workbooks.md)
