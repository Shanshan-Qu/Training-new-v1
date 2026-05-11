# Step 05 — Storage accounts & tier management (Hot / Cool / Cold)

> [!NOTE]
> **Merged lab.** Combines the former Step 05 (storage account fundamentals) and Step 06 (Cool/Cold retrieval economics) into a single session. **Archive tier is excluded** — DIA does not use it. Tier coverage stops at Cold.

_The "this is the heart of your day job" lab._ 💾 Builds deep working knowledge of the storage accounts that hold every preserved digital object — Rosetta IE, deposit areas, WOD web archive content — plus the tier-and-retrieval cost intuition the team needs to quote restores to leadership.

> [!NOTE]
> **Trainee duration:** 120 minutes
> **Lab cost:** under NZD $1 — one tiny Standard_ZRS account and a few KB of blob data.
> **Prerequisites:** Steps 00–04 complete.
> **Pairs with:** Module 2 of the DIA training plan (Storage).

---

## 📖 Session overview

The Preservation Team owns the day-to-day operations of the storage accounts that hold every preserved digital object. You need to:

1. **Read** a storage account end-to-end: kind, redundancy, endpoints, firewall, data protection.
2. **Manage tier transitions** — Hot ↔ Cool ↔ Cold — and understand the cost levers behind them.
3. **Quote retrieval costs** to leadership *before* a multi-TB read so the bill doesn't surprise anyone.

**What you'll learn**
- The **storage account kinds** and which one DSR uses (and why).
- **Redundancy options** (LRS, ZRS, GRS, GZRS) — what each protects against and why DSR uses ZRS in NZ North.
- **Tier model — Hot / Cool / Cold only**. Per-GB storage cost, per-GB read cost, transactions, minimum-retention durations.
- **Endpoints & firewall** — public, private, the `privatelink.blob.core.windows.net` pattern.
- How to **estimate the cost of a retrieval** before doing it, and how to **read retrieval activity** afterwards in `StorageBlobLogs`.
- The Cool 30-day / Cold 90-day **early deletion fee** pitfall.

