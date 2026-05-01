# Step 06 — Storage accounts deep-dive

_The "I understand every storage account in DSR" lab._ 🗄️

> [!NOTE]
> **Duration:** 90 minutes
> **Lab cost:** < NZD $1
> **Pairs with:** Module 6 of the training plan

---

## 📖 Session overview

Storage is the heart of the Preservation Team's responsibility. DSR has four production storage accounts, each chosen carefully for redundancy, performance, and access pattern. This session walks through what makes each account different — kind, tier, redundancy, endpoints, firewall — and how to read those settings safely. By the end, you'll be able to look at any DSR storage account and immediately understand its role in the system.

## 🎯 What you'll learn

- The difference between **StorageV2**, **BlockBlobStorage**, **FileStorage** account kinds
- The four **redundancy options** — LRS, ZRS, GRS, RA-GRS — and why DSR uses ZRS
- The three **performance tiers** — Standard, Premium (FileStorage), Premium (BlockBlob)
- How **access tiers** (Hot, Cool, Cold) work at the account vs blob level
- How **storage account firewall**, **service endpoints**, and **private endpoints** combine
- How DSR's four storage accounts map to the team's responsibilities

## 📚 Before this session — MS Learn pre-work

| Module | Time | Why this matters |
|---|---|---|
| [Introduction to Azure Storage](https://learn.microsoft.com/training/modules/intro-to-azure-storage/) | 30 min | Account types, tiers, redundancy |
| [Configure Azure Storage redundancy](https://learn.microsoft.com/training/modules/configure-storage-redundancy/) | 30 min | Why ZRS for DSR |
| [Secure Azure Storage](https://learn.microsoft.com/training/modules/secure-azure-storage-account/) | 45 min | Firewall, encryption, access keys |

## 🔤 Acronyms used

- **ADLS Gen2** = Azure Data Lake Storage v2 (StorageV2 with hierarchical namespace)
- **GRS / GZRS** = Geo / Geo-Zone Redundant Storage (cross-region)
- **HNS** = Hierarchical Namespace (ADLS Gen2 feature)
- **LRS** = Locally Redundant Storage (3 copies in one datacentre)
- **RA-** = Read-Access prefix (RA-GRS = read access to secondary)
- **SAS** = Shared Access Signature (token-based access)
- **ZRS** = Zone-Redundant Storage (3 copies across 3 AZs)

## ⏱️ EDE accounting

- Trainee self-paced: 90 min
- Instructor-led delivery: 1.5h
- Prep work: 1.75h
- Q&A: 30 min
- **Total EDE per trainee: ~5h**

## 💰 Cost note

- Three test storage accounts at Standard LRS: ~NZD $0.10/day total
- Tear down at end — total < NZD $1.

---

## 🧱 Foundational primer

### Account kinds

| Kind | Holds | Use in DSR |
|---|---|---|
| **StorageV2** (general purpose v2) | Blob, File, Queue, Table | Default for everything; all 4 DSR accounts use this |
| **BlockBlobStorage** | Block blobs only, premium SSD | Not used in DSR |
| **FileStorage** | Files only, premium SSD | Could be used for Rosetta NFS — design uses StorageV2 with Premium Files instead |

### Redundancy (cost vs durability tradeoff)

| Option | Copies | Where | Cost | Use |
|---|---|---|---|---|
| **LRS** | 3 | One datacentre | $ | Lab/dev only |
| **ZRS** | 3 | 3 AZs in region | $$ | **All DSR PRD accounts** — chosen because NZ North has no paired region |
| **GRS** | 6 | Primary region + 1 secondary | $$$ | Not available in NZ North (no pair) |
| **RA-GRS** | 6 | GRS + read access to secondary | $$$$ | Not available in NZ North |

### Access tiers (block blob only)

| Tier | Storage cost | Read cost | Min retention |
|---|---|---|---|
| **Hot** | High | Low | None |
| **Cool** | Lower | Higher | 30 days |
| **Cold** | Lowest of online | Highest of online | 90 days |
| **Archive** | Lowest of all | Hours-long rehydration | 180 days |

DSR uses Hot → Cool → Cold automatically via lifecycle policies. **Archive is not used** (per design).

### DSR's four production storage accounts

| Account | Kind | Redundancy | Holds | Used by |
|---|---|---|---|---|
| `stanlnznfileprdrosi01` | StorageV2 | ZRS | Premium Files (NFS) | Rosetta delivery/repository |
| `stanlnznfileprdrosi02` | StorageV2 | ZRS | Standard Files (SMB) | Rosetta admin shares |
| `stanlnznblobprdrosi01` | StorageV2 | ZRS | Blob (Hot/Cool/Cold) | Rosetta storage groups (NLNZ_IE, NLNZ_File, Metadata, SIP, ANZ_IE) |
| `stanlnznblobprdwod01` | StorageV2 | ZRS | Blob with Legal Hold | WOD archive |

---

## ⌨️ Activity 1 — Inspect each DSR storage account (read-only)

If you have Reader on DSR DEV / UAT:

1. Portal → search the four account names listed above (DEV/UAT equivalents).
2. For each, in **Overview**, note:
   - Performance, Redundancy, Account kind
   - Primary endpoints (Blob, File, Queue, Table)
   - Status

## ⌨️ Activity 2 — Compare accounts side-by-side using Resource Graph

```kusto
Resources
| where type == "microsoft.storage/storageaccounts"
| where name startswith "stanlnzn"
| project name,
          kind,
          sku=sku.name,
          tier=sku.tier,
          accessTier=tostring(properties.accessTier),
          httpsOnly=tostring(properties.supportsHttpsTrafficOnly),
          publicNetwork=tostring(properties.publicNetworkAccess)
| order by name asc
```

✅ One screen, all DSR accounts, with the key settings.

## ⌨️ Activity 3 — Deploy your own three accounts to compare

```bash
RG=rg-training-storage-<your-initials>
az group create -n $RG -l australiaeast --tags purpose=dsr-training

# 1. Standard LRS — cheap, 1 datacentre
az storage account create -n sttrainlrs<your-initials>$RANDOM \
  -g $RG -l australiaeast --sku Standard_LRS --kind StorageV2

# 2. Standard ZRS — like DSR
az storage account create -n sttrainzrs<your-initials>$RANDOM \
  -g $RG -l australiaeast --sku Standard_ZRS --kind StorageV2

# 3. Premium FileStorage (NFS-capable) — like stanlnznfileprdrosi01
az storage account create -n sttrainpfs<your-initials>$RANDOM \
  -g $RG -l australiaeast --sku Premium_ZRS --kind FileStorage
```

Compare them in the portal. Note the differences in:
- Available services (only Premium FileStorage shows the **File shares** blade prominently)
- Configurable access tier (only StorageV2 has Hot/Cool/Cold)
- Cost preview

## ⌨️ Activity 4 — Endpoints, firewall, and Private Endpoint config

1. Pick one of your test accounts → **Networking** → **Public network access**.
2. Read (don't change yet): default = Enabled from all networks.
3. Now set **Selected networks** → see the firewall rule UI.
4. Look at **Private endpoint connections** tab — empty unless you completed Module 5.

In production, DSR storage accounts have:
- Public network access = **Disabled**
- Connectivity only via Private Endpoint from `snet-pe-prd`

## ⌨️ Activity 5 — Encryption settings

1. Storage account → **Encryption** blade.
2. Note: Microsoft-managed keys by default. DSR uses customer-managed keys (CMK) backed by Azure Key Vault — see § 9 of the design doc.
3. Don't change anything in your lab account — just observe the option.

## ⌨️ Activity 6 — Container, file share, queue, table — all in one account

In your StorageV2 account:

1. **Containers** → + Container → name `test-blob`.
2. **File shares** → + File share → name `test-share`. Skip backup. (Hot tier, small quota.)
3. **Queues** → + Queue → name `test-queue`.
4. **Tables** → + Table → name `testtable`.

✅ All four sub-services share one account, one billing line, one access policy.

## ⌨️ Activity 7 — Account-level vs blob-level access tier

1. Account → **Configuration** → **Default access tier** = Hot.
2. Upload a blob → it inherits Hot.
3. On the blob → right-click → **Change tier** → Cool.

The **default** is account-level. Individual blobs can override (and lifecycle rules typically do).

## ⌨️ Activity 8 — Tear down

```bash
az group delete -n rg-training-storage-<your-initials> --yes --no-wait
```

---

## 🦾 Now your turn!

Write a Resource Graph query that returns **all storage accounts in DSR DEV+UAT+PRD** showing:

- Name, redundancy, account kind, public network access setting, HTTPS-only setting
- Sorted by env then name

Save it as a saved query — your weekly storage health check.

---

## ✅ Success checklist

- [ ] You can list every DSR storage account and explain its role
- [ ] You understand why all PRD accounts use ZRS
- [ ] You've deployed accounts in 3 different SKUs and compared them
- [ ] You can find encryption, networking, and firewall settings
- [ ] You know that DSR PRD accounts disable public network access

---

## 💰 Cost note

< NZD $1 with prompt teardown.

---

➡️ **Next:** [Step 07 — Reading the lifecycle policy (Hot / Cool / Cold)](step-07-lifecycle.md)