# Step 07 — Reading the lifecycle policy (Hot / Cool / Cold)

_The "the automation already does this — here's how to read it" lab._ ♨️➡️🧊➡️❄️

> [!NOTE]
> **Duration:** 60 minutes
> **Lab cost:** < NZD $1
> **Pairs with:** Module 7 of the training plan
> **Reframed at customer request:** "We won't be using these features — there are policies and scripts in place."

---

## 📖 Session overview

DIA already has automation that moves data between Hot, Cool, and Cold tiers. The Preservation Team doesn't write these policies — but they need to **read** them, **verify** they're working, and **spot when they misbehave**. This session focuses on those three skills, not on designing tier policies from scratch.

## 🎯 What you'll learn

- What a **lifecycle management policy** looks like in JSON
- How to **read** the existing DSR lifecycle policy and trace what it does
- How to **verify** a policy is actually moving data (not just configured)
- How to **monitor** lifecycle action counts and spot misbehaviour
- The cost impact of each tier transition (you do **not** pay to *move into* a tier; you pay per-blob transaction)

## 📚 Before this session — MS Learn pre-work

| Module | Time | Why this matters |
|---|---|---|
| [Optimize storage costs with lifecycle management](https://learn.microsoft.com/training/modules/optimize-archive-costs-blob-storage/) | 30 min | The standard MS Learn module — but skim, since you won't be authoring policies |
| [Hot, Cool, Cold, and Archive tiers](https://learn.microsoft.com/azure/storage/blobs/access-tiers-overview) (article, 15 min read) | 15 min | Per-tier costs and minimum retention rules |

## 🔤 Acronyms used

- **JSON** = JavaScript Object Notation — the format lifecycle policies are written in
- **TTL** = Time To Live (how long since last modification)

## ⏱️ EDE accounting

- Trainee self-paced: 60 min
- Instructor-led delivery: 1h
- Prep work: 45 min
- Q&A: 20 min
- **Total EDE per trainee: ~3h**

## 💰 Cost note

- One test account, ~10 small blobs, lifecycle simulated: < NZD $1.

---

## 🧱 Foundational primer

### Tier transition rules

- **Free transitions:** None. Each blob moved between tiers counts as 1 transaction (~NZD $0.0001 / 10k transactions for read; writes are similar).
- **Minimum stays:**
  - Cool: 30 days
  - Cold: 90 days
  - Archive: 180 days (DSR doesn't use Archive)
- **Reading** from Cool / Cold is **per-GB charged** — Cold reads are roughly 10x Cool reads.

### What a DSR-style policy looks like (typical)

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "anl-tier-down-rule",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": { "daysAfterModificationGreaterThan": 90 },
            "tierToCold": { "daysAfterModificationGreaterThan": 365 }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["stct-permrp-01/NLNZ_IE/"]
        }
      }
    }
  ]
}
```

This says: blobs in container `stct-permrp-01` under prefix `NLNZ_IE/` go to Cool after 90 days inactivity, then Cold after 365 days. **No restore-to-Hot rule** — once cooled, blobs stay cooled.

### Where DSR does NOT touch lifecycle

- **WOD storage account** has legal hold — blobs **must not be moved**. WOD's lifecycle policy is empty by design (immutability tag prevents modification).

---

## ⌨️ Activity 1 — Read the production lifecycle policy (read-only)

If you have Reader on DSR DEV/UAT/PRD:

1. Portal → `stanlnznblobprdrosi01` → **Lifecycle management**.
2. Click **List view** → see active rules.
3. Click any rule → **Code view** → see the JSON.
4. Copy the JSON to a scratchpad — you'll reference it.

> [!IMPORTANT]
> Don't click **Save** anywhere here. Read-only.

## ⌨️ Activity 2 — Trace what the policy does

For each rule, answer (in your notes):

- Which container/prefix does it target?
- After how many days does it move to Cool? Cold? Archive?
- Does it delete anything, or only tier-down?
- Are blob versions affected the same way?

In production, the answers are:
- Container: `stct-permrp-01` (and others)
- Prefixes: `NLNZ_IE/`, `NLNZ_File/`, `Metadata/`, `SIP/`, `ANZ_IE/`
- Cool after: typically 90–180 days
- Cold after: typically 365 days
- Delete: never (legal/compliance)
- Versions: tiered separately by version-specific rules

## ⌨️ Activity 3 — Deploy a lifecycle policy in your training sub

```bash
RG=rg-training-lifecycle-<your-initials>
az group create -n $RG -l australiaeast --tags purpose=dsr-training

