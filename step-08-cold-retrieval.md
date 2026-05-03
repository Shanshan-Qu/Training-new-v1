# Step 08 — Cold tier retrieval & rehydration cost

_The "why was last month's bill so high?" lab._ 💸 Builds the retrieval-economics intuition the team needs: per-GB read costs at Cold and Archive, rehydration priorities, billing surprises, and how to communicate a retrieval plan to leadership before it happens.

> [!NOTE]
> **Trainee duration:** 75 minutes
> **Instructor EDE:** 3.25 hours (1h prep + 1.25h delivery + 1h Q&A buffer)
> **Lab cost:** under NZD $0.50 — a few small Cold-tier blobs and a single rehydration.
> **Prerequisites:** Steps 05 + 06 complete.
> **Pairs with:** Module 2 of the DIA training plan (Storage).

---

## 📖 Session overview

The DIA digital preservation estate has a long-tail access pattern: most preserved content is read rarely, but when it *is* read — for a researcher request, an audit, a legal discovery — it can be tens of gigabytes at once. Cold and Archive tiers make storage cheap, but **retrieval** is where the bill jumps. This lab makes you fluent in retrieval cost so you can sanity-check before approving a request and explain the bill afterwards.

**What you'll learn**
- The retrieval cost model: per-GB read fees + transaction fees + (for Archive) rehydration.
- The three **rehydration priorities** for Archive: Standard, High, and "no good cheap option".
- How to **estimate the cost of a retrieval** before doing it.
- How to **read recent retrieval activity** in Cost Management and Storage logs.
- The minimum-retention "early deletion fee" pitfall.
- How DSR's reporting picks up retrieval spikes.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **Read cost / data retrieval fee** | Per-GB charge for reading data out of Cool/Cold/Archive (Hot is free per GB). |
| **Transaction cost** | Per-operation charge — listing, GET, PUT each cost a tiny amount. |
| **Rehydration** | Process of moving a blob from Archive (offline) back to Hot/Cool (online). Hours, not seconds. |
| **Standard priority** | Up to 15 hours rehydration, default cost. |
| **High priority** | Under 1 hour for blobs <10 GB, ~10× the cost. |
| **Early deletion fee** | If you remove a blob before its tier's minimum retention (Cool 30d, Cold 90d, Archive 180d), you're charged for the missing days. |
| **Egress** | Outbound data leaving the Azure region (to internet, on-prem, or another region). Has its own cost separate from read. |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Manage Azure Blob Storage lifecycle](https://learn.microsoft.com/training/modules/optimize-blob-storage-costs/) | Sets up tier-cost intuition. |
| [Rehydrate a blob from Archive](https://learn.microsoft.com/azure/storage/blobs/archive-rehydrate-overview) | The exact mechanism this lab covers. |
| [Plan and manage costs for Azure Blob Storage](https://learn.microsoft.com/azure/storage/blobs/storage-blob-cost-optimization) | Microsoft's authoritative cost guide. |

About **1 hour** of optional pre-reading.

## 🧱 Foundational primer

- **Hot tier reads are free per GB.** You only pay transactions and egress (if data leaves the region).
- **Cool tier costs ~$0.018/GB to read.** A 1 TB read = ~$18. Multiply by frequency to see whether Cool is right.
- **Cold tier costs ~$0.045/GB to read.** A 1 TB read = ~$45. Cold is for "we'll likely never read this" data.
- **Archive must be rehydrated first.** You can't read directly from Archive; you must rehydrate to Hot or Cool, which can take 1–15 hours.
- **Archive rehydration is per-GB chargeable.** Standard priority ~$0.022/GB, High priority ~$0.10/GB.
- **Egress has its own price.** Reading from storage = read fee. Sending the bytes outside the region = egress fee. Cumulative.
- **Most retrieval surprises are minimum-retention fees.** "We moved a blob to Cold and deleted it 10 days later" = early-deletion fee for the remaining 80 days.
- **Always quote retrieval cost upfront.** A simple email saying "this restore will cost ~NZD $X plus ~$Y in egress, ETA Z hours" prevents bill-shock complaints later.

## ⌨️ Activity 1 — Build a small Cold-tier blob set

1. Storage account → **Containers → + Container** → name `cold-test`.
2. Upload three small files (any test files; total <100 MB).
3. Select all three blobs → **Change tier → Cold → Save**.
4. Wait ~30 seconds. Refresh — all three blobs now show tier = Cold.

## ⌨️ Activity 2 — Read a Cold-tier blob and watch the cost

1. Click any blob → **Download**. Confirm download.
2. Cloud Shell — read the blob via CLI to also exercise the read path:

```bash
SA=<your storage account>
az storage blob download \
  --account-name $SA \
  --container-name cold-test \
  --name <blob name> \
  --auth-mode login \
  --file /tmp/dl.bin
```

3. The read happened against Cold tier. The cost you just incurred:
   - Read transaction: ~$0.0000013 (one operation).
   - Read data: ~$0.045 × 0.0001 GB ≈ $0.0000045.
   
The numbers are tiny *per blob* but scale linearly with size and count.

## ⌨️ Activity 3 — Estimate a hypothetical retrieval cost

You've been asked: "Restore the last 5 TB of WOD content from Cold tier for an audit."

Walk through the cost mentally:

- Read data: 5 × 1024 GB × $0.045/GB ≈ **$230**
- Read transactions: ~5,000 list operations + 1 GET per blob; with 50 KB avg blob size, that's 100M GETs × $0.13/10K ≈ **$1,300**
- If the data leaves Azure (egress to on-prem): 5 TB × ~$0.10/GB = **$510**

Total: roughly **NZD $2,000–2,500** for the read alone. That number goes in the change request you'd send to leadership, not the surprise on the bill next month.

> [!IMPORTANT]
> Transaction fees on Cold tier are **higher** than Hot. For tiny blobs read en masse, transactions can dominate over read-bytes. Always include both in estimates.

## ⌨️ Activity 4 — Rehydrate from Archive (read-only walkthrough)

You don't have an Archive blob yet (Archive has 180-day minimum retention — costly to test directly). For this activity, walk through the process on a **non-Archive** blob to see the UI; then read the production rehydration page in DSR.

1. In your test storage account, click any blob → **Change tier → Archive → Save**.
2. Now click the same blob again → **Change tier → Hot → Save**. The portal shows: **"Rehydrate operation will be queued."** ETA: up to 15 hours.
3. Cancel out (don't actually queue it — that triggers the early-deletion fee since the blob hasn't been in Archive 180 days).

For the production cost picture, find this rehydration scenario in Cost Management:

```kql
// Storage account-level retrieval bytes by hour
StorageBlobLogs
| where AccountName has "stanlnzn"
| where OperationName in ("GetBlob", "ReadBlob")
| summarize bytes = sum(ResponseBodySize) by AccountName, bin(TimeGenerated, 1h)
| order by TimeGenerated desc
```

> [!TIP]
> "High priority" rehydrate is for ad-hoc legal discovery only. The cost is ~5× higher than Standard. Default to Standard unless leadership explicitly approves High.

## ⌨️ Activity 5 — The minimum-retention pitfall

1. In your test account, upload a tiny test blob to a fresh container `min-ret-test`.
2. Change tier → Cool → Save.
3. Immediately delete the blob.
4. Open Cost Management (search bar → **Cost Management → Cost analysis**) → filter Subscription = trial → **Group by: Meter** → look for "early deletion" line items in the next 24–48h.

The 30-day Cool minimum means you've been billed for ~29 days of storage even though the blob existed for seconds.

## ⌨️ Activity 6 — Find recent retrieval spikes in production (read-only on DSR)

```kql
StorageBlobLogs
| where TimeGenerated > ago(7d)
| where AccountName == "stanlnznblobprdrosi01"
| where OperationName in ("GetBlob","ReadFile")
| summarize bytes_read_gb = sum(ResponseBodySize) / 1024.0 / 1024.0 / 1024.0
            by bin(TimeGenerated, 1h)
| order by TimeGenerated desc
```

Use this to triage "why did the bill spike last Thursday?" tickets. The answer is usually visible in this query.

## 🦾 Now your turn!

1. Build a "retrieval cost estimator" Excel/Workbook with inputs: GB to read, source tier, blob count, egress on/off. Output: NZD estimate.
2. Find the **change feed** option on your storage account (Data protection → Blob change feed). What does it record? (Hint: this is one way to enumerate every read/write retroactively.)
3. From Cost Management, find your trial subscription's "Bandwidth — Egress (External)" meter for last month. Could be $0 if you didn't egress anything.
4. Write a one-paragraph "retrieval plan" template you'd send to leadership: scope, ETA, estimated cost, egress destination.

## ✅ Success checklist

- [ ] You can name the per-GB read cost for Hot, Cool, Cold, Archive (approximate is fine).
- [ ] You can estimate the cost of a 5 TB retrieval from Cold tier including transactions.
- [ ] You can describe Archive rehydration (priorities, ETA, cost).
- [ ] You know the three minimum-retention durations (Cool 30, Cold 90, Archive 180).
- [ ] You can find a recent retrieval spike in Storage logs.

## 📚 Self-serve refresher

- [Blob storage pricing — current](https://azure.microsoft.com/pricing/details/storage/blobs/) — for the live numbers.
- [Reduce costs by managing data lifecycle](https://learn.microsoft.com/azure/storage/blobs/storage-blob-cost-optimization) — Microsoft's authoritative cost-optimisation reference.

## 💰 Cost note

- This lab: under $0.50.
- Production retrieval: see Activity 3 — multi-thousand-NZD scale is normal for TB-class restores.

**Cleanup:** delete `cold-test` and `min-ret-test` containers — *only after* the 30-day Cool minimum on the second one has elapsed (or accept the small early-deletion fee).

---

⬅️ **Previous:** [Step 07 — Blob lifecycle management](step-07-blob-lifecycle.md)
➡️ **Next:** [Step 09 — Azure Files (NFS & SMB)](step-09-azure-files.md)
