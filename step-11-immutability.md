# Step 11 — Immutability & legal hold

_The "this blob shall not be deleted, by anyone, ever" lab._ ⚖️ Builds the WORM (write-once-read-many) policy fluency the team needs for compliance: time-based retention, legal holds, applying and verifying holds, and recognising what an audit needs.

> [!NOTE]
> **Trainee duration:** 75 minutes
> **Lab cost:** under NZD $0.50 — reuses your test storage account.
> **Prerequisites:** Steps 05 + 06 + 09 complete.
> **Pairs with:** Module 2 of the DIA training plan (Storage).

---

## 📖 Session overview

Some of DIA's preserved content is subject to legal hold or statutory retention — items must remain unalterable for years or indefinitely, even by Owner-level identities. Azure achieves this through **immutability policies** at the container or blob version level. This lab walks the two flavours (time-based retention + legal hold), shows how to apply and verify them, and how to recognise the impact on lifecycle and deletion operations.

**What you'll learn**
- The difference between **time-based retention** and **legal hold**.
- The two **scope levels**: container-level (legacy) and **version-level** (current).
- How a policy progresses through **Unlocked** → **Locked** state and what that means.
- How retention interacts with versioning, lifecycle, and soft delete.
- How to **verify** a hold is in place (the audit team's question).
- DSR's pattern: container-level time-based retention with legal hold for active investigations.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **WORM** | Write Once Read Many — once written, can't be modified or deleted. |
| **Time-based retention** | Blob is immutable for N days from creation/version date. After that, deletable. |
| **Legal hold** | Blob is immutable as long as the hold tag exists. Has no automatic expiry. |
| **Container-level policy** | Applied to the whole container; affects every blob inside. Legacy approach. |
| **Version-level policy** | Applied per blob version; finer control. Current best practice. |
| **Unlocked policy** | Reversible — can be modified or removed. Used during testing. |
| **Locked policy** | Permanent — once locked, retention can only be **extended**, never shortened or removed. |
| **Hold tag** | A short string label identifying the legal hold (e.g. `case-12345`, `audit-2026`). |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Immutable storage for blobs](https://learn.microsoft.com/azure/storage/blobs/immutable-storage-overview) | The exact mechanism DSR uses. |
| [Configure immutability policies](https://learn.microsoft.com/azure/storage/blobs/immutable-policy-configure-version-scope) | Step-by-step for version-level policies. |
| [Immutable storage for Azure Blob Storage](https://learn.microsoft.com/azure/storage/blobs/immutable-storage-overview) | DIA's audit/compliance reference. |

About **1.5 hours** of optional pre-reading.

## 🧱 Foundational primer

- **Time-based + legal hold can coexist.** A blob can have a 5-year retention *and* a legal hold; both must be released for the blob to become deletable.
- **A locked policy cannot be shortened or removed.** Even Owner cannot. This is the point of the feature.
- **Immutability survives subscription cancellation.** Microsoft cannot delete the data while a locked policy is active.
- **Lifecycle rules cannot delete immutable blobs.** A "delete after 1 year" rule will skip blobs whose retention hasn't elapsed.
- **Versioning + immutability is the strongest pairing.** Every version is independently immutable.
- **Legal hold is the operational tool**: cheap, applies in seconds, removable when the case closes. Use this for active matters.
- **Time-based is the policy tool**: applied at scope, doesn't change as cases come and go.

## ⌨️ Activity 1 — Container-level legal hold

1. Storage account → **Containers → + Container → Name: legal-test**.
2. Open `legal-test` → **Access policy → + Add policy → Legal hold**.
3. Tag: `lab-test`. Save.
4. Upload a small file to the container. Try to delete it. **Error: "Cannot delete blob because legal hold is set."**

## ⌨️ Activity 2 — Remove the legal hold

1. Same container → **Access policy → Legal hold → Remove**. (Tag `lab-test` removed.)
2. Try to delete the blob again. Now it works.

## ⌨️ Activity 3 — Time-based retention (container-level)

1. **+ Container → Name: retention-test**.
2. **Access policy → + Add policy → Time-based retention → 7 days**. Save.
3. Upload a blob. Try to delete. **Error: blob protected.**
4. Note the **policy state: Unlocked**. Click **... → Edit policy** — you can change it (this is testing mode).
5. Try **Lock policy**. Once locked, you can extend retention but not shorten it.

> [!IMPORTANT]
> Production retention policies are always locked. Only DIA Platform / Compliance can apply locked retention to production storage. **Don't lock anything in lab unless you're certain — locked = irreversible.**

## ⌨️ Activity 4 — Version-level immutability

This is the modern approach.

1. Confirm versioning is on (Step 10 enabled it).
2. Storage account → **Data protection → Version-level immutability**: **Enabled by default**. (You may need to first re-create the container with version-level immutability allowed.)
3. New container `version-immut-test` with version-level immutability allowed at creation.
4. Upload a blob. Click the blob → **Manage policy → + Add policy** → set retention 7 days, leave Unlocked.
5. Try to delete the blob's *current* version. Blocked.
6. Older / newer versions can have separate policies. This is the point.

## ⌨️ Activity 5 — Verify a hold is in place (audit query)

```kql
resources
| where type == "microsoft.storage/storageaccounts/blobservices/containers"
| extend acct = tostring(split(id, "/")[8]),
         container = name
| project acct, container,
          legalHold = properties.hasLegalHold,
          immutability = properties.hasImmutabilityPolicy
| where legalHold == true or immutability == true
```

Run against DSR subs. Every result is a "this container has a hold" — exactly what the audit team needs.

## ⌨️ Activity 6 — Apply a legal hold via CLI (operational pattern)

```bash
SA=<your storage account>

# Add a legal hold
az storage container legal-hold set \
  --account-name $SA \
  --container-name legal-test \
  --tags case-12345 audit-2026 \
  --auth-mode login

# Verify
az storage container legal-hold show \
  --account-name $SA \
  --container-name legal-test \
  --auth-mode login

# Remove (when case closes)
az storage container legal-hold clear \
  --account-name $SA \
  --container-name legal-test \
  --tags case-12345 audit-2026 \
  --auth-mode login
```

In production this is the operational runbook for "investigation X has started" / "case X closed".

## ⌨️ Activity 7 — Interaction with lifecycle

1. Author a lifecycle rule on `legal-test` that says "delete after 1 day".
2. Apply a legal hold (Activity 6).
3. Wait a day (or simulate). The lifecycle rule will run but **skip** the immutable blob.
4. Remove the hold. The next lifecycle run will delete the blob.

This is exactly how DSR retains material under hold even when standard lifecycle would normally clean it.

## 🦾 Now your turn!

1. Apply both a time-based retention policy *and* a legal hold to the same blob version. Verify both are present in `az storage blob show`.
2. Find a Resource Graph query that returns every container with a *locked* immutability policy in the subscription. (Hint: `properties.immutableStorageWithVersioning` — keys vary by API version.)
3. Write a one-paragraph SOP for "applying a legal hold for a Police request": which container, which tag format, who approves, how to verify.
4. Read the storage account's **Activity Log** (Step 03) for any `setLegalHold` events. Who applied them?

## ✅ Success checklist

- [ ] You can name the two immutability flavours (time-based vs legal hold) and explain the difference.
- [ ] You've applied and removed both kinds of hold.
- [ ] You understand "Unlocked" vs "Locked" and why production is always locked.
- [ ] You can verify a hold exists via Portal, CLI, and Resource Graph.
- [ ] You understand that lifecycle rules skip immutable blobs.

## 📚 Self-serve refresher

- [Immutability scope and migration](https://learn.microsoft.com/azure/storage/blobs/immutable-storage-overview) — when to use container vs version scope.
- [Manage legal holds](https://learn.microsoft.com/azure/storage/blobs/immutable-legal-hold-overview) — operational reference.

## 💰 Cost note

- Immutability is **free**.
- Indirect cost: data under hold cannot be lifecycle-deleted, so storage continues until the hold is released.

**Cleanup:** before deleting any test container, **remove all holds first**. The portal will refuse to delete a container with a locked policy — by design.

---

⬅️ **Previous:** [Step 10 — Data protection posture](step-10-data-protection.md)
➡️ **Next:** [Step 12 — Blob inventory & capacity reporting](step-12-blob-inventory.md)
