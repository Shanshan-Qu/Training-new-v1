# Step 12 — Azure Monitor Workbooks

_The "one-pager that updates itself" lab._ 📈 Builds the parameterised dashboards your team will run weekly: backup health, storage capacity, AGW backend status, all on a single canvas you can share by URL.

> [!NOTE]
> **Trainee duration:** 90 minutes
> **Lab cost:** $0 — Workbooks are free, you reuse the workspace from Step 11.
> **Prerequisites:** Step 11 complete; the saved KQL functions you built there.
> **Pairs with:** Module 4 of the DIA training plan (Observability & Reporting).

---

## 📖 Session overview

A Workbook is a sharable, parameterised report built from KQL queries. DSR uses one Workbook per concern — "DSR Health Daily", "ANL Storage Capacity", "Backup Compliance" — and a curated subset is pinned to dashboards. This lab walks you through authoring one from scratch, parameterising it (time range, environment dropdown), wiring tiles to the queries you saved in Step 11, and sharing it with your team.

**What you'll learn**
- The Workbook anatomy: **parameters → queries → visualisations**.
- How to use **time range** and **resource dropdown** parameters.
- How to render KQL results as **grids, charts, tiles**.
- How to share a Workbook (link, pin to dashboard, export to template).
- The DSR pattern of one canonical Workbook per concern.
- Common pitfalls: stale parameters, expensive queries, RBAC drift on shared links.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **Workbook** | An interactive report built on KQL queries with parameters and visualisations. |
| **Parameter** | A typed input (time range, resource selector, dropdown) that queries can reference. |
| **Query step** | A single KQL block that produces a tile. |
| **Visualisation** | How the query result is rendered — grid, chart, tile, map, etc. |
| **Template** | The exported JSON of a Workbook. Versioned in Git in DSR. |
| **Pinned tile** | A single Workbook query embedded on an Azure Dashboard. |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Visualize data with Azure Monitor Workbooks](https://learn.microsoft.com/training/modules/visualize-data-workbooks/) | The official tour. |
| [Workbook parameters](https://learn.microsoft.com/azure/azure-monitor/visualize/workbooks-parameters) | The most useful single doc — every parameter type. |
| [Share a workbook](https://learn.microsoft.com/azure/azure-monitor/visualize/workbooks-access-control) | RBAC and link sharing. |

About **1.5 hours** of optional pre-reading.

## 🧱 Foundational primer

- **A Workbook lives in a Log Analytics workspace** (or in the Monitor service for cross-workspace).
- **Parameters are typed.** A Time Range param drops directly into queries via `{TimeRange}`.
- **Resource selector parameters** support multi-select. Drives queries via `{Resource}`.
- **Anyone with Reader on the workspace can run a Workbook.** Authoring needs Contributor.
- **Workbooks are JSON.** DSR stores curated Workbooks in Git; CI deploys them via ARM template.
- **Don't put Owner-equivalent data on a shared Workbook** without checking RBAC. The link grants whatever the viewer's existing permissions are; broader sharing means broader exposure.

## ⌨️ Activity 1 — Create your first Workbook

1. Workspace → **Workbooks → + New**.
2. Click the empty canvas → **Add → Add text**. Type "ANL Health — Lab Edition".
3. **Add → Add parameters**. Add:
   - Time Range, name `TimeRange`, default Last 24 hours.
   - Resource picker, name `Storage`, type `Microsoft.Storage/storageAccounts`, multi-select.
4. **Add → Add query**. Paste:

```kql
Heartbeat
| where TimeGenerated {TimeRange}
| summarize lastBeat = max(TimeGenerated) by Computer
| extend ageMin = datetime_diff('minute', now(), lastBeat)
```

5. Visualisation: Grid. Run query. You see your VMs.
6. Save (top-left). Name: `Lab — ANL Health`. Subscription/RG: your lab RG.

## ⌨️ Activity 2 — Add a chart tile

1. Canvas → **Add → Add query**. Paste:

```kql
StorageBlobLogs
| where TimeGenerated {TimeRange}
| where StatusCode >= 500
| summarize errors = count() by bin(TimeGenerated, 5m)
```

2. Visualisation: **Time chart**. Run.
3. Style: title "5xx errors over time".
4. Done editing → Save.

## ⌨️ Activity 3 — Use the resource dropdown

Replace the previous query with one that respects the Storage parameter:

```kql
StorageBlobLogs
| where TimeGenerated {TimeRange}
| where _ResourceId in~ ({Storage:id})
| where StatusCode >= 500
| summarize errors = count() by tostring(split(_ResourceId, '/')[8]),
                                bin(TimeGenerated, 5m)
| render timechart
```

Now the chart only shows accounts the viewer has selected.

## ⌨️ Activity 4 — KPI tile

1. **Add → Add query**.

```kql
StorageBlobLogs
| where TimeGenerated {TimeRange}
| summarize requests = count(),
            errors = countif(StatusCode >= 500),
            errorRate = round(100.0 * countif(StatusCode >= 500) / count(), 2)
```

2. Visualisation: **Tiles**. Configure:
   - Tiles → Title: "Total Requests", Subtitle: "{requests}"
   - Tile 2: "Errors", "{errors}"
   - Tile 3: "Error Rate", "{errorRate}%" (red threshold > 1.0)

These are the always-visible numbers users want at a glance.

## ⌨️ Activity 5 — Pin a tile to an Azure Dashboard

1. On any tile → ⋮ → **Pin to dashboard**.
2. Pick or create a dashboard. Save.
3. Open the dashboard from the Azure Portal home → see your tile updating live.

DSR has a "DSR Operations" dashboard with tiles from ~6 Workbooks. This is how the daily standup view is built.

## ⌨️ Activity 6 — Share the Workbook

1. From the Workbook → **Share**. Copy the link.
2. RBAC reminder: viewer needs Reader on the workspace and the resources referenced.
3. Send the link to a colleague (or your own personal email — internal only).
4. Open it in another browser session — confirm the parameters preselect correctly.

## ⌨️ Activity 7 — Export to template (Git-friendly)

1. **Edit → Advanced editor**. You see the JSON.
2. Copy the JSON. Save to `~/anl-workbooks/anl-health.json` (or commit to your repo).
3. To redeploy on a fresh sub: portal → **Workbooks → + New → Advanced editor → Paste JSON → Run**.

This is how DSR keeps Workbooks reproducible across environments.

## 🦾 Now your turn!

1. Add a parameter `Environment` with options `prd`, `tst`, `dev` (drop-down). Wire one query to filter by `tags.environment == "{Environment}"`.
2. Build a Workbook called **ANL Backup Daily** with three tiles: backup job count last 24h, failures last 24h, oldest restore point per protected resource. (Some queries return empty until you do Step 11; that's fine.)
3. Find a Microsoft-published gallery Workbook (e.g. Azure Backup) — use it as a template, save a copy, customise.
4. Pin three tiles to a single dashboard and screenshot the result.

## ✅ Success checklist

- [ ] You've authored a Workbook with at least one parameter and one chart.
- [ ] You can use `{TimeRange}` and a resource picker in a KQL query.
- [ ] You can pin a tile to an Azure Dashboard.
- [ ] You've shared a Workbook by link and verified what RBAC the viewer needs.
- [ ] You've exported a Workbook to JSON and re-imported it.
- [ ] You understand which DSR concerns each have a canonical Workbook.

## 📚 Self-serve refresher

- [Workbook visualisations gallery](https://learn.microsoft.com/azure/azure-monitor/visualize/workbooks-visualizations) — every chart type with examples.
- [Common Workbook patterns](https://github.com/microsoft/Application-Insights-Workbooks) — community library.

## 💰 Cost note

- Workbooks themselves: free.
- Underlying queries: charged per GB scanned in the workspace (negligible for daily reports).
- No cleanup required — Workbooks have no separate cost footprint.

---

⬅️ **Previous:** [Step 11 — Operational Visibility & Alerts](step-11-operational-visibility.md)
➡️ **Next:** [Step 13 — Cost Management for an application owner](step-13-cost-management.md)