SA=sttrainlc<your-initials>$RANDOM
az storage account create -n $SA -g $RG -l australiaeast \
  --sku Standard_LRS --kind StorageV2 --access-tier Hot

cat > /tmp/lifecycle.json <<'EOF'
{
  "rules": [
    {
      "enabled": true,
      "name": "anl-style-tier-down",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": { "daysAfterModificationGreaterThan": 30 },
            "tierToCold": { "daysAfterModificationGreaterThan": 90 }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["test-container/preservation/"]
        }
      }
    }
  ]
}
EOF

az storage account management-policy create \
  --account-name $SA \
  --resource-group $RG \
  --policy @/tmp/lifecycle.json
```

## ⌨️ Activity 4 — Verify the policy

```bash
az storage account management-policy show \
  --account-name $SA \
  --resource-group $RG \
  --output json
```

Inspect the JSON — confirm `enabled=true`, the prefix matches your intent, the day thresholds look right.

## ⌨️ Activity 5 — Upload test blobs

```bash
KEY=$(az storage account keys list -n $SA -g $RG --query '[0].value' -o tsv)

az storage container create --name test-container \
  --account-name $SA --account-key $KEY

# Upload 5 sample blobs
for i in 1 2 3 4 5; do
  echo "test data $i" > /tmp/blob-$i.txt
  az storage blob upload \
    --account-name $SA --account-key $KEY \
    --container-name test-container \
    --name preservation/blob-$i.txt \
    --file /tmp/blob-$i.txt
done

# Check current tiers
az storage blob list \
  --account-name $SA --account-key $KEY \
  --container-name test-container \
  --query "[].{name:name, tier:properties.blobTier}" \
  --output table
```

✅ All blobs are Hot.

## ⌨️ Activity 6 — Simulate the lifecycle (you can't time-travel 30 days, but…)

Lifecycle Management runs once every 24h. To verify the policy *would* trigger if blobs were old enough, you can manually move a blob:

```bash
az storage blob set-tier \
  --account-name $SA --account-key $KEY \
  --container-name test-container \
  --name preservation/blob-1.txt \
  --tier Cool

az storage blob list \
  --account-name $SA --account-key $KEY \
  --container-name test-container \
  --query "[].{name:name, tier:properties.blobTier, lastModified:properties.lastModified}" \
  --output table
```

✅ blob-1 is Cool, others remain Hot. Cost dropped immediately for blob-1.

## ⌨️ Activity 7 — Monitor lifecycle action counts (production check)

In production, you'd query Log Analytics for lifecycle activity. Preview:

```kusto
StorageBlobLogs
| where TimeGenerated > ago(7d)
| where OperationName in ("SetBlobTier","SetBlobAccessTier","TierChange")
| summarize ActionCount = count() by bin(TimeGenerated, 1d), DestinationTier = tostring(parse_json(Properties).destinationAccessTier)
| render columnchart
```

This is the chart your operations dashboard should show — daily count of tier-down events. A flatline = policy not running.

## ⌨️ Activity 8 — Tear down

```bash
az group delete -n $RG --yes --no-wait
```

---

## 🦾 Now your turn!

Take the JSON from the production lifecycle policy (Activity 1) and write a one-paragraph plain-English summary suitable for a status update email:

> "Our blob storage automatically moves preservation files to lower-cost tiers as they age. Files in the *X* prefix move to Cool after *Y* days and to Cold after *Z* days. The policy never deletes data and does not touch the WOD archive."

Save this in your notes — you'll re-use it in Module 23 (Reporting).

---

## ✅ Success checklist

- [ ] You can read a lifecycle policy JSON and explain what it does
- [ ] You've deployed a lifecycle policy and verified it's enabled
- [ ] You understand the minimum retention rules (30/90/180 days)
- [ ] You know WOD does NOT have lifecycle (legal hold)
- [ ] You know the KQL query that proves the policy is running

---

## 💰 Cost note

< NZD $1 if torn down promptly.

---

➡️ **Next:** [Step 08 — Azure Files: NFS vs SMB](step-08-azure-files.md)