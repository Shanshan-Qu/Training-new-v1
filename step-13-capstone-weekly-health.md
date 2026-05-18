# Step 13 — Capstone: Weekly Health Report (co-design workshop)

> [!IMPORTANT]
> **STATUS: RESHAPED.** Run this as a **co-design workshop**: the DP team defines the KPIs, layout, and audience for the Weekly Health Report **live in the session**, then we build the Workbook against their spec. Pre-work: ask the team to bring 3–5 candidate KPIs.

_The "Friday morning one-pager" lab._ 🏆 Combines everything from Phase 1–4 into a real, repeatable artefact: the Weekly Health Report your team will publish every Friday.

> [!NOTE]
> **Trainee duration:** 180 minutes (this is a build, not a walkthrough)
> **Lab cost:** $0 — uses the Workbook from Step 11 and queries from Step 10.
> **Prerequisites:** Steps 00–10 complete. The functions you saved in Step 10 and the Workbook from Step 11 are required inputs.
> **Pairs with:** Module 4 + Module 5 of the DIA training plan (Observability + Reporting). **This is a Capstone — output is a real deliverable, not a lab artefact.**

---

## 📖 Session overview

Your team publishes a Weekly Health Report every Friday at 9 AM NZT. The audience is the Preservation Team Lead, Cloud Platform peers, and the Application Owners. The format is a one-pager: traffic light per tier (green/amber/red), one chart, three callouts, link to the underlying Workbook for detail. This lab walks the **build** end-to-end: define the metric set, write the queries, render in a Workbook, apply a consistent style, share by URL. By the end of the lab you have a working report committed to your team's GitHub.

**What you'll build**
- A **Workbook** named `ALNZ — Weekly Health Report`.
- **Five core tiles**: VM heartbeat, AGW backend health, Storage 5xx, Backup compliance, Defender alerts.
- **One headline trend chart** (e.g. AGW request volume over 7 days).
- **Three KPI tiles** with traffic-light thresholds.
- A short **markdown narrative section** ("This week's highlights") that the report owner edits weekly.
- A **shareable link** (and a pinned dashboard view).

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **Capstone** | A multi-step exercise that combines earlier labs into a single deliverable. |
| **Traffic light** | Green/Amber/Red colour coding on a tile. Visually scannable in 2 seconds. |
| **Trend chart** | A time-series line/area showing how a metric has moved over the period. |
| **Headline metric** | The single number the report leads with — typically backup-compliance % or 5xx error rate. |
| **Report owner** | One named team member each week, on rotation, who edits the narrative section. |

## 📚 Prepare in advance

- Re-read Step 11 (Workbooks). You'll be in the Workbook editor for most of this lab.
- Have your Step 10 saved functions ready (the `Alnz1HeartbeatGaps`, `Alnz2StorageErrors`, etc.).
- Have the **DSR Application Operations runbook** open — Section 5 ("Weekly Health Report") is the audit trail of past reports and the source of truth for the metric set.

## 🧱 Foundational principles

- **Single source of truth.** The Workbook *is* the report. The PDF/email exports are derivatives.
- **No more than seven tiles.** A one-pager that needs scrolling defeats the purpose.
- **Traffic-light thresholds match runbook SLA.** Don't invent thresholds; use the documented ones.
- **Narrative is short.** Three bullets max. If something needs a paragraph, link to a separate write-up.
- **Date range is locked.** "Last 7 days, ending Friday 09:00 NZT". Make this a fixed parameter so the visuals are reproducible.

## ⌨️ Activity 1 — Author the Workbook shell

1. Workspace → **Workbooks → + New**.
2. Top of canvas → **Add → Add text** (markdown):

```markdown
# ALNZ — Weekly Health Report
**Reporting period:** {TimeRange:label}  |  **Owner:** _Set this week's owner name_
```

3. **Add → Add parameters**:
   - `TimeRange`, default Last 7 days.
   - `Environment`, dropdown values `prd, tst, dev`, default `prd`.

4. Save. Name: `ALNZ — Weekly Health Report`. Subscription/RG: your prod Log Analytics workspace's RG.

## ⌨️ Activity 2 — KPI tile row (3 tiles)

```kql
// KPI 1 — Backup compliance %
let total = AddonAzureBackupProtectedInstance
            | where TimeGenerated {TimeRange}
            | summarize total = dcount(BackupItemUniqueId);
let inSLA = AddonAzureBackupProtectedInstance
            | where TimeGenerated {TimeRange}
            | extend ageHours = datetime_diff('hour', now(), todatetime(LatestRecoveryPoint_s))
            | where ageHours <= 26
            | summarize inSLA = dcount(BackupItemUniqueId);
total | extend inSLA = toscalar(inSLA), pct = round(100.0 * toscalar(inSLA) / total, 1)
| project pct
```

Visualisation: Tiles. Title "Backup compliance". Subtitle "{pct}%". Threshold: ≥98 green, 95–98 amber, <95 red.

```kql
// KPI 2 — 5xx error rate (storage)
StorageBlobLogs
| where TimeGenerated {TimeRange}
| summarize total = count(),
            errors = countif(StatusCode >= 500),
            pct = round(100.0 * countif(StatusCode >= 500) / count(), 3)
```

Threshold: <0.1% green, 0.1–0.5% amber, >0.5% red.

