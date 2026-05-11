# Step 12 — Blob inventory & capacity reporting

_The "where did all the storage go?" lab._ 📊 Sets up Blob Inventory — the weekly CSV that powers DSR's capacity-by-tier reporting, the Monthly Cost Report, and any "how much is at Cold tier?" question from leadership.

> [!NOTE]
> **Trainee duration:** 90 minutes
> **Lab cost:** under NZD $0.50 — one inventory rule and the resulting CSV.
> **Prerequisites:** Steps 05 + 06 complete.
> **Pairs with:** Module 2 + Module 5 of the DIA training plan (Storage / Reporting). **Closes Phase 2** — you'll use the inventory CSVs in Phase 4 (Workbooks) and Phase 5 (Capstones).

---

## 📖 Session overview

Storage Insights gives near-real-time metrics, but for *blob-level* reporting (count by tier, last access date, individual blob size) you need **Blob Inventory**: a scheduled job that writes a CSV (or Parquet) of every blob in a storage account to a destination container, with all the properties you ask for. DSR runs weekly inventory on each storage account; the resulting CSVs feed the Monthly Cost Report and the capacity-by-tier dashboard. This lab gets you fluent in authoring inventory rules, reading the output, and querying it with KQL or Excel.

**What you'll learn**
- How **Blob Inventory** works (scheduled job, output format, destination container).
- How to author a **rule**: filters, fields, schedule.
- How to **read** the resulting CSV — the columns that matter (Name, Size, Tier, LastModified, AccessTier, Snapshot, etc.).
- How to **import the CSV** into KQL (Log Analytics) for capacity-by-tier queries, or into Excel for tabular reporting.
- The DSR weekly pattern: rule per storage account, output to a dedicated `inventory` container, retention 90 days.
- How to spot anomalies (sudden tier shift, unexpected blob growth, snapshot bloat).

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **Blob Inventory** | An Azure Storage feature that writes a CSV/Parquet listing of every blob in an account on a schedule. |
| **Rule** | A single inventory definition: which container, which fields, which schedule. |
| **Manifest** | The summary file produced alongside each inventory run. |
| **Inventory CSV** | The actual blob listing — one row per blob (and optionally per version/snapshot). |
| **Schema fields** | The columns to include — Name, Size, AccessTier, etc. Choose minimal to keep CSV small. |
| **Inventory run** | One execution of the rule; produces one CSV + manifest in the destination container. |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Calculate blob count and total size with inventory](https://learn.microsoft.com/azure/storage/blobs/calculate-blob-count-size) | The exact pattern for the Monthly Cost Report. |
| [Enable Azure Storage blob inventory reports](https://learn.microsoft.com/azure/storage/blobs/blob-inventory) | The reference for every field. |
| [Power Query for Excel](https://support.microsoft.com/office/power-query-overview-and-learning-ed614c81-4b00-4291-bd3a-55d80767f81d) | Read the CSV into a refreshable Excel workbook. |

About **1.5 hours** of optional pre-reading.

## 🧱 Foundational primer

- **Inventory runs daily or weekly.** Pick weekly for typical capacity reporting; daily only for high-velocity accounts.
- **Output is per-rule, per-run.** Each run writes a folder dated `YYYY/MM/DD/HH-MM-SS/<rule-name>/` in the destination container.
- **Schema is configurable.** Minimal: Name, Tier, Size. Add what you need; every column adds CSV size and cost.
- **Versions and snapshots are *optional* in the listing.** Include them when you want to see version overhead.
- **The CSV is the source of truth for capacity reporting.** Cost Management aggregates differently and lags 24–48h.
- **Inventory CSVs are themselves blobs** — they accrue cost. Apply lifecycle rule on the inventory container to expire old reports.
- **Use Parquet for KQL ingestion at scale.** CSV is human-readable; Parquet is columnar and faster for big accounts.

## ⌨️ Activity 1 — Author your first inventory rule

1. Storage account → **Blob inventory → + Add a rule**.
2. Name: `lab-weekly-inv`. Object type: **Blob** (uncheck Container/Version unless you need them).
3. Output container: **+ Create new** → name `inventory`. Output format: **CSV**. Schedule: **Weekly**.
4. Filter — Blob types: BlockBlob. Prefix: leave blank to inventory all.
5. Schema fields — at minimum:
   - `Name`
   - `Content-Length`  (size in bytes)
   - `BlobType`
   - `AccessTier`
   - `LastModified`
   - `Creation-Time`
6. Save.

> [!TIP]
> Pick the schema fields the report consumer actually uses. The bigger the schema, the bigger the CSV, the more storage and read transactions cost.

## ⌨️ Activity 2 — Trigger an immediate run (read-only walkthrough)

Inventory runs are triggered by the schedule — there's no "run now" button. To force one:
1. Edit the rule's schedule from Weekly → Daily → Save → wait until next 1 AM UTC. (Don't actually do this in lab; we'll just wait for the weekly slot.)
2. Or use the CLI: `az storage account blob-inventory-policy create/show` to inspect the policy JSON.

For the rest of this lab, use the **sample CSV** below (we'd otherwise need to wait a week):

```csv
Name,Content-Length,BlobType,AccessTier,LastModified,Creation-Time
container1/wod/2023/jan/page1.warc,524288000,BlockBlob,Cool,2026-04-15T10:23:01.0000000Z,2023-01-12T10:23:01.0000000Z
container1/wod/2024/jul/page2.warc,318767104,BlockBlob,Hot,2026-04-30T08:11:20.0000000Z,2024-07-08T08:11:20.0000000Z
container1/ie/aip/00012345.zip,1073741824,BlockBlob,Cold,2026-01-02T22:01:45.0000000Z,2022-08-19T22:01:45.0000000Z
... (production has millions of rows)
```

## ⌨️ Activity 3 — Capacity-by-tier in Excel

1. Save the sample CSV to your laptop.
2. Excel → Data → From Text/CSV → import.
3. Pivot table: Rows = AccessTier; Values = Sum of Content-Length / 1024^3 (GiB).
4. You now have GB-by-tier in seconds. Add a column for cost-per-tier (Hot $0.03, Cool $0.018, Cold $0.006) and you have the cost-by-tier breakdown for the Monthly Cost Report.

## ⌨️ Activity 4 — Capacity-by-tier in KQL (production pattern)

When the inventory CSVs are ingested to a Log Analytics workspace (via a small Logic App or Data Factory pipeline — DIA Platform owns this part), you query them like this:

```kql
StorageBlobInventory_CL
| where TimeGenerated > ago(7d)
| where AccountName_s == "stanlnznblobprdwod01"
| summarize totalGB = sum(Content_Length_d) / pow(1024, 3) by AccessTier_s
| order by totalGB desc
```

Or by month for trend:

```kql
StorageBlobInventory_CL
| where AccountName_s == "stanlnznblobprdwod01"
| extend month = startofmonth(TimeGenerated)
| summarize totalGB = sum(Content_Length_d)/pow(1024,3) by month, AccessTier_s
| render columnchart kind=stacked
```

## ⌨️ Activity 5 — Set retention on the inventory container

Inventory CSVs are blobs themselves — apply a lifecycle rule.

1. Storage account → **Lifecycle management → + Add rule**.
2. Name: `expire-inventory-csvs-after-90d`. Filter prefix: `inventory/`.
3. Action: delete after 90 days since modification.
4. Save.

This is the DSR pattern — keep 90 days of weekly inventories online; archive longer history out of band if needed.

## ⌨️ Activity 6 — Spot anomalies

In the Monthly Cost Report, you'll compare two adjacent inventories:

```kql
let curr = StorageBlobInventory_CL | where TimeGenerated between (ago(7d) .. now())
            | summarize totalGB_now = sum(Content_Length_d)/pow(1024,3) by AccessTier_s;
let prev = StorageBlobInventory_CL | where TimeGenerated between (ago(35d) .. ago(28d))
            | summarize totalGB_prev = sum(Content_Length_d)/pow(1024,3) by AccessTier_s;
curr | join prev on AccessTier_s
| project AccessTier_s, totalGB_now, totalGB_prev,
          delta_GB = totalGB_now - totalGB_prev,
          delta_pct = (totalGB_now - totalGB_prev) / totalGB_prev * 100
```

Any tier with a >20% MoM change is worth flagging in the report.

## 🦾 Now your turn!

1. Add a **VersionId** column to your inventory schema and re-run. The CSV now has one row per version — what does the size delta look like vs. base blobs only?
2. Author a second inventory rule with prefix filter `wod/` only. Compare CSV sizes — should be smaller.
3. Find the manifest file for one of your runs. Open it and identify: number of objects, total size, schema, completion timestamp.
4. Write the KQL that answers: "Of the WOD storage account's content, what fraction is older than 365 days but still on Hot tier?"

## ✅ Success checklist

- [ ] You can author an inventory rule and explain every field.
- [ ] You can read a CSV and produce capacity-by-tier in Excel.
- [ ] You can write a KQL query that aggregates the inventory by tier.
- [ ] You understand inventory CSVs are themselves blobs and need their own retention.
- [ ] You can spot a 20% MoM change between two inventory snapshots.

## 📚 Self-serve refresher

- [Inventory schema reference](https://learn.microsoft.com/azure/storage/blobs/blob-inventory#inventory-schema) — every field documented.
- [Power Query — Azure Blob Storage connector](https://learn.microsoft.com/power-query/connectors/azure-blob-storage) — for the Excel-based reports.

## 💰 Cost note

- Inventory rule: free to define.
- Each run: charged per object enumerated (~$0.002 per 10K objects). For a 50M-object DSR account, ~$10 per run.
- Output CSVs: regular blob storage cost.
- 90-day retention via lifecycle: also negligible.

**Cleanup:** disable your inventory rule (or leave it — it's nearly free). Delete the `inventory` container if you want a clean account.

---

⬅️ **Previous:** [Step 10 — Data protection posture](step-10-data-protection.md) _(Step 11 Immutability dropped)_
➡️ **Next:** [Step 14 — Application Gateway + WAF for operators](step-14-app-gateway-waf.md) _(Phase 3 begins; Step 13 Rosetta architecture dropped)_
