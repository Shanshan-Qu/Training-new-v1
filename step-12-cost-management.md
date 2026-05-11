# Step 12 — Cost Management for an application owner

_The "where is the money going?" lab._ 💰 Builds the muscle for Monthly Cost Reporting: saved views, scheduled exports, anomaly alerts, and the tag hygiene that makes the whole thing possible.

> [!NOTE]
> **Trainee duration:** 90 minutes
> **Lab cost:** $0 — Cost Management is free.
> **Prerequisites:** Steps 00–09 complete (you've created enough resources to see real cost data).
> **Pairs with:** Module 5 of the DIA training plan (Reporting). Feeds directly into the Phase 5 Capstone (Step 14 — Monthly Cost Report).

> [!IMPORTANT]
> **Shared-sandbox callout.** Cost Management **exports**, **budgets**, and **anomaly alerts** are scoped to the *subscription*, and their names must be unique across the cohort. Always suffix every name in this lab with `-<your-initials>` (e.g. `anl-daily-export-sq`, `anl-monthly-budget-sq`). Saved views are private to you and don't need a suffix.

---

## 📖 Session overview

The Preservation Team owns the Monthly Cost Report for ANL. Cost Management is the tool. This lab teaches the saved-view pattern (one view per dimension that matters), scheduled CSV exports (the same data the Capstone consumes), and the anomaly alerts that prevent the next "why did blob cost double?" surprise. Tag hygiene gets attention too — without `app_name=anl`, none of this works.

**What you'll learn**
- The Cost Management hierarchy: **billing scope → subscription → resource group → resource → tag**.
- How to build **saved views** with parameters (date range, group-by tag, filter env).
- **Scheduled exports** of cost data to a storage account (the source for the Capstone Workbook).
- **Anomaly detection** and budget alerts at sub and tag scope.
- The DSR tag taxonomy: `app_name`, `environment`, `tier`, `cost_centre`, `owner`.
- How to verify a tag is on every resource it should be on.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **Cost Management** | Azure's free billing analytics surface. Free for all customers. |
| **Saved view** | A named cost analysis with filters, group-by, and date range. |
| **Scheduled export** | A daily/weekly CSV of cost data dropped to a storage container. |
| **Budget** | A monthly target with alert thresholds (50%, 80%, 100%). |
| **Anomaly alert** | Cost Management's ML-based notice when daily spend deviates significantly. |
| **Tag** | A key-value label on a resource, used to attribute cost to apps/teams. |
| **EA / MCA** | Enterprise Agreement / Microsoft Customer Agreement — the contract DIA holds. |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Describe Azure Cost Management](https://learn.microsoft.com/training/modules/describe-cost-management-azure/) | The basics. |
| [Optimize costs with Cost Management](https://learn.microsoft.com/azure/cost-management-billing/costs/cost-mgt-best-practices) | The patterns DSR follows. |
| [Schedule a cost export](https://learn.microsoft.com/azure/cost-management-billing/costs/tutorial-export-acm-data) | The export pattern feeding the Capstone. |

About **1.5 hours** of optional pre-reading.

## 🧱 Foundational primer

- **Cost Management lags 24–48h.** Don't expect today's cost in real time.
- **Tag inheritance is opt-in.** A tag on the RG does *not* automatically apply to child resources unless Policy enforces it.
- **`app_name=anl` is the magic key.** Filtering all DSR cost on this tag is the canonical "what does ANL cost?" answer.
- **Scope matters.** A saved view at sub scope sees everything; at RG scope, just that RG. The Monthly Report uses sub scope filtered by tag.
- **Exports are append-mode** — daily files, organised by year/month/day in the destination container.
- **Anomaly alerts are free** but only run at sub or RG scope.

## ⌨️ Activity 1 — Open Cost Management at sub scope

1. Subscription → **Cost Management → Cost analysis**.
2. Default view: Accumulated cost month-to-date by service.
3. Note the warning if your sub is new — first 24h of data may not be available.

## ⌨️ Activity 2 — Build a saved view "ANL by service"

1. Cost analysis → **+ New view**.
2. Filters: `Tag : app_name = anl`.
3. Group by: Service name.
4. Granularity: Daily.
5. Time range: Last 30 days.
6. Save view: name `ANL — Spend by service`. Private (only you see it). Save.

You can now load this view from any browser as one click.

## ⌨️ Activity 3 — Build a saved view "ANL by environment"

1. **+ New view** (clone of #1).
2. Group by: Tag → `environment`.
3. Save: `ANL — Spend by environment`.

You'll see the prd/tst/dev split. If a resource has no `environment` tag, it appears under "(Untagged)" — that's a tag-hygiene action item.

## ⌨️ Activity 4 — Build a saved view "ANL by storage account"

1. **+ New view**.
2. Filters: `Tag: app_name = anl` AND `Service name: Storage`.
3. Group by: Resource.
4. Save: `ANL — Storage spend`.

This is the view you'll eyeball weekly. Sudden movement = something to investigate.

## ⌨️ Activity 5 — Schedule a CSV export

1. Cost Management → **Exports → + Add**.
2. Name: `anl-daily-export-<your-initials>` (subscription-scoped — must be unique across the cohort). Frequency: Daily. Date: Month-to-date costs.
3. Storage account: pick a storage account you own. Container: `cost-exports`. Folder: `anl/`.
4. Save.
5. Wait ~24h, then check the container — files appear at `anl/<year>/<month>/<filename>.csv`.

This is the source for the Capstone (Step 14) — you point a Workbook at this container.

## ⌨️ Activity 6 — Set a budget with alerts

1. Cost Management → **Budgets → + Add**.
2. Name: `anl-monthly-budget-<your-initials>` (subscription-scoped — must be unique across the cohort). Scope: subscription. Filter: `Tag: app_name = anl`.
3. Amount: a sensible monthly target (talk to your manager — it's a real conversation).
4. Alerts at 80%, 90%, 100%. Recipient: **your own email only** for the lab (do not send to the team DL — every trainee creating one would spam the list).
5. Save.

If ANL spend hits 80% of the monthly budget, your team gets an email. Standard practice in DSR.

## ⌨️ Activity 7 — Anomaly alert

1. Cost Management → **Cost alerts → + Add → Anomaly alert**.
2. Name: `anl-anomaly-<your-initials>` (subscription-scoped — must be unique across the cohort). Scope: subscription. Filter: `Tag: app_name = anl`.
3. Recipient: your email.
4. Save.

Cost Management trains a baseline on ~14 days of history and alerts when daily spend deviates by configured thresholds. Quietly sits in the background until something interesting happens.

## ⌨️ Activity 8 — Tag hygiene audit

```kql
// Resource Graph — every ANL resource missing the env tag
resources
| where tags.app_name == "anl"
| where isempty(tags.environment) or tags.environment == ""
| project name, type, resourceGroup, location
```

These are the resources that show as "(Untagged)" in your environment view. Send the list to whoever owns the resource and ask them to fix.

## 🦾 Now your turn!

1. Add a fourth saved view: "ANL — Cost by tier" (group by `tier` tag). What percentage is `tier=storage` vs `tier=web`?
2. Set up a scheduled email of the "ANL — Spend by service" view to your manager every Monday morning.
3. From the export CSV, find the row with the highest daily cost. Is it expected (e.g., backup) or surprising (e.g., egress)?
4. Open Cost Management at *RG* scope for the lab RG — confirm you see only that RG's spend.

## ✅ Success checklist

- [ ] You can name the four DSR tag keys and what each means.
- [ ] You can build, save, and load a view in Cost Management.
- [ ] You've scheduled a CSV export to a storage container.
- [ ] You've set a budget with email alerts.
- [ ] You've enabled anomaly detection.
- [ ] You can write the Resource Graph query that finds untagged resources.

## 📚 Self-serve refresher

- [Cost analysis common scenarios](https://learn.microsoft.com/azure/cost-management-billing/costs/cost-analysis-common-uses) — pattern bank.
- [Cost Management API](https://learn.microsoft.com/rest/api/cost-management/) — for automation in Step 14's Capstone.

## 💰 Cost note

- Cost Management itself: free.
- Storage container holding exports: trivial blob cost (~$0.02/month for typical export sizes).
- No cleanup needed if you keep the views — they cost nothing.

---

⬅️ **Previous:** [Step 11 — Azure Monitor Workbooks](step-11-workbooks.md)
➡️ **Next:** [Step 13 — Capstone: Weekly Health Report](step-13-capstone-weekly-health.md)