```kql
// KPI 3 — High-severity Defender alerts
SecurityAlert
| where TimeGenerated {TimeRange}
| where AlertSeverity == "High"
| summarize highCount = count()
```

Threshold: 0 green, 1–2 amber, ≥3 red.

## ⌨️ Activity 3 — Tier health grid (1 tile)

```kql
Heartbeat
| where TimeGenerated {TimeRange}
| extend tier = case(Computer startswith "vm-rosi", "Web/App",
                     Computer startswith "vm-roix", "Index",
                     Computer startswith "vm-rowf", "Workflow",
                     Computer startswith "vm-ora",  "Oracle",
                     Computer startswith "vm-wod",  "WOD",
                     "Other")
| summarize lastBeat = max(TimeGenerated), vmCount = dcount(Computer) by tier
| extend ageMin = datetime_diff('minute', now(), lastBeat)
| project tier, vmCount, lastBeat, ageMin,
          status = case(ageMin <= 5, "🟢", ageMin <= 15, "🟡", "🔴")
```

Visualisation: Grid. Sort by tier.

## ⌨️ Activity 4 — Headline trend (1 tile)

```kql
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS"
| where Category == "ApplicationGatewayAccessLog"
| where TimeGenerated {TimeRange}
| summarize requests = count(),
            errors = countif(httpStatus_d >= 500) by bin(TimeGenerated, 1h)
| render timechart
```

Title: "AGW request volume + 5xx — last 7 days".

## ⌨️ Activity 5 — Narrative section (markdown)

Add a text tile *editable each week*:

```markdown
## This week's highlights

- _Highlight 1 — short, cross-link to detail._
- _Highlight 2._
- _Highlight 3._

**Action items carried into next week:** _list_
```

Convention: report owner edits this section before publishing each Friday.

## ⌨️ Activity 6 — Apply traffic-light thresholds consistently

Open each KPI tile → Tile settings → **Render preferences** → set thresholds matching the runbook:

| Metric | Green | Amber | Red |
|---|---|---|---|
| Backup compliance % | ≥98 | 95–98 | <95 |
| Storage 5xx rate | <0.1% | 0.1–0.5% | >0.5% |
| High-sev Defender alerts | 0 | 1–2 | ≥3 |
| Heartbeat lag (per tier) | ≤5 min | 5–15 min | >15 min |

**The thresholds match the runbook.** Don't invent.

## ⌨️ Activity 7 — Pin a one-glance dashboard

1. From the Workbook, pin the three KPI tiles + the trend chart to a new dashboard.
2. Dashboard name: `ALNZ Weekly Health (latest)`.
3. Share with the Preservation Team distribution group (Reader on the workspace required).

## ⌨️ Activity 8 — Export to JSON, commit to Git

1. Workbook → Edit → **Advanced editor → JSON**.
2. Copy. Save to `~/alnz-workbooks/alnz-weekly-health.json`.
3. Commit:

```bash
cd ~/alnz-workbooks
git add alnz-weekly-health.json
git commit -m "Capstone: Weekly Health Report v1"
git push
```

Now any teammate can deploy the same Workbook into their workspace via the Advanced editor.

## ⌨️ Activity 9 — Generate a PDF (optional, for stakeholder email)

1. Workbook → Print (browser print → Save as PDF, US Letter landscape).
2. Filename convention: `ALNZ-WeeklyHealth-YYYY-MM-DD.pdf`.
3. Save to the team SharePoint folder.

## ⌨️ Activity 10 — Run through the report end-to-end

Pretend it's Friday 09:00:

1. Open Workbook. Read each tile out loud.
2. Identify any non-green status. Note the cause (or "investigation pending").
3. Write the three highlight bullets.
4. Write any action items to carry over.
5. Email a short summary to the distribution group with the Workbook link.

This is the actual Friday morning ritual. The lab simulates it once.

## 🦾 Now your turn!

1. Add a fourth KPI tile for **Storage capacity growth this week** (delta GiB from Step 08's inventory).
2. Build a parameter `Tier` so the report can be filtered to a single Rosetta tier.
3. Set the dashboard to refresh every 1 hour automatically.
4. Add a "Last 4 weeks of backup compliance" trend tile to spot drift.

## ✅ Success checklist

- [ ] You've authored the Workbook with KPI + grid + chart + narrative tiles.
- [ ] All thresholds match the DSR runbook.
- [ ] The Workbook's JSON is committed to your team Git.
- [ ] You've pinned a one-glance dashboard for the team.
- [ ] You've walked the Friday-morning ritual end-to-end.
- [ ] The report owner role is documented (rotation roster, what they edit, when they publish).

## 📚 Self-serve refresher

- DSR Application Operations runbook, Section 5 ("Weekly Health Report").
- [Workbook templates gallery](https://learn.microsoft.com/azure/azure-monitor/visualize/workbooks-overview) — Microsoft-curated examples.

## 💰 Cost note

- Workbook itself: free.
- Underlying queries: a few cents per week of workspace data scanned. Trivial.
- No cleanup — this is a real artefact.

---

⬅️ **Previous:** [Step 12 — Cost Management for an application owner](step-12-cost-management.md)
➡️ **Next:** [Step 14 — Capstone: Monthly Cost Report](step-14-capstone-monthly-cost.md)
