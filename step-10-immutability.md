# Step 10 — Immutability & legal hold

_The "WOD must never change" lab._ 🔒

> [!NOTE]
> **Duration:** 75 minutes
> **Lab cost:** < NZD $1
> **Pairs with:** Module 10 of the training plan

---

## 📖 Session overview

The **Whole-of-Domain (WOD) archive** is held under a legal hold tag named `WholeOfDomainArchiveHold` (per § 9.15 of the design). Loss of this hold = compliance breach. This session covers immutability — what it is, the difference between **time-based retention** and **legal hold**, how to apply both, and how to verify the WOD legal hold is still in place. The last point is critical: regular verification is the Preservation Team's responsibility.

## 🎯 What you'll learn

- The difference between **time-based retention** and **legal hold**
- Container-level vs version-level immutability scope
- How to **apply** a legal hold tag
- How to **verify** a legal hold is in place (the weekly check)
- What happens when someone tries to delete a legally-held blob
- The escalation path if a hold is found removed
- How DSR's WOD account uses this exact pattern

## 📚 Before this session — MS Learn pre-work

| Module | Time | Why this matters |
|---|---|---|
| [Configure immutable storage](https://learn.microsoft.com/azure/storage/blobs/immutable-storage-overview) (article) | 25 min | Concepts and terminology |
| [Manage immutable blobs](https://learn.microsoft.com/azure/storage/blobs/immutable-policy-configure-version-scope) (article) | 25 min | The CLI/PowerShell commands you'll use |

## 🔤 Acronyms used

- **WORM** = Write Once Read Many (the regulatory term)

## ⏱️ EDE accounting

- Trainee self-paced: 75 min
- Instructor-led delivery: 1.25h
- Prep work: 1h
- Q&A: 30 min
- **Total EDE per trainee: ~4h**

## 💰 Cost note

- < NZD $1.

---

## 🧱 Foundational primer

### Two flavours of immutability

| Type | Stops what? | Removable? |
|---|---|---|
| **Time-based retention** | Delete and modify until N days have passed | Auto-expires; cannot be removed early in `Locked` state |
| **Legal hold** | Delete and modify forever, until tag removed | Removable by anyone with `Storage Blob Data Owner` |

WOD uses **legal hold** (indefinite, removable only by authorised actors).

### Scope: container vs version

- **Container-scoped** (legacy): the policy applies to all blobs in the container.
- **Version-scoped** (newer, recommended for WOD): the policy applies to specific blob versions; supports versioning + immutability together.

DSR uses version-scoped policies for WOD.

### What "you can't modify or delete" actually means

Operations blocked when a hold is in place:

- Delete blob
- Delete container
- Overwrite blob (Put Blob, Put Block List)
- Set blob tier (in some configurations)
- Snapshot delete (snapshots inherit hold)

Operations still allowed:

- Read blob
- List blob
- Set blob metadata (in some configurations)
- Snapshot create

---

## ⌨️ Activity 1 — Inspect WOD's legal hold (read-only)

If you have Reader on DSR DEV/UAT/PRD:

1. Portal → `stanlnznblobprdwod01` → **Containers** → click the WOD container.
2. **Access policy** tab → see the legal hold tag.
3. Confirm: tag = `WholeOfDomainArchiveHold`, status = enabled.

If you do not have Reader, skip to Activity 2.

## ⌨️ Activity 2 — Deploy a WOD-style account in your training sub

```bash
RG=rg-training-immut-<your-initials>
az group create -n $RG -l australiaeast --tags purpose=dsr-training

SA=sttrainwod<your-initials>$RANDOM
az storage account create -n $SA -g $RG -l australiaeast \
  --sku Standard_LRS --kind StorageV2

KEY=$(az storage account keys list -n $SA -g $RG --query '[0].value' -o tsv)
az storage container create -n wod-archive --account-name $SA --account-key $KEY

# Upload a sample blob
echo "archived web crawl" > /tmp/crawl.txt
az storage blob upload -c wod-archive -n crawl.txt -f /tmp/crawl.txt \
  --account-name $SA --account-key $KEY
```

## ⌨️ Activity 3 — Apply a legal hold tag

```bash
az storage container legal-hold set \
  --account-name $SA --account-key $KEY \
  --container-name wod-archive \
  --tags WholeOfDomainArchiveHold

# Verify
az storage container legal-hold show \
  --account-name $SA --account-key $KEY \
  --container-name wod-archive \
  --output table
```

✅ Tag applied. The container is now in legal hold.

## ⌨️ Activity 4 — Try to delete a held blob

```bash
az storage blob delete -c wod-archive -n crawl.txt \
  --account-name $SA --account-key $KEY
```

❌ This fails:

```
The blob has an active legal hold and cannot be modified.
```

✅ The hold works.

## ⌨️ Activity 5 — Try to overwrite a held blob

```bash
echo "tampered content" > /tmp/crawl.txt
az storage blob upload -c wod-archive -n crawl.txt -f /tmp/crawl.txt \
  --account-name $SA --account-key $KEY --overwrite
```

❌ Same — fails. Immutability blocks overwrite.

## ⌨️ Activity 6 — The weekly hold-verification check (production)

This is the check that should run every Monday, alerting the Preservation Team if the hold is missing:

```bash
RESULT=$(az storage container legal-hold show \
  --account-name $SA --account-key $KEY \
  --container-name wod-archive \
  --query "tags[?contains(@,'WholeOfDomainArchiveHold')] | length(@)" -o tsv)

if [ "$RESULT" -ge "1" ]; then
  echo "OK: WholeOfDomainArchiveHold present"
else
  echo "ALERT: hold missing — escalate to Cloud Security"
fi
```

Wrap this in an Azure Automation Runbook and schedule weekly. Or:

In Log Analytics:

```kusto
AzureActivity
| where TimeGenerated > ago(7d)
| where OperationNameValue contains "legalHold"
| project TimeGenerated, Caller, OperationNameValue, ActivityStatusValue, Properties
```

This shows any legal-hold-related activity in the last 7 days. Any unexpected `Set` or `Delete` event = investigation.

## ⌨️ Activity 7 — Time-based retention (for comparison)

WOD uses legal hold, but version-scoped time-based is good to understand:

```bash
# Enable versioning first (required for version-scoped policy)
az storage account blob-service-properties update \
  --account-name $SA -g $RG --enable-versioning true

# Apply a 7-day version-scoped policy in unlocked state
az storage container immutability-policy create \
  --account-name $SA --resource-group $RG \
  --container-name wod-archive \
  --period 7 \
  --allow-protected-append-writes false
```

Inspect in portal: container → **Access policy** → see the immutability policy.

> [!IMPORTANT]
> An **unlocked** time-based policy can be removed. A **locked** policy cannot — only expires by time. WOD's policy must be locked in production.

## ⌨️ Activity 8 — Remove the legal hold (cleanup only)

```bash
az storage container legal-hold clear \
  --account-name $SA --account-key $KEY \
  --container-name wod-archive \
  --tags WholeOfDomainArchiveHold

# Now delete works
az storage blob delete -c wod-archive -n crawl.txt \
  --account-name $SA --account-key $KEY

az group delete -n $RG --yes --no-wait
```

---

## 🦾 Now your turn!

Build the **weekly hold-verification report** as a saved KQL query that:

- Confirms the WOD legal hold tag is active on `stanlnznblobprdwod01`
- Lists any legal-hold modification activity in the last 7 days
- Flags as ALERT if any `Clear` or `SetLegalHold` event is found from outside DIA Cloud Security or Storage Owner accounts

Schedule this query as a **Log Analytics alert** with email notification to the Preservation Team.

---

## ✅ Success checklist

- [ ] You can name the difference between time-based retention and legal hold
- [ ] You've applied and removed a legal hold in your lab
- [ ] You've verified that delete and overwrite are blocked
- [ ] You've written the weekly hold-verification check
- [ ] You know the escalation path if the hold is found missing

---

## 🚨 Escalation path — hold found missing

1. Confirm via CLI (Activity 6) that the hold is in fact removed (not just a UI glitch).
2. Pull the Activity Log for the storage account for the last 14 days.
3. Identify who removed the hold (`Caller` field).
4. **Escalate immediately** to DIA Cloud Security and the Archives Library Manager.
5. Re-apply the hold *only* under explicit instruction.

---

## 💰 Cost note

< NZD $1.

---

➡️ **Next:** [Step 11 — Blob inventory](step-11-blob-inventory.md)