# Step 15 — Capstone: Monthly Cost Report

_The "where did the money go this month?" lab._ 🏆 Builds the second team-owned artefact: the Monthly Cost Report. Combines Cost Management (Step 13), Blob Inventory (Step 08), and Workbooks (Step 12) into the canonical $-per-tier-per-account view leadership reads on the first business day of each month.

> [!NOTE]
> **Trainee duration:** 180 minutes
> **Lab cost:** $0 — uses the export from Step 13 and the inventory from Step 08.
> **Prerequisites:** Steps 13 (Cost Management) + 12 (Operational Visibility) + 13 (Workbooks) complete. The scheduled cost export and a working blob inventory must be in place.
> **Pairs with:** Module 5 of the DIA training plan (Reporting). **Capstone — output is a real deliverable.**

---

## 📖 Session overview

The Monthly Cost Report answers four questions, in order:

1. What did ANL cost this month?
2. How does that compare to last month and budget?
3. What were the top three movers (positive and negative)?
4. What's the action plan for any anomaly?

This lab walks the build: ingest the daily cost CSV from Step 13 into a Workbook table, join with the latest blob inventory for capacity context, render the four canonical views, write the narrative, publish. By the end you have a repeatable monthly cycle.

**What you'll build**
- A **Workbook** named `ANL — Monthly Cost Report`.
- **Four core views**: total + MoM, by service, by tier, by storage account.
- A **top-3 movers** computation (ranked deltas).
- A **narrative section** for the analyst's commentary (~5 bullets).
- A **PDF export** for the distribution list.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **MoM** | Month-over-Month change. |
| **Mover** | A line item with a notable spend delta vs. previous period. |
| **Allocation** | The act of attributing a cost line to a tier or app. |
| **Billing day** | The day a cost record settles. Cost Management lags 24–48h, so the last 1–2 days of a month often arrive late. |
| **Reservation** | Pre-paid compute / DB capacity. Out of scope for this report; mentioned for completeness. |

## 📚 Prepare in advance

- Re-read Step 13 (Cost Management) — particularly the saved-view names and export schedule.
- Have the **previous month's report** open if one exists. The format is tracked in the SharePoint reports library.
- Check the latest blob inventory CSV for each ANL storage account is in the `inventory` container.

## 🧱 Foundational principles

- **One source per question.** Cost data → Cost Management export. Capacity context → Blob Inventory. Don't cross-source for the same number.
- **Always show MoM.** A standalone monthly number is meaningless.
- **Narrative > tables.** Numbers in the appendix; conclusions on page 1.
- **Cite the methodology.** Footnote: "Costs include the `app_name=anl` tag at PRD scope; excludes shared platform services."
- **Reproducible.** Anyone on the team should be able to regenerate this month's report from the JSON template + the data sources.

## ⌨️ Activity 1 — Verify your data sources are flowing

```bash
# Check cost export is landing
az storage blob list --account-name <export-account> --container-name cost-exports \
  --prefix anl/ --output table | head

# Check latest blob inventory is recent (per storage account)
az storage blob list --account-name <storage-account> --container-name inventory \
  --output table | head
```

Both should show files within the last 7 days. If not, fix the upstream pipeline before continuing.

## ⌨️ Activity 2 — Author the Workbook shell

1. Workbooks → **+ New**.
2. Top-of-canvas markdown:

```markdown
# ANL — Monthly Cost Report
**Period:** {Month:label}  |  **Owner:** _Name + email_
**Methodology:** PRD subscription scope; tag `app_name=anl`. Excludes shared platform services. Costs in NZD.
```

3. **Add → Add parameters**:
   - `Month` parameter — type "Date range", default "This month". Locked granularity = month.
   - `Subscription` parameter — type "Subscription". Single-select. Default = ANL prd.

4. Save: `ANL — Monthly Cost Report`.

## ⌨️ Activity 3 — KPI row: total + MoM

Cost Management API has a built-in "Cost details" data source — but the most reliable pattern is to query the exported CSVs in the storage account. DSR ingests them via Data Factory into `AnlCost_CL`. If your environment doesn't have that ingestion, fall back to Cost Management API directly via the Workbook's Costs data source.

```kql
// Total this period and previous period
let curr = AnlCost_CL
            | where Date_t >= startofmonth({Month:start})
            | where Date_t < startofmonth({Month:end})
            | summarize total_now = sum(CostInBillingCurrency_d);
let prev = AnlCost_CL
            | where Date_t >= startofmonth(datetime_add('month', -1, {Month:start}))
            | where Date_t < startofmonth({Month:start})
            | summarize total_prev = sum(CostInBillingCurrency_d);
curr | extend total_prev = toscalar(prev),
              mom_pct = round(100.0 * (total_now - toscalar(prev)) / toscalar(prev), 1)
```

Render as Tiles: "This month $", "Last month $", "MoM %".

## ⌨️ Activity 4 — View 1: by service

```kql
AnlCost_CL
| where Date_t >= startofmonth({Month:start})
| where Date_t < startofmonth({Month:end})
| summarize cost = sum(CostInBillingCurrency_d) by ServiceName_s
| order by cost desc
| extend pct = round(100.0 * cost / sum(cost), 1)
```

Render as Grid + Pie.

## ⌨️ Activity 5 — View 2: by tier

Join cost lines to the tag values.

```kql
AnlCost_CL
| where Date_t >= startofmonth({Month:start})
| where Date_t < startofmonth({Month:end})
| extend tier = tostring(Tags_s["tier"])
| summarize cost = sum(CostInBillingCurrency_d) by tier
| order by cost desc
```

