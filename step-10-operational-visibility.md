# Step 10 — Operational Visibility & Alerts (Azure Monitor + Backup Center + Defender)

> [!IMPORTANT]
> **Operational Visibility consolidates three topics into one session:**
> - **Azure Monitor** (Logs / Metrics / Alerts / KQL) — trimmed to core content the team will actually run
> - **Backup Center read-only ops** — folded in as a section, pending ownership confirmation (DP vs Service Reliability)
> - **Defender for Storage awareness** — folded in as a short awareness slide
>
> Target duration: shorter than the original 180-min Monitor session. Drop KQL deep-dives the team already uses elsewhere.

_The "I can find any log in 30 seconds AND get paged when things break" lab._ 🔎🔔 Phase 4 opens with the single most-used skill of your operations role: querying the Log Analytics workspace, reading platform metrics, and configuring alerts that page you when DSR misbehaves.

> [!NOTE]
> **Trainee duration:** 180 minutes
> **Lab cost:** under NZD $1 — small workspace, free-tier ingestion volume, alerts are free for the first few rules.
> **Prerequisites:** Steps 00–09 complete (this lab pulls together signals from every prior step).
> **Pairs with:** Module 4 of the DIA training plan (Observability & Reporting).

---

## 📖 Session overview

In Phase 1–3 you saw KQL queries scattered across labs — Resource Graph in Step 01, AzureDiagnostics in Step 09. This lab makes that fluency explicit, then builds on top of it the second half of an operator's day: **metrics and alerts**. You'll create a small workspace, send VM and storage signals into it, build the queries you'll run during incidents and reporting cycles, then wire those signals into **metric alerts**, **log alerts**, and **action groups** that page the on-call.

By the end you can both find any signal across the DSR estate AND be notified automatically when one of them breaches.

