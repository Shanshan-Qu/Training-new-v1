# Step 06 — Blob lifecycle management

_The "moving petabytes without lifting a finger" lab._ ♻️ Authors and reads lifecycle rules — the engine that automatically transitions Rosetta and WOD blobs between Hot, Cool, Cold, and Archive based on age and access pattern, without manual intervention.

> [!NOTE]
> **Trainee duration:** 90 minutes
> **Instructor EDE:** 3.5 hours (1h prep + 1.5h delivery + 1h Q&A buffer)
> **Lab cost:** under NZD $0.50 — reuses your storage account from Step 05.
> **Prerequisites:** Step 05 complete.
> **Pairs with:** Module 2 of the DIA training plan (Storage). **Lighter than originally proposed** — Emma's 28-Apr feedback noted the team already has policies and scripts for tier moves, so this lab focuses on **reading** existing rules and understanding cost effects, not authoring complex new rules from scratch.

---

## 📖 Session overview

DSR's preservation strategy lives partly in lifecycle rules: WOD content cools to Cold after 90 days, IE blobs cool to Cool after 180 days, archived material can move to Archive after years. Your team won't write new policies often — those are governance decisions that go through change control — but you **will** read them, debug "why didn't this blob move?" tickets, and explain cost effects to leadership. This lab gives you that fluency.

