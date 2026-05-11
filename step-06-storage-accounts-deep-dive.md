# Step 06 — Storage accounts deep-dive

_The "this is the heart of your day job" lab._ 💾 Builds deep working knowledge of Azure storage accounts: kinds, redundancy, endpoints, performance tiers, firewall, and how to read every property of `stanlnznblobprdrosi01` and friends.

> [!NOTE]
> **Trainee duration:** 90 minutes
> **Lab cost:** under NZD $0.50 — two tiny storage accounts and a few KB of blob data.
> **Prerequisites:** Steps 01–04 complete.
> **Pairs with:** Module 2 of the DIA training plan (Storage) — addresses Emma's "storage focus throughout" feedback.

---

## 📖 Session overview

The Preservation Team owns the day-to-day operations of the storage accounts that hold every preserved digital object — Rosetta IE, deposit areas, WOD web archive content. Before we get into lifecycle, retrieval, immutability, and reporting, you need to be able to read a storage account end-to-end: its kind, its redundancy, its endpoints, its firewall, its performance, and its data protection. This lab covers that foundation.

**What you'll learn**
- The five **storage account kinds** and which one DSR uses (and why).
- Every **redundancy option** (LRS, ZRS, GRS, GZRS, RA-GRS, RA-GZRS) — what each protects against and what it costs.
- **Performance tiers** — Standard vs Premium, Hot vs Cool vs Cold vs Archive (we'll cover transitions in Step 07).
- **Endpoints** — public, private, and the role of `privatelink.blob.core.windows.net`.
- **Storage firewall** rules — public network access, IP allowlists, and the "selected networks" pattern DSR uses.
- How to read every property of a storage account from Portal, JSON view, and Resource Graph.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **Storage account** | The container that holds blobs, files, queues, and tables. The unit of billing, security, and naming. |
| **Kind** | The flavour of storage account: `StorageV2` (general-purpose), `BlobStorage`, `FileStorage`, `BlockBlobStorage`, etc. |
| **LRS** | Locally redundant storage — 3 copies in one datacentre. |
| **ZRS** | Zone-redundant — 3 copies across 3 availability zones in one region. |
| **GRS / GZRS** | Geo-redundant — LRS/ZRS plus async copy to a paired region. NZ North has **no paired region** so GRS is unavailable. |
| **RA-GRS / RA-GZRS** | "Read-access" variants — the secondary region is readable. |
| **Hot / Cool / Cold / Archive** | Access tiers — different storage and retrieval prices for blobs accessed at different frequencies. |
| **Endpoint** | The URL where the storage account is reachable (`<name>.blob.core.windows.net` etc.). |
| **Firewall** | Storage-account-level network rules (public, allowlist, private only). |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Azure Storage fundamentals](https://learn.microsoft.com/training/modules/azure-storage-fundamentals/) | The model behind every account in DSR. |
| [Choose a redundancy option](https://learn.microsoft.com/azure/storage/common/storage-redundancy) | Understand why DSR uses ZRS in NZ North. |
| [Configure storage account network security](https://learn.microsoft.com/training/modules/secure-azure-storage-account/) | The firewall pattern protecting Rosetta data. |
| [Performance tiers and access tiers](https://learn.microsoft.com/azure/storage/blobs/access-tiers-overview) | Foundation for Steps 06 & 07 on lifecycle and retrieval. |

About **2 hours** of optional pre-reading.

## 🧱 Foundational primer

- **`StorageV2` is the standard kind.** All DSR storage accounts are `StorageV2` — it supports blobs, files, queues, tables, and all access tiers.
- **NZ North has no paired region.** GRS / GZRS / RA-GRS are unavailable. DSR uses **ZRS** for production blob storage — protects against datacentre-level failure but not regional.
- **Tiers apply per blob**, not per account. The account has a *default* tier; individual blobs can be in different tiers.
- **Cool and Cold tiers have minimum retention durations**: Cool = 30 days, Cold = 90 days. Move a blob out before the minimum and you're billed for the full minimum at that tier's rate.
- **Storage accounts have multiple endpoints** (blob, file, queue, table, dfs, web). Each can have its own private endpoint.
- **Firewall is whitelist-style.** Default = "Disabled" or "Allow from selected networks". Public access is rare in DSR — Private Endpoints are the pattern.
- **Naming is fixed and global.** Account name = lowercase, 3–24 chars, must be globally unique across all of Azure.

## ⌨️ Activity 1 — Inspect a real DSR-style storage account from Resource Graph

This works against any subscription you have Reader on.

1. Resource Graph Explorer → run:

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

2. Note for each storage account: kind, redundancy SKU, default tier, public access, TLS version.
3. In production DSR you'd see: `kind = StorageV2`, `sku = Standard_ZRS`, `publicNetworkAccess = Disabled`, `minTlsVersion = TLS1_2`, `allowBlobPublicAccess = false`.

> [!IMPORTANT]
> If `allowBlobPublicAccess = true` or `publicNetworkAccess = Enabled` on a production account, that's a finding for the audit team. Both should be `false`/`Disabled` in DSR.

## ⌨️ Activity 2 — Create a Standard_ZRS storage account

1. Portal → **Storage accounts → + Create**.
2. Subscription: trial. RG: `rg-labs-foundations-<your-initials>`. Name: `stlabszrs<your-initials><nn>` (lowercase, ≤24 chars).
3. Region: **Australia East** (no ZRS in trial NZ North if that's even available to you).
4. Performance: **Standard**. Redundancy: **ZRS**.
5. **Advanced** tab: leave defaults (TLS 1.2, secure transfer required).
6. **Networking** tab: **Public endpoint (all networks)** for now — we'll lock this down in Activity 5.
7. **Data protection** tab: enable **Soft delete for blobs** (7 days), **Container soft delete** (7 days), **Versioning**.
8. Tags: `purpose=dsr-training`, `owner=<email>`, `app_name=lab`.
9. Review + create → Create.

## ⌨️ Activity 3 — Walk every blade

Open the new account and visit each of these blades; the goal is recognition, not configuration.

| Blade | What to look for |
|---|---|
| **Overview** | The endpoints (blob, file, queue, table, web, dfs). |
| **Configuration** | Default access tier, minimum TLS, allow shared key, public access. |
| **Networking** | Public network access, firewall, private endpoints, virtual networks. |
| **Data protection** | Soft delete, versioning, change feed, point-in-time restore. |
| **Encryption** | Microsoft-managed keys vs Customer-managed keys (DSR uses CMK in Key Vault for prod). |
| **Lifecycle management** | Rules to transition / delete blobs (we'll author one in Step 07). |
| **Storage browser** | Built-in blob/file explorer in the portal. |
| **Diagnostic settings (under Monitoring)** | Logs and metrics destinations — Log Analytics workspace for DSR. |

## ⌨️ Activity 4 — Upload a blob and inspect its properties

1. Account → **Containers → + Container** → Name: `lab`. Public access: Private.
2. Open `lab` → **Upload** → pick any small file from your laptop. Default tier: Hot.
3. Click the uploaded blob → **Properties / Edit metadata**. Note: blob URL, content type, content MD5, last modified, ETag.
4. Click **Change tier** → see Hot / Cool / Cold / Archive options. Don't change yet.
5. **Generate SAS** at the blob level → see options for permissions, expiry, allowed IPs, signing key. SAS URLs expire. Useful for time-limited shares.

> [!TIP]
> In DSR, SAS tokens are rare — most access is via Managed Identity. SAS exists for occasional egress to other government agencies. Always use the shortest expiry that works.

## ⌨️ Activity 5 — Lock down public access

1. Account → **Networking** → **Public network access** → **Enabled from selected virtual networks and IP addresses**.
2. Add your current public IP to the firewall allowlist (the portal will offer this).
3. Save. Wait ~30 seconds.
4. Test: refresh the **Containers** blade — still works because your IP is allowed.
5. Now flip to **Disabled**. Refresh — you'll get an authentication / network error. This is what production looks like.
6. Set back to **Enabled from selected networks** and keep your IP allowed for the rest of the labs.

## ⌨️ Activity 6 — Read every property via Azure CLI

Cloud Shell:

```bash
RG=rg-labs-foundations-<your-initials>
SA=<your storage account>

# The full property set
az storage account show -g $RG -n $SA -o json | head -120

# A few specific values
az storage account show -g $RG -n $SA --query "kind"
az storage account show -g $RG -n $SA --query "sku.name"
az storage account show -g $RG -n $SA --query "properties.accessTier"
az storage account show -g $RG -n $SA --query "properties.networkAcls"
az storage account show -g $RG -n $SA --query "properties.encryption.services"
```

Compare the JSON output to what the portal shows on each blade — every blade is a view of this same JSON.

## ⌨️ Activity 7 — Read the production storage layout (read-only on DSR)

Use this Resource Graph query against the DSR subscriptions when you have Reader access:

```kql
resources
| where type == "microsoft.storage/storageaccounts"
| where tags.app_name == "anl"
| extend tier = tostring(properties.accessTier),
         publicAccess = tostring(properties.publicNetworkAccess),
         redundancy = tostring(sku.name)
| project name, tier, redundancy, publicAccess, location, resourceGroup, subscriptionId
| order by resourceGroup, name
```

You should see the production storage accounts (e.g. `stanlnznblobprdrosi01`, `stanlnznfileprdrosi01`, `stanlnznblobprdwod01`) with `publicAccess = Disabled` and `redundancy = Standard_ZRS`.

## 🦾 Now your turn!

1. From your test storage account, write a Resource Graph query that returns the account *and* every blob container in the account, joined.
   - Hint: container type is `microsoft.storage/storageaccounts/blobservices/containers`.
2. From CLI, list the containers in your account (`az storage container list ...`) using **Azure AD authentication**, not the account key.
3. From the JSON view of your storage account, find the `properties.encryption.keySource`. What is it set to? (Should be `Microsoft.Storage`.)
4. Try to enable a redundancy *upgrade* (Standard_ZRS → Standard_GZRS) — in Australia East this is allowed, and you'll see the cost-impact warning. Don't actually save it; just observe.

## ✅ Success checklist

- [ ] You can name the five storage account kinds and identify which DSR uses.
- [ ] You can name the redundancy options and explain why GRS isn't available in NZ North.
- [ ] You can read a storage account's JSON view and identify tier, redundancy, public access, TLS, encryption.
- [ ] You can write a Resource Graph query that lists every storage account with public access enabled (compliance check).
- [ ] You've toggled public network access off and back on.
- [ ] You've uploaded and inspected a blob.

## 📚 Self-serve refresher

- [Storage account overview](https://learn.microsoft.com/azure/storage/common/storage-account-overview) — the canonical reference.
- [Resource Graph storage queries](https://learn.microsoft.com/azure/governance/resource-graph/samples/starter#azure-storage) — pre-built audit queries.
- [Storage firewall and virtual networks](https://learn.microsoft.com/azure/storage/common/storage-network-security) — when "I can't connect" is a firewall problem.

## 💰 Cost note

- Empty Standard_ZRS storage account: ~NZD $0.05/month.
- Tiny test blob (a few KB): negligible.
- Total this lab: well under $0.50.

**Cleanup:** keep the storage account; it's reused in Steps 06–11. Delete it when you finish Phase 2.

---

⬅️ **Previous:** [Step 05 — Networking primer](step-05-networking-primer.md)
➡️ **Next:** [Step 08 — Cold tier retrieval & rehydration cost](step-08-cold-retrieval.md) _(Step 07 lifecycle content merged into this module)_