Render as Grid + Pie. Untagged rows show as "" — that's a tag-hygiene action item from Step 13.

## ⌨️ Activity 6 — View 3: by storage account, with capacity context

```kql
let cost = AnlCost_CL
            | where Date_t >= startofmonth({Month:start})
            | where Date_t < startofmonth({Month:end})
            | where ServiceName_s == "Storage"
            | summarize cost = sum(CostInBillingCurrency_d) by ResourceId_s
            | extend account = tostring(split(ResourceId_s, '/')[8]);
let cap = StorageBlobInventory_CL
            | where TimeGenerated > ago(7d)
            | summarize totalGB = sum(Content_Length_d) / pow(1024, 3) by AccountName_s;
cost | join kind=leftouter cap on $left.account == $right.AccountName_s
| project account, cost, totalGB, costPerTB = round(cost / (totalGB / 1024), 2)
| order by cost desc
```

Now leadership can see "this account costs X/month and holds Y TB — i.e. Z $/TB". That's the unit economics.

## ⌨️ Activity 7 — Top 3 movers

```kql
let curr = AnlCost_CL
            | where Date_t >= startofmonth({Month:start})
            | where Date_t < startofmonth({Month:end})
            | summarize cost_now = sum(CostInBillingCurrency_d) by ResourceId_s;
let prev = AnlCost_CL
            | where Date_t >= startofmonth(datetime_add('month', -1, {Month:start}))
            | where Date_t < startofmonth({Month:start})
            | summarize cost_prev = sum(CostInBillingCurrency_d) by ResourceId_s;
curr | join kind=fullouter prev on ResourceId_s
| extend delta = coalesce(cost_now, 0.0) - coalesce(cost_prev, 0.0),
         pct = round(100.0 * (coalesce(cost_now, 0.0) - coalesce(cost_prev, 0.0))
                            / iff(coalesce(cost_prev, 0.0) == 0, 1.0, cost_prev), 1)
| where abs(delta) > 50  // ignore noise <$50
| project ResourceId_s, cost_now, cost_prev, delta, pct
| order by abs(delta) desc
| take 3
```

Render as Grid. The narrative section addresses each.

## ⌨️ Activity 8 — Narrative section

Markdown tile, edited each cycle by the report owner:

```markdown
## Commentary

- **This month: $X (MoM ±Y%).** _One-sentence headline._
- **Mover #1:** _Resource / why / action._
- **Mover #2:** _Resource / why / action._
- **Mover #3:** _Resource / why / action._
- **Outlook:** _Expected next month based on planned changes._
```

Five bullets. No more. The detail lives in the tiles.

## ⌨️ Activity 9 — Capacity tier breakdown (closes the loop)

Pull from the inventory CSVs (Step 08):

```kql
StorageBlobInventory_CL
| where TimeGenerated > ago(7d)
| where AccountName_s in (<your ANL accounts>)
| summarize totalGB = sum(Content_Length_d)/pow(1024,3) by AccessTier_s
| extend monthlyCost_NZD = case(
    AccessTier_s == "Hot",     totalGB * 0.045,    // ZRS Hot per GiB
    AccessTier_s == "Cool",    totalGB * 0.024,
    AccessTier_s == "Cold",    totalGB * 0.0125,
    AccessTier_s == "Archive", totalGB * 0.003,
    0.0)
| order by monthlyCost_NZD desc
```

This is the answer to "should we move more to Cool/Archive?". Confirm prices in the live pricing page before publishing — they change.

## ⌨️ Activity 10 — Publish

1. Print Workbook to PDF (US Letter landscape).
2. Filename: `ANL-MonthlyCost-YYYY-MM.pdf`.
3. Upload to SharePoint reports library.
4. Email short summary + PDF link to distribution group on the **first business day of the following month**.
5. Commit Workbook JSON to Git.

## 🦾 Now your turn!

1. Add a **Year-to-date trend** chart (12-month rolling).
2. Compute **forecast next month** from the last 90 days of daily cost (use `series_decompose_forecast` in KQL).
3. Add a tile that flags any **untagged cost lines** — these are tag-hygiene action items.
4. Build an automation that emails the report PDF to the distribution group on the first of the month (Logic Apps, or a runbook in Step 16).

## ✅ Success checklist

- [ ] You've authored the Workbook with the four canonical views.
- [ ] You've computed Top-3 movers and written commentary.
- [ ] The capacity-by-tier breakdown is populated from the latest inventory.
- [ ] The PDF export is in SharePoint with the correct filename.
- [ ] The Workbook JSON is committed.
- [ ] You can name the data sources and the tag taxonomy.

## 📚 Self-serve refresher

- DSR Application Operations runbook, Section 6 ("Monthly Cost Report").
- [Cost Management API reference](https://learn.microsoft.com/rest/api/cost-management/) — for direct queries.
- [Blob pricing](https://azure.microsoft.com/en-us/pricing/details/storage/blobs/) — refresh tier prices each cycle.

## 💰 Cost note

- All free — Workbook + KQL + SharePoint upload.
- Capstone artefact lives on. No cleanup.

---

⬅️ **Previous:** [Step 14 — Capstone: Weekly Health Report](step-14-capstone-weekly-health.md)
➡️ **Next:** [Step 16 — Capstone: Incident Triage Tabletop](step-16-capstone-incident-triage.md)