**What you'll learn**
- How **lifecycle rules** are structured (filters + actions).
- The four **tier transitions** and their per-GB / per-month cost effects.
- Why **`daysAfterModificationGreaterThan`** is the most common trigger and where it can mislead you (`daysAfterLastAccessTimeGreaterThan` is different).
- How to **simulate a lifecycle run** before applying it to production.
- The **interaction with versioning, soft delete, and snapshots** — these have their own lifecycle settings.
- How DSR's existing rules look in JSON and how to read them.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **Lifecycle rule** | A JSON policy attached to a storage account that automatically tiers or deletes blobs based on filters. |
| **Filter** | The "which blobs" — by prefix, blob type, blob index tags. |
| **Action** | The "what happens" — `tierToCool`, `tierToCold`, `tierToArchive`, `delete`. |
| **`daysAfterModificationGreaterThan`** | Days since last write. Most common trigger. |
| **`daysAfterLastAccessTimeGreaterThan`** | Days since last read — only works if **last access tracking is enabled** on the account (off by default; has a slight cost). |
| **Snapshot** | A read-only copy of a blob at a point in time. |
| **Version** | A new immutable copy of a blob created on every write when versioning is on. Treated like a snapshot for lifecycle. |
| **Rehydration** | Bringing a blob back from Archive to Hot/Cool — slow (hours) and per-GB chargeable. |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Optimize storage costs with lifecycle management](https://learn.microsoft.com/training/modules/optimize-blob-storage-costs/) | The exact pattern DSR uses for WOD and IE storage. |
| [Lifecycle management policy](https://learn.microsoft.com/azure/storage/blobs/lifecycle-management-overview) | The reference for every field in the JSON. |
| [Access tiers — comparison and pricing](https://learn.microsoft.com/azure/storage/blobs/access-tiers-overview) | Per-GB rates and minimum retention you must know. |

About **1.5 hours** of optional pre-reading.

## 🧱 Foundational primer

- **Lifecycle rules run once per day** (around 24 hours after a change). Don't expect instant action.
- **Tier transitions cost transactions.** Each blob moved is one write transaction. Across millions of blobs that adds up — usually still cheaper than the storage savings.
- **Minimum retention applies after a transition.** A blob moved to Cool must stay 30 days; to Cold must stay 90 days; to Archive must stay 180 days. Early deletion = early-deletion fee billed pro-rata.
- **Last-access tracking is OFF by default.** Without it, `daysAfterLastAccessTimeGreaterThan` rules behave like `daysAfterModificationGreaterThan`. Turn on tracking *before* relying on access-based rules.
- **Versions and snapshots have separate clauses.** A rule for `version` blobs is different from a rule for `baseBlob` blobs. WOD versioning blobs cool independently of base blobs.
- **Filter by prefix** uses the path including container name (e.g. `wod/2023/`). Test prefixes carefully — wrong prefix = wrong blobs moved.
- **Lifecycle never re-tiers up.** It can only move down (Hot → Cool → Cold → Archive) or delete. Bringing a blob back is a separate manual rehydrate action.

## ⌨️ Activity 1 — Read a DSR-style lifecycle policy

Here's a representative DSR rule (slightly simplified) for the WOD blob storage account:

```json
{
  "rules": [
    {
      "name": "wod-cool-after-90d",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["wod-content/"]
        },
        "actions": {
          "baseBlob": {
            "tierToCool": { "daysAfterModificationGreaterThan": 90 },
            "tierToCold": { "daysAfterModificationGreaterThan": 365 }
          },
          "version": {
            "tierToCold": { "daysAfterCreationGreaterThan": 90 },
            "delete":     { "daysAfterCreationGreaterThan": 730 }
          }
        }
      }
    }
  ]
}
```

Read it aloud:
1. Apply to all block blobs under `wod-content/` (only).
2. Move base blobs to Cool after 90 days since last modification.
3. Move base blobs to Cold after 365 days since last modification.
4. Move *versions* (older copies) to Cold 90 days after the version was created.
5. Delete versions older than 2 years.

> [!TIP]
> Always read base-blob rules and version rules separately. Versioning is what makes WOD storage cost-controlled — without that "delete versions after 2 years" line, every overwrite would keep growing.

## ⌨️ Activity 2 — Author a tiny lifecycle rule on your test account

1. Storage account → **Lifecycle management → + Add a rule**.
2. Rule name: `lab-cool-after-1`.
3. Scope: **Limit blobs with filters**.
4. Blob type: Block blobs. Subtype: Base blobs.
5. Filter set → Prefix match: `lab/`.
6. **Base blobs** → Move to Cool tier 1 day after last modified.
7. Review + add.

(We use 1 day so we can simulate it; production is 90+.)

## ⌨️ Activity 3 — Simulate the rule

There's no native "dry-run" but you can use the storage account's REST API to preview which blobs match.

Cloud Shell:

```bash
SA=<your storage account>

# List all blobs under the lab prefix with their last-modified date
az storage blob list \
  --account-name $SA \
  --container-name lab \
  --auth-mode login \
  --query "[].{name:name, lastModified:properties.lastModified, tier:properties.blobTier}" \
  -o table
```

Any blob with `lastModified` > 1 day ago will be transitioned the next day at the daily lifecycle run.

## ⌨️ Activity 4 — Manually tier a blob (compare to lifecycle outcome)

1. Storage account → **Containers → lab** → click your test blob.
2. **Change tier → Cool → Save**.
3. Refresh — note the **Access tier** column now says Cool.
4. Run the same `az storage blob list` from Activity 3 — `properties.blobTier` reflects the change.

This is the same operation lifecycle would perform automatically — you just did it on demand.

## ⌨️ Activity 5 — Calculate the cost effect (back-of-envelope)

Use today's NZN-region prices (approximate, in NZD; verify with Azure Pricing Calculator):

| Tier | Storage / GB / month | Read / 10K transactions | Read data / GB |
|---|---:|---:|---:|
| Hot | ~$0.03 | ~$0.01 | $0 |
| Cool | ~$0.018 | ~$0.02 | ~$0.018 |
| Cold | ~$0.006 | ~$0.13 | ~$0.045 |
| Archive | ~$0.003 | ~$8 | ~$0.075 |

For a 100 TB blob set:

| Tier | Storage cost / month |
|---|---:|
| Hot | ~$3,000 |
| Cool | ~$1,800 |
| Cold | ~$600 |
| Archive | ~$300 |

The savings are large — but read costs grow. Step 07 unpacks the retrieval economics.

> [!IMPORTANT]
> Whenever you propose changing a tier policy, attach the cost delta in NZD/month to the change request. Storage cost is the part of the cloud bill leadership notices.

## ⌨️ Activity 6 — Read tier statistics with a Resource Graph + KQL combo

For per-blob tier reporting we need Storage Insights metrics or Inventory CSVs (covered in Step 11). For now, use the storage **container metric**:

```kql
StorageBlobLogs
| where TimeGenerated > ago(1d)
| where AccountName == "<your storage account>"
| summarize Reads = countif(OperationName == "GetBlob"),
            Writes = countif(OperationName == "PutBlob")
            by AccountName, bin(TimeGenerated, 1h)
```

(Diagnostic settings → Storage logs → Log Analytics workspace must be configured. We'll set this up in Step 16.)

## 🦾 Now your turn!

1. Author a second rule on your storage account: delete blobs in container `temp/` after 7 days.
2. From Resource Graph, write a query returning every storage account in your subscription with the *number of lifecycle rules* (hint: `array_length(properties.policy.rules)` on the management policy resource type).
3. Find the JSON of your `lab-cool-after-1` rule via CLI: `az storage account management-policy show -n $SA -g $RG`.
4. Disable your rule (set `enabled: false`) without deleting it. Useful for "switch off the rule while we investigate" tickets.

## ✅ Success checklist

- [ ] You can read a lifecycle JSON rule and predict its effect.
- [ ] You can name the four tiers and the minimum retention each requires.
- [ ] You understand that lifecycle runs once per day and transitions are not instant.
- [ ] You've authored, simulated, and disabled a rule.
- [ ] You can explain the cost delta of cooling 100 TB of data from Hot to Cold.

## 📚 Self-serve refresher

- [Lifecycle policy JSON reference](https://learn.microsoft.com/azure/storage/blobs/lifecycle-management-policy-configure) — every field documented.
- [Pricing calculator](https://azure.microsoft.com/pricing/calculator/) — for cost-deltas in change requests.

## 💰 Cost note

- Lifecycle rules: free to define.
- Transactions: counted per blob moved (~$0.01 per 10,000).
- Lab storage: pennies.

**Cleanup:** delete your test rule when finished or leave it — disabled rules cost nothing.

---

⬅️ **Previous:** [Step 05 — Storage accounts deep-dive](step-05-storage-accounts-deep-dive.md)
➡️ **Next:** [Step 07 — Cold tier retrieval & rehydration cost](step-07-cold-retrieval.md)
