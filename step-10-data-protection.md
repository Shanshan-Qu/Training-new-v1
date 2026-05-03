# Step 10 — Data protection posture

_The "what stops accidental deletion" lab._ 🛡️ Builds the recoverability mental model: snapshots, soft delete, versioning, point-in-time restore, change feed — and how to *prove* a storage account is properly protected.

> [!NOTE]
> **Trainee duration:** 90 minutes
> **Lab cost:** under NZD $0.50 — reuses your test storage account.
> **Prerequisites:** Steps 05 + 06 complete.
> **Pairs with:** Module 2 of the DIA training plan (Storage). **Note:** backup *configuration* is owned by DIA Platform Team, but the Preservation Team owns *posture awareness* — you'll be asked "is X protected?" and need to be able to read the answer.

---

## 📖 Session overview

DSR runs a multi-layer data protection model. Some layers (Recovery Services Vault backup of files / VMs) are owned by DIA Platform; others (blob soft delete, versioning, point-in-time restore) sit on the storage account itself and are part of your daily read. This lab walks every layer of storage-account-level protection, shows you how to verify it's enabled, and how to *use* it to recover an accidentally deleted blob — read-only on production.

**What you'll learn**
- The five storage-account protection features: **soft delete (blob)**, **soft delete (container)**, **versioning**, **change feed**, **point-in-time restore**.
- How each one helps recovery — and what each one **does not** protect against.
- How to verify a storage account is protected (one Resource Graph query for the audit team).
- How to recover a deleted blob using soft delete or versioning.
- The interaction with lifecycle rules (versions cost money — you've seen this in Step 07).

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **Soft delete (blob)** | Deleted blobs are kept hidden for N days before permanent deletion. Recoverable by you. |
| **Soft delete (container)** | Deleted containers retained for N days. (Recovers the *container* including its blobs.) |
| **Versioning** | Every overwrite creates a new immutable copy of the blob; previous versions remain queryable. |
| **Change feed** | A log of every change to the storage account, kept as ordered append blobs. Useful for forensic queries and incremental ingest. |
| **Point-in-time restore (PITR)** | Roll the *whole* storage account (or container range) back to a point in time. Requires versioning + change feed + soft delete enabled first. |
| **Snapshot (blob)** | A read-only copy of a blob taken explicitly. Older mechanism; mostly superseded by versioning. |
| **Recovery Services Vault** | Azure's separate backup service. Protects VMs, file shares, SQL. **Not** what protects blobs — blob protection lives on the account. |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Data protection overview for Blob Storage](https://learn.microsoft.com/azure/storage/blobs/data-protection-overview) | The exact protection layers DSR uses. |
| [Point-in-time restore for block blobs](https://learn.microsoft.com/azure/storage/blobs/point-in-time-restore-overview) | The "roll back the account" feature. |
| [Blob change feed](https://learn.microsoft.com/azure/storage/blobs/storage-blob-change-feed) | What feeds the WOD audit and inventory pipelines. |

About **1.5 hours** of optional pre-reading.

## 🧱 Foundational primer

- **Soft delete is the first line of defence.** It's cheap, automatic, and undoes most accidents.
- **Versioning protects against overwrite, not delete.** Without soft delete, a deletion still removes all versions.
- **PITR requires the full stack** (versioning, change feed, blob soft delete) and supports up to **365 days** lookback.
- **Soft-deleted data is still billed** at the regular tier rate for the retention period. WOD versioning + soft delete is part of why the storage line item looks the way it does.
- **Change feed is append-only.** The history is immutable; great for audit and reporting.
- **Account-level snapshots / backup with Azure Backup** is for the VMs, not for blob data. Don't confuse the two.
- **Restoring is not free.** PITR transactions and rehydration apply. Cost-estimate before invoking.

## ⌨️ Activity 1 — Inspect data protection posture across all accounts

```kql
resources
| where type == "microsoft.storage/storageaccounts/blobservices"
| extend acct = tostring(split(id, "/")[8])
| project acct,
          softDelete = properties.deleteRetentionPolicy,
          containerSoftDelete = properties.containerDeleteRetentionPolicy,
          versioning = properties.isVersioningEnabled,
          changeFeed = properties.changeFeed.enabled,
          restorePolicy = properties.restorePolicy
```

You'll see every storage account's protection state in one view. In DSR, the prod accounts should show: blob soft delete 14 days, container soft delete 14 days, versioning enabled, change feed enabled, restore policy enabled (90 days).

## ⌨️ Activity 2 — Enable the full protection stack on your test account

1. Storage account → **Data protection** blade.
2. Enable: **Soft delete for blobs (14 days)**, **Soft delete for containers (14 days)**, **Versioning**, **Change feed**, **Point-in-time restore (7 days for the lab)**.
3. Save.

## ⌨️ Activity 3 — Recover a soft-deleted blob

1. Container `lab` → click your test blob from Step 06 → **Delete** (single blob).
2. Top of the Containers list → toggle **Show deleted blobs**.
3. The blob appears with status **Deleted, X days remaining**. Right-click → **Undelete**.
4. Blob is restored.

## ⌨️ Activity 4 — Recover a soft-deleted container

1. **+ Container → Name: doomed**. Upload one blob inside.
2. Delete the container.
3. **Containers** blade → **Show deleted containers** toggle.
4. Right-click `doomed` → **Undelete**.
5. The container reappears with its blob intact.

## ⌨️ Activity 5 — Versioning in action

1. In container `lab`, upload a small text file `notes.txt`.
2. Edit a copy locally (change one character) and upload again, replacing.
3. Click `notes.txt` → **Versions** tab. You see two versions, each with a timestamp.
4. Click the older version → **Make current version**. The blob now reflects the older content.

## ⌨️ Activity 6 — Read the change feed

The change feed lives at `$blobchangefeed/log/...`. You can list entries via CLI:

```bash
SA=<your storage account>
az storage blob list \
  --account-name $SA \
  --container-name '$blobchangefeed' \
  --auth-mode login \
  --query "[].name" -o tsv | head -10
```

Each segment is an Avro file you can stream or download. WOD's audit pipeline reads these to know what changed since last run.

## ⌨️ Activity 7 — Point-in-time restore (read-only walkthrough)

1. Storage account → **Data protection → Point-in-time restore for containers → Restore**.
2. The blade asks: which containers, which timestamp, which destination.
3. Don't click Restore (it costs and we don't have meaningful data to restore). The point is recognise the UI and the inputs.

> [!IMPORTANT]
> PITR is for **bulk** recovery — e.g. someone ran a delete script. For single blobs, soft delete + versioning is faster and cheaper.

## ⌨️ Activity 8 — Resource Graph audit query (compliance)

The audit team will ask: "Show me every storage account in DSR with versioning *not* enabled."

```kql
resources
| where type == "microsoft.storage/storageaccounts/blobservices"
| extend acctName = tostring(split(id, "/")[8])
| extend versioning = tobool(properties.isVersioningEnabled),
         softDelete = tobool(properties.deleteRetentionPolicy.enabled)
| where versioning == false or softDelete == false
| project acctName, versioning, softDelete, id
```

Every result is a finding — escalate to whoever owns that storage account.

## 🦾 Now your turn!

1. Tweak your soft delete to 7 days. Write the CLI command (`az storage account blob-service-properties update`) that achieves this.
2. From the change feed, find the entry corresponding to your `notes.txt` overwrite in Activity 5. (Hint: download the Avro segment and inspect with `avro-tools` or a Python script.)
3. Disable versioning temporarily, then re-enable. What happens to existing versions? (Spoiler: they remain.)
4. Find the cost of soft-deleted storage in Cost Management. Group by **Meter** = `Soft Delete`.

## ✅ Success checklist

- [ ] You can name the five blob protection features and what each protects against.
- [ ] You've recovered a soft-deleted blob and a soft-deleted container.
- [ ] You can roll back a blob to a previous version.
- [ ] You can write a Resource Graph query that finds storage accounts with insufficient protection.
- [ ] You understand that PITR requires the full stack (versioning + change feed + soft delete).

## 📚 Self-serve refresher

- [Soft delete for blobs](https://learn.microsoft.com/azure/storage/blobs/soft-delete-blob-overview) — exact retention rules.
- [Versioning vs soft delete vs snapshots — when to use each](https://learn.microsoft.com/azure/storage/blobs/data-protection-overview) — picking the right tool.

## 💰 Cost note

- Soft delete: regular storage rates × retention period for any deleted blob.
- Versioning: regular rates × every version retained. Pair with lifecycle rules to expire old versions.
- Change feed: append blobs are very cheap.
- PITR: free to enable; costs only on actual restore.

**Cleanup:** clean up `doomed` again if you re-deleted it, and remove `notes.txt` versions if you don't need them. Or leave the lot — the costs are negligible until you hit GBs of versions.

---

⬅️ **Previous:** [Step 09 — Azure Files (NFS & SMB)](step-09-azure-files.md)
➡️ **Next:** [Step 11 — Immutability & legal hold](step-11-immutability.md)