> [!IMPORTANT]
> **Archive tier is out of scope.** DIA does not use Archive. The rehydration mechanism, rehydration priorities, and Archive's 180-day minimum retention are not covered. If a future business case ever needs Archive, that's a separate design conversation with Cloud Architecture — not an ops decision.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **Storage account** | The container that holds blobs, files, queues, tables. The unit of billing, security, and naming. |
| **Kind** | The flavour of storage account: `StorageV2` is what DSR uses for blob accounts. |
| **LRS / ZRS / GRS / GZRS** | Redundancy SKUs. DSR production = **ZRS** (3 copies across 3 AZs in one region). GRS is unavailable in NZ North (no paired region). |
| **Hot / Cool / Cold** | Access tiers — different storage and retrieval prices. DSR uses all three. |
| **Read cost / data retrieval fee** | Per-GB charge for reading data out of Cool/Cold (Hot is free per GB). |
| **Transaction cost** | Per-operation charge — listing, GET, PUT each cost a tiny amount. Higher on Cool/Cold than Hot. |
| **Early deletion fee** | If you remove or move a blob before its tier's minimum retention (Cool 30d, Cold 90d), you're charged for the missing days. |
| **Egress** | Outbound data leaving the Azure region. Separate cost from read. |
| **Endpoint** | The URL where the storage account is reachable (`<name>.blob.core.windows.net` etc.). |
| **Firewall** | Storage-account-level network rules (public, allowlist, private only). |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for DSR |
|---|---|
| [Azure Storage fundamentals](https://learn.microsoft.com/training/modules/azure-storage-fundamentals/) | The model behind every account in DSR. |
| [Choose a redundancy option](https://learn.microsoft.com/azure/storage/common/storage-redundancy) | Why DSR uses ZRS in NZ North. |
| [Hot, Cool, and Cold blob tiers](https://learn.microsoft.com/azure/storage/blobs/storage-blob-storage-tiers) | Tier-cost intuition. **Skim the Archive section — out of scope here.** |
| [Plan and manage costs for Azure Storage](https://learn.microsoft.com/azure/storage/common/storage-plan-manage-costs) | The authoritative cost guide. |

About **2 hours** of optional pre-reading.

## 🧱 Foundational primer

### Account fundamentals
- **`StorageV2` is the standard kind.** All DSR blob accounts are `StorageV2`.
- **NZ North has no paired region** → GRS / GZRS unavailable. DSR uses **ZRS**.
- **Storage accounts have multiple endpoints** (blob, file, queue, table, dfs, web). Each can have its own private endpoint.
- **Firewall is whitelist-style.** Public access is rare in DSR — Private Endpoints are the pattern.
- **Naming is global.** Account name = lowercase, 3–24 chars, globally unique.

### Tier model (DSR uses these three only)

| Tier | Storage $/GB/mo (approx, AUE) | Read $/GB | Min retention | Use it for |
|---|---|---|---|---|
| **Hot** | ~$0.022 | $0 | none | Active deposit, current IE access |
| **Cool** | ~$0.012 | ~$0.018 | 30 days | Recently preserved, occasional access |
| **Cold** | ~$0.0036 | ~$0.045 | 90 days | "We'll likely never read this" — most of DSR's preserved corpus |
| ~~Archive~~ | _not used at DIA_ | — | — | — |

- **Hot reads are free per GB.** You only pay transactions and egress (if data leaves the region).
- **Cool/Cold per-GB reads add up.** 1 TB read from Cold ≈ $45. Multiply by frequency.
- **Transaction fees are higher** on Cool/Cold than Hot. For tiny blobs read en masse, transactions can dominate over read-bytes.
- **Egress is separate.** Read fee + (if leaving Azure region) egress fee. Cumulative.

### The minimum-retention trap
"We moved a blob to Cold and deleted it 10 days later" = early-deletion fee for the remaining 80 days. **Always quote retrieval cost upfront** — a one-line email saying "this restore will cost ~NZD $X plus ~$Y in egress" prevents bill-shock later.

---

## ⌨️ Activity 1 — Inspect a real DSR-style storage account from Resource Graph

Resource Graph Explorer:

```kql
resources
| where type == "microsoft.storage/storageaccounts"
| project name,
          kind,
          sku = sku.name,
          accessTier = properties.accessTier,
          allowBlobPublicAccess = properties.allowBlobPublicAccess,
          minTlsVersion = properties.minimumTlsVersion,
          publicNetworkAccess = properties.publicNetworkAccess,
          location,
          resourceGroup
```

In production DSR you should see: `kind = StorageV2`, `sku = Standard_ZRS`, `publicNetworkAccess = Disabled`, `minTlsVersion = TLS1_2`, `allowBlobPublicAccess = false`.

> [!IMPORTANT]
> If `allowBlobPublicAccess = true` or `publicNetworkAccess = Enabled` on a production account, that's a finding for the audit team.

## ⌨️ Activity 2 — Create a Standard_ZRS storage account

1. Portal → **Storage accounts → + Create**.
2. RG: `rg-labs-foundations-<your-initials>`. Name: `stlabszrs<initials><nn>` (lowercase, ≤24 chars).
3. Region: **Australia East**. Performance: **Standard**. Redundancy: **ZRS**.
4. **Advanced** tab: leave defaults (TLS 1.2, secure transfer required).
5. **Networking** tab: **Public endpoint (all networks)** for now — locked down in Activity 5.
6. **Data protection** tab: enable **Soft delete for blobs** (7 days), **Container soft delete** (7 days), **Versioning**.
7. Tags: `purpose=dsr-training`, `app_name=lab`.
8. Create.

## ⌨️ Activity 3 — Walk every blade

Open the new account and visit each blade — goal is recognition.

| Blade | What to look for |
|---|---|
| **Overview** | Endpoints (blob, file, queue, table, web, dfs). |
| **Configuration** | Default access tier, minimum TLS, allow shared key, public access. |
| **Networking** | Public network access, firewall, private endpoints. |
| **Data protection** | Soft delete, versioning, change feed, point-in-time restore. |
| **Encryption** | Microsoft-managed vs Customer-managed keys (DSR uses CMK in Key Vault for prod). |
| **Lifecycle management** | Rules to transition / delete blobs. |
| **Diagnostic settings** (under Monitoring) | Logs and metrics destinations — Log Analytics workspace for DSR. |

## ⌨️ Activity 4 — Upload a blob and exercise tier changes (Hot ↔ Cool ↔ Cold)

1. Account → **Containers → + Container** → Name `lab`. Public access: Private.
2. **Upload** any small file. Default tier: Hot.
3. Click the blob → **Change tier** → see **Hot / Cool / Cold**. (Archive is also offered by the portal; **do not select it** — DIA doesn't use Archive.)
4. Move the blob to **Cool** → Save. Refresh — tier = Cool.
5. Move it again to **Cold** → Save. Refresh — tier = Cold.

> [!IMPORTANT]
> The portal does not stop you selecting Archive. Treat it as off-limits unless Cloud Architecture has signed off a business case.

## ⌨️ Activity 5 — Lock down public access

1. Account → **Networking** → **Public network access** → **Enabled from selected virtual networks and IP addresses**.
2. Add your current public IP. Save. Wait ~30 seconds.
3. Refresh **Containers** — still works (your IP allowed).
4. Flip to **Disabled** — refresh → auth/network error. This is what production looks like.
5. Set back to **Enabled from selected networks** with your IP allowed for the rest of the labs.

## ⌨️ Activity 6 — Read every property via Azure CLI

```bash
RG=rg-labs-foundations-<your-initials>
SA=<your storage account>

az storage account show -g $RG -n $SA --query "kind"
az storage account show -g $RG -n $SA --query "sku.name"
az storage account show -g $RG -n $SA --query "properties.accessTier"
az storage account show -g $RG -n $SA --query "properties.networkAcls"
az storage account show -g $RG -n $SA --query "properties.encryption.services"
```

Every portal blade is a view of this same JSON.

## ⌨️ Activity 7 — Read a Cold-tier blob and watch the cost

1. Move your blob back to Cold if it isn't there.
2. Cloud Shell — read the blob via CLI:

```bash
az storage blob download \
  --account-name $SA \
  --container-name lab \
  --name <blob name> \
  --auth-mode login \
  --file /tmp/dl.bin
```

3. Cost incurred:
   - Read transaction: ~$0.0000013.
   - Read data: ~$0.045 × blob size in GB.

Numbers are tiny per blob but scale **linearly** with size and count.

## ⌨️ Activity 8 — Estimate a hypothetical retrieval cost (the leadership-email skill)

You've been asked: **"Restore the last 5 TB of WOD content from Cold tier for an audit."**

Walk through the cost mentally:

- **Read data:** 5 × 1024 GB × $0.045/GB ≈ **$230**
- **Read transactions:** ~5,000 list ops + 1 GET per blob; with 50 KB avg blob size, that's ~100M GETs × $0.13/10K ≈ **$1,300**
- **Egress** (if data leaves Azure to on-prem): 5 TB × ~$0.10/GB = **$510**

**Total: roughly NZD $2,000–2,500** for the read alone. That number goes in the change request to leadership, not the surprise on the bill next month.

## ⌨️ Activity 9 — The Cool minimum-retention pitfall

1. Upload a tiny test blob to a fresh container `min-ret-test`.
2. **Change tier → Cool** → Save.
3. **Immediately delete** the blob.
4. Cost Management → Cost analysis → **Group by Meter** → look for "early deletion" line items in the next 24–48h.

You've just paid for ~29 days of Cool storage on a blob that existed for seconds. Same principle applies to Cold with a 90-day clock.

## ⌨️ Activity 10 — Find recent retrieval spikes (read-only on DSR)

```kql
StorageBlobLogs
| where TimeGenerated > ago(7d)
| where AccountName == "stanlnznblobprdrosi01"
| where OperationName in ("GetBlob","ReadFile")
| summarize bytes_read_gb = sum(ResponseBodySize) / 1024.0 / 1024.0 / 1024.0
            by bin(TimeGenerated, 1h)
| order by TimeGenerated desc
```

Use this to triage "why did the bill spike last Thursday?" tickets.

## ⌨️ Activity 11 — Read the production storage layout (read-only on DSR)

```kql
resources
| where type == "microsoft.storage/storageaccounts"
| where name has "stanlnzn"
| project name,
          redundancy = sku.name,
          publicAccess = properties.publicNetworkAccess,
          accessTier = properties.accessTier,
          resourceGroup
| order by resourceGroup, name
```

You should see `stanlnznblobprdrosi01`, `stanlnznfileprdrosi01`, `stanlnznblobprdwod01` with `publicAccess = Disabled` and `redundancy = Standard_ZRS`.

## 🦾 Now your turn!

1. Resource Graph: list every storage account whose **default access tier ≠ Hot**. Which ones use Cool? Which use Cold? (Audit query.)
2. Build a **retrieval cost estimator** (a small Workbook or spreadsheet) with inputs: GB to read, source tier (Hot/Cool/Cold), blob count, egress on/off. Output: NZD estimate.
3. From CLI, list containers in your account using **Azure AD auth** (`--auth-mode login`), not the account key.
4. Write a one-paragraph "retrieval plan" template for leadership: scope, ETA, estimated cost, egress destination.
5. Find your trial subscription's "Bandwidth — Egress (External)" meter for last month in Cost Management.

## ✅ Success checklist

- [ ] You can name the storage account kinds and identify which DSR uses.
- [ ] You can explain why GRS isn't available in NZ North and which redundancy DSR runs.
- [ ] You can read a storage account's JSON view and identify tier, redundancy, public access, TLS, encryption.
- [ ] You can write a Resource Graph query that lists every storage account with public access enabled.
- [ ] You can move a blob between Hot / Cool / Cold and explain the cost impact.
- [ ] You can estimate the cost of a 5 TB retrieval from Cold including transactions and egress.
- [ ] You know the two minimum-retention durations DSR cares about (**Cool 30, Cold 90**).
- [ ] You can find a recent retrieval spike in `StorageBlobLogs`.
- [ ] You **do not** propose Archive for any DSR workload.

## 📚 Self-serve refresher

- [Storage account overview](https://learn.microsoft.com/azure/storage/common/storage-account-overview) — canonical reference.
- [Resource Graph storage queries](https://learn.microsoft.com/azure/governance/resource-graph/samples/starter#azure-storage) — pre-built audit queries.
- [Storage firewall and virtual networks](https://learn.microsoft.com/azure/storage/common/storage-network-security) — when "I can't connect" is a firewall problem.
- [Blob storage pricing — current](https://azure.microsoft.com/en-us/pricing/details/storage/blobs/) — for the live numbers.

## 💰 Cost note

- Empty Standard_ZRS storage account: ~NZD $0.05/month.
- Tiny blobs across Hot/Cool/Cold: negligible.
- One early-deletion fee from Activity 9: a few cents.
- Total this lab: well under $1.

**Cleanup:** keep the storage account; it's reused in Steps 06–08. Delete it when you finish Phase 2.

---

⬅️ **Previous:** [Step 04 — Networking primer](step-04-networking-primer.md)
➡️ **Next:** [Step 06 — Azure Files (DSR-specific)](step-06-azure-files.md)
