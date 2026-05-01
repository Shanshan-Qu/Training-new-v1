# Step 11 — Blob inventory

_The "what's actually in our storage?" lab._ 📋

> [!NOTE]
> **Duration:** 90 minutes
> **Lab cost:** < NZD $1
> **Pairs with:** Module 11 of the training plan

---

## 📖 Session overview

DSR runs **weekly Blob Inventory jobs** on every blob storage account (per § 9.16.1.6 of the design). Each job produces a CSV (or Parquet) listing every blob in the account — name, size, tier, last modified, encryption scope, lease status, and tags. The reports are kept for 365 days. They're the foundation for capacity-by-tier reporting, audit trails, and lifecycle verification.

This session teaches you how to *consume* the inventories that already exist — not how to set them up.

## 🎯 What you'll learn

- What the **Blob Inventory** report actually contains
- How to **find** the latest inventory CSV
- How to **read** an inventory CSV with KQL or Excel
- How to build a **capacity-by-tier** report from the inventory
- How to build a **capacity-by-prefix** report (Storage Group breakdown)
- How to **schedule** weekly delivery of inventory-derived reports

## 📚 Before this session — MS Learn pre-work

| Module | Time | Why this matters |
|---|---|---|
| [Use Azure Storage blob inventory](https://learn.microsoft.com/azure/storage/blobs/blob-inventory) (article) | 25 min | What inventories contain, format options |
| [Get started with KQL](https://learn.microsoft.com/training/modules/analyze-monitor-data-kusto-query-language/) | 30 min | If you didn't complete it for Module 16 yet |

## 🔤 Acronyms used

- **CSV** = Comma-Separated Values
- **Parquet** = a columnar binary format (smaller files, faster querying than CSV)

## ⏱️ EDE accounting

- Trainee self-paced: 90 min
- Instructor-led delivery: 1.5h
- Prep work: 55 min
- Q&A: 30 min
- **Total EDE per trainee: ~4.5h**

## 💰 Cost note

- One Standard storage account with inventory enabled: < NZD $1.

---

## 🧱 Foundational primer

### What an inventory job produces

Each weekly run writes a CSV (or Parquet) to a designated container. The schema is configurable — DSR's typical schema:

| Column | Example |
|---|---|
| `Name` | `stct-permrp-01/NLNZ_IE/2024/05/file.warc` |
| `Creation-Time` | 2024-05-12T11:23:47Z |
| `Last-Modified` | 2024-05-12T11:23:47Z |
| `Content-Length` | 1048576 (bytes) |
| `Access Tier` | Hot / Cool / Cold |
| `Access Tier Inferred` | true / false |
| `BlobType` | BlockBlob |
| `Encryption Scope` | (DSR uses CMK) |
| `Has Legal Hold` | true / false |
| `Lease Status` | unlocked |
| `Tags` | (custom blob tags) |

### Inventory location in DSR

Each blob storage account has a dedicated **inventory container** (e.g. `inventory-reports`). The job writes to:

```
{account}.blob.core.windows.net/inventory-reports/{rule-name}/{run-id}/
```

Retention: 365 days (PRD). Older reports are auto-deleted.

---

## ⌨️ Activity 1 — Inspect DSR's inventory configuration (read-only)

If you have Reader on DSR DEV/UAT/PRD:

1. Portal → `stanlnznblobprdrosi01` → **Blob inventory**.
2. **Inventory rules** → list active rules, schedule, target container, format.
3. Click any rule → see schema (columns) and prefix scope.
4. **Inventory reports** tab → see the latest run output.
5. Download a recent CSV (small if filtered by prefix).

## ⌨️ Activity 2 — Set up inventory on your training account

```bash
RG=rg-training-inv-<your-initials>
az group create -n $RG -l australiaeast --tags purpose=dsr-training

SA=sttraininv<your-initials>$RANDOM
az storage account create -n $SA -g $RG -l australiaeast \
  --sku Standard_LRS --kind StorageV2

KEY=$(az storage account keys list -n $SA -g $RG --query '[0].value' -o tsv)

# Source: where the data lives
az storage container create -n preservation --account-name $SA --account-key $KEY

# Destination: where the inventory CSV is written
az storage container create -n inventory-reports --account-name $SA --account-key $KEY

# Upload some sample blobs
for tier in hot cool cold; do
  for i in 1 2 3; do
    echo "sample-$tier-$i" > /tmp/$tier-$i.txt
    az storage blob upload -c preservation \
      -n preservation/$tier-prefix/sample-$i.txt -f /tmp/$tier-$i.txt \
      --account-name $SA --account-key $KEY
  done
done
```

Now configure inventory via portal (CLI for inventory is verbose):

1. Portal → your account → **Blob inventory** → **+ Add inventory rule**.
2. Name: `weekly-anl-style`
3. Container: `inventory-reports`
4. Object type: Blob
5. Schedule: **Daily** (lab — production uses weekly)
6. Format: **CSV**
7. Schema fields: select Name, Last-Modified, Content-Length, Access Tier, Has Legal Hold, BlobType
8. Filter prefix: leave blank (capture everything)
9. Save.

> [!TIP]
> The first run executes within ~24 hours of saving. For a faster lab, you can simulate the output below.

## ⌨️ Activity 3 — Read the inventory CSV (lab simulation)

If you can't wait 24h, here's a sample CSV inventory output to practise with:

```csv
Name,Last-Modified,Content-Length,Access Tier,Access Tier Inferred,BlobType,Has Legal Hold
preservation/hot-prefix/sample-1.txt,2026-05-02T01:00:00Z,16,Hot,true,BlockBlob,false
preservation/hot-prefix/sample-2.txt,2026-05-02T01:00:00Z,16,Hot,true,BlockBlob,false
preservation/cool-prefix/sample-1.txt,2026-05-01T01:00:00Z,16,Cool,false,BlockBlob,false
preservation/cool-prefix/sample-2.txt,2026-05-01T01:00:00Z,16,Cool,false,BlockBlob,false
preservation/cold-prefix/sample-1.txt,2026-04-30T01:00:00Z,16,Cold,false,BlockBlob,false
```

Save as `/tmp/inventory.csv`.

## ⌨️ Activity 4 — Capacity-by-tier (Excel)

Open `/tmp/inventory.csv` in Excel. Pivot table:

- Rows: Access Tier
- Values: Sum of Content-Length, Count of Name

Result:

```
Tier   | Total bytes | Blob count
Hot    | 32          | 2
Cool   | 32          | 2
Cold   | 16          | 1
```

This is your **capacity-by-tier weekly report**.

## ⌨️ Activity 5 — Capacity-by-prefix using KQL (advanced)

Upload the CSV to Log Analytics or query directly with `externaldata`:

```kusto
externaldata (Name:string, LastModified:datetime, ContentLength:long, AccessTier:string, AccessTierInferred:bool, BlobType:string, HasLegalHold:bool)
[
  h@"https://<SA>.blob.core.windows.net/inventory-reports/.../inventory.csv"
]
with(format='csv', ignoreFirstRecord=true)
| extend Prefix = tostring(split(Name, '/')[1])
| summarize TotalBytes = sum(ContentLength), BlobCount = count() by Prefix, AccessTier
| order by Prefix asc, AccessTier asc
```

Production use:

```kusto
| extend StorageGroup = case(
    Name startswith "stct-permrp-01/NLNZ_IE/", "NLNZ_IE",
    Name startswith "stct-permrp-01/NLNZ_File/", "NLNZ_File",
    Name startswith "stct-permrp-01/Metadata/", "Metadata",
    Name startswith "stct-permrp-01/SIP/", "SIP",
    Name startswith "stct-permrp-01/ANZ_IE/", "ANZ_IE",
    "Other")
| summarize TotalGB = sum(ContentLength) / 1024 / 1024 / 1024, BlobCount = count() by StorageGroup, AccessTier
```

✅ This breaks down DSR by Storage Group and tier — exactly what you'd email weekly.

## ⌨️ Activity 6 — Verify legal hold from inventory

Inventory includes the `Has Legal Hold` column — use it as a backup verification of Module 10's hold check:

```kusto
... 
| where HasLegalHold == false
| where Name startswith "wod-archive/"
| count
```

Result should always be **0**. Anything else = ALERT.

## ⌨️ Activity 7 — Schedule weekly delivery

Two options:

| Method | Setup | Best for |
|---|---|---|
| Logic App | Watches the inventory container, processes the CSV, emails a summary | Code-light, easy to maintain |
| Azure Automation Runbook | PowerShell script, scheduled weekly | Fits DSR's existing snapshot Runbook pattern |
| Power BI | Connects to the inventory CSV, refreshes weekly, embedded in Teams | Visual reports |

In production, DSR uses the Runbook pattern. The Preservation Team subscribes to its email output.

## ⌨️ Activity 8 — Tear down

```bash
az group delete -n $RG --yes --no-wait
```

---

## 🦾 Now your turn!

Build the **weekly inventory consumer** — pseudo-script:

1. Find the latest CSV in `inventory-reports/`
2. Compute capacity-by-tier and capacity-by-storage-group
3. Compare to last week (delta)
4. Flag if any tier shifted unexpectedly (>20% change)
5. Email the team

The exact implementation depends on language — but you should sketch the logic.

---

## ✅ Success checklist

- [ ] You've inspected DSR's inventory rules
- [ ] You've configured an inventory rule on your training account
- [ ] You've read an inventory CSV in Excel and KQL
- [ ] You've built capacity-by-tier and capacity-by-prefix reports
- [ ] You've used inventory to verify legal hold
- [ ] You know how the weekly delivery is scheduled in DSR

---

## 💰 Cost note

< NZD $1. Inventory writes are tiny (a few MB per week per account).

---

➡️ **Next:** [Step 12 — Rosetta architecture walkthrough](step-12-rosetta-architecture.md)