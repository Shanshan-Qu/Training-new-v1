# Step 09 — Data protection posture

_The "what's protecting our data, and how do I prove it?" lab._ 🛡️

> [!NOTE]
> **Duration:** 90 minutes
> **Lab cost:** < NZD $1
> **Pairs with:** Module 9 of the training plan

---

## 📖 Session overview

DSR uses six layered data-protection features at the storage account level: **soft delete, snapshots, versioning, point-in-time restore, change feed, and blob inventory**. They overlap and complement each other. This session walks through each one — what it does, where it's enabled in DSR (per § 9.16 of the design), how to verify it's still on, and how to *consume* it (e.g. find a deleted blob's snapshot).

## 🎯 What you'll learn

- What each of the six features does and which DSR account uses what
- How **soft delete** retention windows work (Files 31 days; Blob 365 PRD / 31 UAT / 7 DEV)
- How **snapshots** are scheduled (daily 11 PM via Azure Automation Runbook) and retained (31 days)
- How **versioning** is configured ("Keep all Versions" in PRD)
- What **Point-in-Time Restore (PITR)** is — and that it works only for Hot/Cool, **not Cold**
- How to read the **Change Feed** for forensic queries
- How **Blob Inventory** generates weekly CSV reports (covered in detail in Module 11)

## 📚 Before this session — MS Learn pre-work

| Module | Time | Why this matters |
|---|---|---|
| [Protect your data with Azure Storage features](https://learn.microsoft.com/training/modules/protect-data-blob-storage/) | 30 min | Covers soft delete, versioning, PITR |
| [Manage Azure Blob Storage lifecycle](https://learn.microsoft.com/training/modules/manage-azure-blob-storage-lifecycle/) | 30 min | Lifecycle interaction with versioning |

## 🔤 Acronyms used

- **PITR** = Point-in-Time Restore

## ⏱️ EDE accounting

- Trainee self-paced: 90 min
- Instructor-led delivery: 1.5h
- Prep work: 1h
- Q&A: 30 min
- **Total EDE per trainee: ~4.5h**

## 💰 Cost note

- One Standard storage account with all six protections enabled: ~NZD $0.05/day
- Tear down at end → < NZD $1.

---

## 🧱 Foundational primer

### The six layers, mapped to DSR

| Feature | Files (`...file...`) | Blob Rosetta (`...blobprdrosi01`) | Blob WOD (`...blobprdwod01`) |
|---|---|---|---|
| Soft delete | 31 days | 365 days (PRD), 31 (UAT), 7 (DEV) | 365 days |
| Snapshots | Daily 11 PM, 31-day retention | Daily 11 PM, 31-day retention | Daily 11 PM, 31-day retention |
| Versioning | N/A (Files) | Keep all versions (PRD) | Keep all versions |
| Point-in-Time Restore | N/A | **Hot/Cool only** — not Cold | Not used (immutability covers it) |
| Change Feed | N/A | Enabled | Enabled |
| Blob Inventory | N/A | Weekly, 365-day retention | Weekly, 365-day retention |

### Soft delete vs versioning vs snapshots — what's different?

- **Soft delete** = "trash bin". Deleted blob is recoverable for N days.
- **Versioning** = every overwrite creates a new version; previous version is preserved automatically.
- **Snapshots** = manual or scheduled point-in-time copies.
- **PITR** = restore the *whole container* to a specific point in time. Different from per-blob recovery.
- **Change feed** = an append-only log of every blob operation. Forensic.
- **Blob inventory** = a periodic report of *what's in the account*. (Different from change feed = *what happened*.)

> [!IMPORTANT]
> PITR works only on **Hot or Cool** blobs. Once data has tiered to **Cold**, PITR cannot restore it. This is why DSR also keeps long soft-delete + versioning windows.

---

## ⌨️ Activity 1 — Inspect each protection in DSR (read-only)

If you have Reader on DSR DEV:

1. Portal → `stanlnznblobprdrosi01` (DEV equivalent) → **Data protection** blade.
2. For each toggle, note:
   - Soft delete: enabled, retention days
   - Versioning: enabled, retention
   - PITR: enabled, restore window
   - Change feed: enabled, retention
3. **Containers** → click any → **Snapshots** tab → see existing snapshots.

## ⌨️ Activity 2 — Deploy a fully-protected account (production-like)

```bash
RG=rg-training-protect-<your-initials>
az group create -n $RG -l australiaeast --tags purpose=dsr-training

SA=sttrainprot<your-initials>$RANDOM
az storage account create -n $SA -g $RG -l australiaeast \
  --sku Standard_LRS --kind StorageV2 --access-tier Hot

# Enable blob soft delete (7 days for lab)
az storage account blob-service-properties update \
  --account-name $SA -g $RG \
  --enable-delete-retention true \
  --delete-retention-days 7

# Enable container soft delete
az storage account blob-service-properties update \
  --account-name $SA -g $RG \
  --enable-container-delete-retention true \
  --container-delete-retention-days 7

# Enable versioning
az storage account blob-service-properties update \
  --account-name $SA -g $RG \
  --enable-versioning true

# Enable change feed (retention 7 days for lab)
az storage account blob-service-properties update \
  --account-name $SA -g $RG \
  --enable-change-feed true \
  --change-feed-retention-days 7

# Enable PITR (restore window must be < soft-delete window)
az storage account blob-service-properties update \
  --account-name $SA -g $RG \
  --enable-restore-policy true \
  --restore-days 6
```

## ⌨️ Activity 3 — Test soft delete

```bash
KEY=$(az storage account keys list -n $SA -g $RG --query '[0].value' -o tsv)

az storage container create -n preservation --account-name $SA --account-key $KEY
echo "important data" > /tmp/important.txt
az storage blob upload -c preservation -n important.txt -f /tmp/important.txt \
  --account-name $SA --account-key $KEY

# Delete it
az storage blob delete -c preservation -n important.txt \
  --account-name $SA --account-key $KEY

# Try to list — soft-deleted blobs are hidden by default
az storage blob list -c preservation \
  --account-name $SA --account-key $KEY \
  --output table

# List INCLUDING deleted
az storage blob list -c preservation \
  --account-name $SA --account-key $KEY \
  --include d \
  --output table

# Recover
az storage blob undelete -c preservation -n important.txt \
  --account-name $SA --account-key $KEY
```

✅ The blob is restored.

## ⌨️ Activity 4 — Test versioning

```bash
# Overwrite the blob 3 times
for i in 1 2 3; do
  echo "version $i" > /tmp/important.txt
  az storage blob upload -c preservation -n important.txt -f /tmp/important.txt \
    --account-name $SA --account-key $KEY --overwrite
done

# List versions
az storage blob list -c preservation -n important.txt \
  --account-name $SA --account-key $KEY \
  --include v \
  --query "[].{name:name, version:versionId, isCurrent:isCurrentVersion, modified:properties.lastModified}" \
  --output table
```

✅ You see 3 versions.

In the portal: Container → blob → **Versions** tab → list, download, promote to current.

## ⌨️ Activity 5 — Take a manual snapshot

```bash
az storage blob snapshot -c preservation -n important.txt \
  --account-name $SA --account-key $KEY

az storage blob list -c preservation -n important.txt \
  --account-name $SA --account-key $KEY \
  --include s \
  --query "[].{name:name, snapshot:snapshot}" \
  --output table
```

In production, this happens automatically via an Azure Automation Runbook scheduled at 23:00 NZST daily (per § 9.16.1.2).

## ⌨️ Activity 6 — Use Change Feed for forensics

Change Feed is an append-only log. Files appear under `$blobchangefeed/log/`.

```bash
# Wait ~5 min for change feed to populate, then:
az storage blob list -c '$blobchangefeed' \
  --account-name $SA --account-key $KEY \
  --output table
```

You'll see Avro files containing every blob operation. Tools like Azure Storage Explorer or `azure-storage-blobchangefeed` Python SDK parse them.

In production, this is how you'd answer "show me every delete on the WOD account in the last 24 hours" — useful for early-warning ransomware detection.

## ⌨️ Activity 7 — Test Point-in-Time Restore

```bash
# Note the current time, wait a minute
RESTORE_POINT=$(date -u -d '1 minute ago' +%Y-%m-%dT%H:%M:%SZ)

# Make some changes
echo "post-restore-point change" > /tmp/important.txt
az storage blob upload -c preservation -n important.txt -f /tmp/important.txt \
  --account-name $SA --account-key $KEY --overwrite

# Restore the container to the earlier point in time
az storage blob restore -g $RG --account-name $SA \
  --time-to-restore $RESTORE_POINT \
  --blob-range preservation/important.txt preservation/important.txt~~~
```

> [!IMPORTANT]
> PITR runs at the **container** level, not the blob level. You specify a range of blob names (alphabetical) to restore.

## ⌨️ Activity 8 — Tear down

```bash
az group delete -n $RG --yes --no-wait
```

---

## 🦾 Now your turn!

Write a Resource Graph query that lists all DSR storage accounts and shows whether each protection feature is **enabled** or **disabled**. Sample columns:

```
name | softDelete | versioning | changeFeed | pitr | inventory
```

Hint: blob service properties are at `microsoft.storage/storageaccounts/blobservices`.

This is your **weekly data-protection compliance check**.

---

## ✅ Success checklist

- [ ] You can name the six data-protection features and what each does
- [ ] You've deployed an account with all six enabled
- [ ] You've tested soft delete, versioning, snapshot, and PITR
- [ ] You've examined the change feed log structure
- [ ] You know PITR is Hot/Cool only — not Cold
- [ ] You have a saved compliance-check query

---

## 💰 Cost note

< NZD $1.

---

➡️ **Next:** [Step 10 — Immutability & legal hold](step-10-immutability.md)