**What you'll learn**
- Log Analytics workspace structure: tables, retention, ingestion rate, daily caps.
- The five tables that matter most for our role: `Heartbeat`, `AzureDiagnostics`, `AzureMetrics`, `Syslog`, `StorageBlobLogs`.
- KQL fundamentals: `where`, `project`, `summarize`, `extend`, `bin`, `join`, `render`.
- The **Azure Monitor metrics** blade — platform metrics vs custom metrics, splitting, multi-resource charts.
- **Metric alerts** vs **log (scheduled-query) alerts** — when to use which.
- **Action groups** — email, SMS, Teams webhook, ITSM ticket.
- How to pin saved queries and metric charts to dashboards (Step 11 builds on this).
- The cost model — what makes a query expensive, what alerts cost.

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
| **Platform metric** | A pre-aggregated numeric series emitted free by every Azure resource (CPU%, Transactions, Latency). 1-minute granularity, 93-day retention. |
| **Metric alert** | An alert that fires when a platform metric breaches a threshold. Cheap (~NZD $0.20/month per signal), evaluates every minute. |
| **Log (scheduled-query) alert** | An alert that runs a KQL query on a schedule and fires on the result. Powerful but slower (5-min minimum) and costs per evaluation. |
| **Action group** | A reusable bundle of "what to do when an alert fires" — emails, SMS, Teams webhook, runbook, ITSM ticket. |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Get started with Log Analytics queries](https://learn.microsoft.com/azure/azure-monitor/logs/get-started-queries) | The core mental model — same dialect as Resource Graph and App Insights. |
| [Analyze monitoring data with KQL (path)](https://learn.microsoft.com/training/paths/analyze-monitoring-data-with-kql/) | Practical query patterns for ops. |
| [Write efficient KQL queries](https://learn.microsoft.com/azure/data-explorer/kusto/query/best-practices) | Cost and performance shaping — read this once. |
| [Improve incident response with alerts on Azure](https://learn.microsoft.com/training/modules/incident-response-with-alerting-on-azure/) | Metric vs log alerts, action groups, alert processing rules. |
| [Analyze monitoring data with KQL (path)](https://learn.microsoft.com/training/paths/analyze-monitoring-data-with-kql/) | The metrics blade — splitting, multi-resource, pinning. |

About **4 hours** of optional pre-reading.

## 🧱 Foundational primer

- **A workspace is per-region, per-subscription** in concept. DSR runs one workspace per environment (`law-anl-nzn-prd`, `law-anl-nzn-tst`, etc.).
- **Diagnostic Settings are how data gets in.** No setting = no data. Audit early.
- **AzureDiagnostics is a wide table** — different resource types put different columns. Always filter by `ResourceType` and `Category`.
- **Always scope by time first.** A query without `TimeGenerated > ago(...)` reads the whole retention window.
- **Project early.** `| project a, b, c` after a `where` cuts data volume passed downstream.
- **Functions vs saved queries** — functions are reusable templates, saved queries are one-shot.
- **Workbooks (Step 11) consume KQL** — every query you save here becomes a dashboard tile later.

## ⌨️ Activity 1 — Create a tiny lab workspace

1. Portal → **Log Analytics workspaces → + Create**.
2. RG: `rg-labs-foundations-<your-initials>`. Name: `law-lab-<initials>`. Region: Australia East.
3. Pricing tier: **Pay-as-you-go (Per GB 2018)**.
4. Daily cap: set 1 GB to be safe.
5. Create. Wait ~1 min.

## ⌨️ Activity 2 — Wire a VM into the workspace

Use the lab VM from Step 09 (or any small VM you have).

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

## ⌨️ Activity 8 — The Azure Monitor metrics blade

Logs and metrics are two different pipelines. **Metrics** are pre-aggregated, free, 1-minute granularity, 93-day retention — perfect for fast charts and cheap alerting. You don't need a Diagnostic Setting; every Azure resource emits platform metrics by default.

1. Portal → **Monitor → Metrics**.
2. **Scope:** pick the storage account from Step 05.
3. **Metric namespace:** *Account*. **Metric:** `Transactions`. **Aggregation:** Sum.
4. The chart appears. Change time range to **Last 24 hours**.
5. Click **Apply splitting → API name** — now you see Read/Write/Delete broken out.
6. Click **+ Add metric → Availability** (Avg). You now have a two-line chart.
7. Click **Save to dashboard → Pin to dashboard**. You'll reuse this in Step 11 (Workbooks).

Repeat for a VM:

1. Scope: your lab VM. Metric: `Percentage CPU` (Avg). Splitting: none.
2. Add a second chart for `Available Memory Bytes`.
3. Add `Disk Read Operations/Sec` + `Disk Write Operations/Sec` as a third chart.

> [!TIP]
> The metrics blade has a **Multi-resource** mode — pick all `STANLNZN*` accounts at once and chart Transactions across all of them on one canvas. This is exactly the view you want during a "is it just one account or all of them?" incident.

## ⌨️ Activity 9 — Create a metric alert (storage availability)

Goal: page on-call when storage availability drops below 99% for 5 consecutive minutes.

1. Portal → **Monitor → Alerts → + Create → Alert rule**.
2. **Scope:** your lab storage account.
3. **Condition → See all signals → Availability** (Platform metric).
4. Threshold: **Static**, **Less than 99**, aggregation **Average**, granularity **1 minute**, evaluation frequency **1 minute**.
5. **Actions:** click **+ Create action group** (next activity creates one — circle back here after).
6. **Details:**
   - Severity: **2 — Warning**
   - Name: `alert-stg-availability-lab`
   - Description: "Storage availability < 99% over 5 min"
7. Save.

Cost: ~NZD $0.20/month per signal monitored. DSR runs ~10 of these in production.

## ⌨️ Activity 10 — Create an action group

Action groups define **what happens** when an alert fires. They're reusable across alert rules.

1. Portal → **Monitor → Alerts → Action groups → + Create**.
2. RG: `rg-labs-foundations-<initials>`. Name: `ag-lab-oncall-<initials>`. Display name (12 chars max): `LabOnCall`.
3. **Notifications tab:**
   - Email: your address. Name: `me`.
   - (Optional) SMS: your mobile. Country code 64 for NZ.
4. **Actions tab (optional but real-world):**
   - **Webhook → Teams incoming webhook URL** (if you have one for your team channel).
   - **ITSM** if DIA uses ServiceNow / Jira to receive alerts.
   - **Automation runbook** to auto-remediate (e.g. restart a VM).
5. Create. Test it: **Test action group → Email** → confirm you receive the test mail.

In production DSR runs at least three action groups:
- `ag-anl-prd-critical` — pages on-call (SMS + Teams + ITSM ticket).
- `ag-anl-prd-warning` — Teams + email only.
- `ag-anl-prd-cost` — email to FinOps mailbox for budget alerts.

Now go back to your alert rule from Activity 9 and attach `LabOnCall`.

## ⌨️ Activity 11 — Create a log (scheduled-query) alert

Use this when the condition is too complex for a single metric — e.g. "more than 50 5xx storage errors against any single account in the last 15 min".

1. Workspace → **Logs**. Paste:

```kql
StorageBlobLogs
| where TimeGenerated > ago(15m)
| where StatusCode >= 500
| summarize errors = count() by AccountName
| where errors > 50
```

2. Run it once to confirm syntax.
3. **+ New alert rule** (top of the query window).
4. **Condition:**
   - Measurement: **Number of results**.
   - Operator: **Greater than 0**.
   - Frequency: 5 min. Lookback: 15 min.
5. **Actions:** attach `LabOnCall`.
6. **Details:** Severity 2, name `alert-stg-5xx-burst-lab`.
7. Save.

Cost: ~NZD $1.50/month per rule (5-min frequency). Production DSR runs ~6 of these.

## ⌨️ Activity 12 — Activity Log alerts (audit)

Different beast: alerts on **control-plane events** like "someone deleted a storage account".

1. Portal → **Monitor → Alerts → + Create → Alert rule**.
2. **Scope:** subscription level.
3. **Condition → See all signals → Activity Log → Administrative → Delete Storage Account**.
4. Action: `LabOnCall`. Severity: **0 — Critical**.
5. Save.

This is a free signal — no metric, no log query — but covers the worst kinds of "who deleted prod?" events.

## ⌨️ Activity 13 — Inspect alert state and history

1. Portal → **Monitor → Alerts → Alerts (fired)** — see currently firing.
2. **Alert rules** — see all rules across the sub.
3. Click any rule → **History** — past fires, ack'd state, resolved time.
4. **Action groups → your group → Action history** — was the email/SMS actually delivered?

> [!TIP]
> When triaging an incident, always check **Alert history** first to see whether monitoring caught it. If a real outage didn't fire an alert, you have a monitoring gap to fix afterwards.

## 🦾 Now your turn!

1. Write the KQL for "show every storage account in the workspace, with its 95th-percentile blob op latency over the last 24 hours, sorted descending".
2. Save your "five weekly queries" as functions named `Anl1HeartbeatGaps`, `Anl2StorageErrors`, etc. (Use a consistent prefix.)
3. Find the **Log Analytics ingestion alerts** feature — set up an alert when daily ingestion exceeds 5 GB.
4. Read the existing DSR runbook section for "AGW backend unhealthy" — what's the saved function name? Run it for the last hour.
5. **Build a metric alert** for "VM CPU > 90% for 10 minutes" on your lab VM, using `LabOnCall`.
6. **Build a log alert** for "any Heartbeat gap > 10 minutes" on any VM in the workspace.
7. Trigger one of your alerts on purpose (e.g. stress the VM CPU with `stress-ng`) and confirm you receive the notification end-to-end.

## ✅ Success checklist

- [ ] You can name the five most-used tables and what each holds.
- [ ] You can write a KQL query with `where`, `project`, `summarize`, `bin`, and `render`.
- [ ] You've wired a VM and a storage account into your workspace.
- [ ] You can save and reuse a function.
- [ ] You can spot a query that's expensive (no time filter, scans wide tables).
- [ ] You can read a platform metric chart with splitting and multi-resource scope.
- [ ] You've created **at least one metric alert**, **one log alert**, and **one Activity Log alert**.
- [ ] You've created an **action group** and tested it (received a real email/SMS).
- [ ] You can explain when to use a metric alert vs a log alert.
- [ ] You've **deleted** the lab workspace and alert rules if they aren't needed beyond the lab.

## 📚 Self-serve refresher

- [KQL quick reference](https://learn.microsoft.com/azure/data-explorer/kql-quick-reference) — every operator on one page.
- [Diagnostic Settings reference](https://learn.microsoft.com/azure/azure-monitor/essentials/diagnostic-settings) — what categories ship to which table.
- [Azure Monitor alerts overview](https://learn.microsoft.com/azure/azure-monitor/alerts/alerts-overview) — metric vs log vs activity log, action groups, alert processing rules.
- [Supported metrics with Azure Monitor](https://learn.microsoft.com/azure/azure-monitor/reference/supported-metrics/metrics-index) — every platform metric, by resource type.
- [Azure Monitor pricing](https://azure.microsoft.com/en-us/pricing/details/monitor/) — authoritative source for the numbers in the cost table below.
- [Log Analytics cost calculator + table tier guide](https://learn.microsoft.com/azure/azure-monitor/logs/cost-logs) — when to use Analytics vs Basic vs Auxiliary logs.

## 💰 Cost note — Azure Monitor (LAW + Logs + Alerts)

Azure Monitor charges across **four** independent dimensions. Most surprises in the bill come from #2 (ingestion) and #4 (log alerts).

### 1. Log Analytics workspace (LAW) — three ingestion plans

Azure Monitor Logs offers **three table plans** that change ingestion price, query price, retention behaviour and feature support. Choosing the right plan per table is the single biggest cost lever.

| Dimension | **Analytics** (default) | **Basic Logs** | **Auxiliary Logs** (preview) |
|---|---|---|---|
| **Ingestion price** (Australia East, NZD, indicative) | ~**$4.30 / GB** | ~**$0.85 / GB** (≈ 5× cheaper) | ~**$0.25 / GB** (≈ 17× cheaper) |
| **Query price** | Free (included) | ~**$0.0075 / GB scanned** | ~**$0.0075 / GB scanned** |
| **Interactive retention** | Up to **31 days free**, then per-GB | **30 days** (fixed) | **30 days** (fixed) |
| **Long-term retention** | Add archive (up to 12 yrs) | Add archive (up to 12 yrs) | Add archive (up to 12 yrs) |
| **Query language** | Full KQL | KQL **subset** (no joins, no `make-series`) | KQL subset, simple filter/project |
| **Alerts** | ✅ Log alerts supported | ❌ Not supported | ❌ Not supported |
| **Workbooks / dashboards** | ✅ | ⚠️ Limited | ⚠️ Limited |
| **Best for** | Tables you query, alert on, or join | High-volume verbose tables you rarely read (storage `StorageBlobLogs`, AGW access logs) | Compliance/audit logs you keep "just in case" |

> [!IMPORTANT]
> A table's plan can be changed in-place: **Log Analytics workspace → Tables → … → Manage table → Table plan**. Plan change applies to *future* ingested data only.

Other LAW pricing notes:

- **Workspace itself is free** — you only pay for ingestion + retention.
- **Free tier:** the first **5 GB / workspace / month** of Analytics ingestion is free (enough for the whole lab).
- **Commitment tiers** (100 / 200 / 500 / 1 000 / 2 000 / 5 000 GB per day) give 15–30% off Analytics ingestion. Worth considering once your daily volume is stable.

### 2. Log retention (the silent line item)

| Tier | Free retention | Beyond | Cost beyond free |
|---|---|---|---|
| Analytics Logs (default) | **31 days free** | Per GB-month | ~NZD $0.18 / GB / month |
| Basic / Auxiliary Logs | 30 days fixed (no extension) | n/a | n/a — move to archive |
| Sentinel-enabled workspace | **90 days free** | Per GB-month | ~NZD $0.18 / GB / month |
| Archive tier | n/a | 1 day to 12 years | ~NZD $0.04 / GB / month + per-GB restore fee |

### 3. Metrics

| Item | Price | Notes |
|---|---|---|
| Platform metrics (CPU, availability, etc.) | **Free** | Always-on, 93-day retention. |
| Custom metrics ingested via API/AMA | ~NZD $0.40 / 1M time-series samples | Rare for DSR. |
| Metrics queries via API | First 1M calls free, then ~NZD $1 / 1M calls | Negligible for portal users. |

### 4. Alerts

| Alert type | Price (per rule, per month) | Latency | Typical use |
|---|---|---|---|
| **Metric alert** | ~NZD $0.20 per signal | ~1 min | Storage availability, AGW health, VM CPU |
| **Log (scheduled-query) alert** — 5-min frequency | ~NZD $1.50 | 5 min min | 5xx bursts, soft-delete misuse, immutability changes |
| **Log alert** — 1-min frequency | ~NZD $4.50 | 1 min | Use sparingly — sev-1 only |
| **Activity Log alert** | **Free** | seconds | "Storage account deleted", role assignment changes |
| **Smart Detection** (App Insights) | Free | varies | Off for DSR |

### 5. Action groups & notifications

| Channel | Price | Notes |
|---|---|---|
| Action group itself | **Free** | |
| Email | **Free** | First 1,000 / month then negligible |
| Push notification (mobile app) | Free | First 1,000 / month |
| SMS | ~NZD $0.05 / message (NZ/AU) | Country-rated |
| Voice call | ~NZD $0.20 / call (NZ/AU) | Use for sev-1 only |
| Webhook / Logic App / Function | Free from action group; **target service** charges separately | |

### Lab cost summary (this module)

- LAW workspace: **$0** (well under the 5 GB/month free tier).
- Lab alerts (3 rules, deleted at the end): **<$0.05** total prorated.
- Action group: **$0**.
- **Total expected lab spend: under NZD $0.50.**

---

### 🧮 Worked example — 10 GB/day workspace

> [!NOTE]
> The numbers below are **illustrative only**, computed from the public price list. They are **not** based on DSR billing. To get DSR's real number, run the KQL query at the end of this section against the production workspace.

**Assumptions** (so you can recompute):
- Region: Australia East, prices in NZD, **indicative** (always check [Azure Monitor pricing](https://azure.microsoft.com/en-us/pricing/details/monitor/))
- Daily ingestion: **10 GB/day** = 300 GB/month
- Free Analytics tier: 5 GB/month deducted
- Retention: 90 days analytics for everything
- Alerts: **8 metric** + **4 log (5-min)** + **5 activity-log** rules
- Notifications: 1 action group, email only

#### Scenario A — Everything on **Analytics** plan (status quo for most teams)

| Component | Calc | Cost / month (NZD) |
|---|---|---|
| Ingestion | (300 − 5) × $4.30 | **$1 268.50** |
| Retention 31→90 days | 295 GB × (90 − 31)/30 × $0.18 | **$104.43** |
| Metric alerts (8) | 8 × $0.20 | $1.60 |
| Log alerts (4 @ 5-min) | 4 × $1.50 | $6.00 |
| Activity Log alerts (5) | free | $0.00 |
| Action group + email | free | $0.00 |
| **Total** | | **~$1 380 / month** |

#### Scenario B — **Mixed plan** (high-volume tables on Basic, audit on Auxiliary)

Assume the 10 GB/day breaks down as:
- 6 GB/day → high-volume, rarely-queried tables (`StorageBlobLogs`, `AGWAccessLogs`) → **Basic Logs**
- 1 GB/day → compliance/audit tables (`AzureActivity`, role changes) → **Auxiliary Logs**
- 3 GB/day → tables you actively query and alert on (`Heartbeat`, `Perf`, `AzureDiagnostics` for AGW firewall) → **Analytics**

| Component | Calc | Cost / month (NZD) |
|---|---|---|
| Analytics ingestion | (3 × 30 − 5) × $4.30 | **$365.50** |
| Basic Logs ingestion | 6 × 30 × $0.85 | **$153.00** |
| Auxiliary Logs ingestion | 1 × 30 × $0.25 | **$7.50** |
| Query scans on Basic/Aux | est. 5 GB scanned/month × $0.0075 | $0.04 |
| Retention 31→90 d (Analytics only) | 90 GB × 59/30 × $0.18 | **$31.86** |
| Metric alerts (8) | 8 × $0.20 | $1.60 |
| Log alerts (4 @ 5-min) — note: only against Analytics tables | 4 × $1.50 | $6.00 |
| Activity Log alerts (5) | free | $0.00 |
| Action group + email | free | $0.00 |
| **Total** | | **~$565 / month** |

#### Savings summary

| Scenario | Monthly cost | Savings vs A | % saved |
|---|---|---|---|
| **A — All Analytics** | ~$1 380 | — | — |
| **B — Mixed plan** | ~$565 | ~$815 / month | **~59%** |

**Trade-offs to flag before moving any DSR table to Basic/Auxiliary:**
1. You can no longer set **log alerts** against Basic/Auxiliary tables (metric alerts on the same resource still work).
2. Query language is a **subset** of KQL (no `join`, no `make-series`, limited transforms).
3. Workbook tiles backed by Basic/Aux tables may need rewriting.
4. Plan changes apply to **future** data only — historical data stays on whatever plan it was ingested under.

> [!TIP]
> Before changing any DSR table's plan, run a 30-day audit: "Which alerts query this table? Which workbook tiles? Which saved queries?" If the answer is "none in the last 90 days", it's a Basic Logs candidate. We do this audit hands-on in **Step 12 — Cost Management**.

#### How to get the **real** DSR ingestion number

Run this KQL in the DSR Log Analytics workspace (Reader is enough):

```kusto
// Last 30 days of billable ingestion, grouped by table
Usage
| where TimeGenerated > ago(30d)
| where IsBillable == true
| summarize BillableGB = sum(Quantity) / 1000 by DataType
| order by BillableGB desc
```

Then either:
- Multiply each row by the regional Analytics rate, or
- Open **Cost Management → Cost analysis** in the DSR subscription and filter Service name = "Log Analytics" / "Azure Monitor" for the authoritative actual spend.

```bash
# Cleanup (only if you don't want to keep the workspace)
az monitor log-analytics workspace delete -g rg-labs-foundations-<your-initials> -n law-lab-<initials> --yes

# Delete lab alert rules
az monitor metrics alert delete -g rg-labs-foundations-<your-initials> -n alert-stg-availability-lab
az monitor scheduled-query delete -g rg-labs-foundations-<your-initials> -n alert-stg-5xx-burst-lab
az monitor action-group delete -g rg-labs-foundations-<your-initials> -n ag-lab-oncall-<your-initials>
```

---

⬅️ **Previous:** [Step 09 — WOD container operations](step-09-wod-container-ops.md)
➡️ **Next:** [Step 11 — Azure Monitor Workbooks](step-11-workbooks.md)
