# Step 01 — Azure foundations

_The "what is this thing called Azure?" lab._ 🏗️

> [!NOTE]
> **Duration:** 90 minutes
> **Lab cost:** < NZD $0.50 (one Standard storage account, torn down at end)
> **Pairs with:** Module 1 of the training plan

---

## 📖 Session overview

This session introduces the core building blocks of Azure — the platform that holds the DSR landing zone. It covers how Microsoft organises resources in Azure (tenants, subscriptions, resource groups), how regions and redundancy zones work, and how DIA's naming and tagging conventions map onto these concepts.

By the end, you'll have created your own resource group, deployed a storage account, and torn it down — all the way through the lifecycle of an Azure resource. You'll also understand why DSR resources are named the way they are.

## 🎯 What you'll learn

- The difference between a **tenant**, **subscription**, **resource group**, and **resource**
- How **Azure regions** and **Availability Zones** provide redundancy (and why DSR uses NZ North + ZRS)
- How **Azure RBAC** works — Owner vs Contributor vs Reader
- How DIA's naming standard (`rg-dia-anl-nzn-{net,stor,app,svc,mgt}-<env>`) and tagging standard (`app_name=anl`, `org_name=dia`) work
- How to deploy your first resource into your own training subscription
- How to read the **JSON view** of any Azure resource (the source of truth)

## 📚 Before this session — MS Learn pre-work

| Module | Time | Why this matters |
|---|---|---|
| [Describe cloud computing](https://learn.microsoft.com/training/modules/describe-cloud-compute/) | 30 min | Mental model for what "the cloud" actually is |
| [Describe Azure architecture and services](https://learn.microsoft.com/training/modules/azure-architecture-fundamentals/describe-core-architectural-components-of-azure/) | 45 min | Regions, subscriptions, resource groups, ARM |
| [Describe Azure identity, access, and security](https://learn.microsoft.com/training/modules/describe-azure-identity-access-security/) | 30 min | RBAC, Entra ID basics — sets up Module 3 |

## 🔤 Acronyms used

- **ARM** = Azure Resource Manager. The API that deploys everything in Azure. Every "create resource" click eventually becomes an ARM call.
- **AZ** = Availability Zone. A physically separate datacentre within a region.
- **CLI** = Command-Line Interface
- **GRS / LRS / ZRS** = Redundancy options for storage (Geo / Locally / Zone redundant)
- **RBAC** = Role-Based Access Control
- **RG** = Resource Group

## ⏱️ EDE accounting

- Trainee self-paced: 90 min
- Instructor-led delivery: 1.5h
- Prep work (MS Learn): 1.75h
- Q&A: 30 min
- **Total EDE per trainee: ~5h**

## 💰 Cost note

- Standard LRS storage account: ~NZD $0.03/day if left running.
- This lab tears down at the end — total cost should be < $0.50.

---

## 🧱 Foundational primer

If you read nothing else, read this:

- **Tenant** = your DIA identity directory in Azure. Holds all users, groups, and apps.
- **Subscription** = a billing boundary. DSR has 3: DEV, UAT, PRD.
- **Resource Group (RG)** = a folder that holds related resources. Has a region. Resources inside can be in any region, but the RG's region is where its metadata lives.
- **Resource** = an actual thing — a VM, a storage account, a network. Lives inside an RG.
- **Region** = a geographic area (e.g. `australiaeast`, `newzealandnorth`). Resources live in a region.
- **Availability Zone (AZ)** = a physically separate datacentre within a region. ZRS storage replicates across 3 AZs.
- **ARM template / Bicep / Terraform** = three ways to describe resources as code instead of clicking. Behind the scenes, all of them call the same ARM API.

> [!IMPORTANT]
> NZ North has **no paired region** for cross-region replication, which is why the DSR design uses **ZRS** (zone-redundant) for production data and recommends an asynchronous secondary copy out-of-region.

## 🏷️ DIA naming standard (memorise this)

```
rg-dia-anl-nzn-{net|stor|app|svc|mgt}-<env>
```

| Part | Means |
|---|---|
| `rg` | resource group |
| `dia` | organisation |
| `anl` | app name (Archives & National Library) |
| `nzn` | region (NZ North) |
| `net|stor|app|svc|mgt` | category (network / storage / application / shared services / management) |
| `<env>` | `dev`, `uat`, or `prd` |

## 🏷️ DIA tagging standard

| Tag | Value | Why |
|---|---|---|
| `app_name` | `anl` | Slice cost reports by app |
| `org_name` | `dia` | Multi-tenant cost segregation |
| `env` | `dev` / `uat` / `prd` | Filter and policy by environment |

---

## ⌨️ Activity 1 — Sign in and inspect your subscription

```bash
az login
az account show --output table
```

Note the **Subscription ID** — you'll use it later.

## ⌨️ Activity 2 — Create a resource group

In Cloud Shell:

```bash
az group create \
  --name rg-training-foundations-<your-initials> \
  --location australiaeast \
  --tags purpose=dsr-training app_name=training env=lab
```

Replace `<your-initials>` (e.g. `sq` for Shanshan Qu).

✅ You'll see JSON output confirming the group was created.

## ⌨️ Activity 3 — Inspect the resource group in the portal

1. [portal.azure.com](https://portal.azure.com) → search `rg-training-foundations-<your-initials>`.
2. Note the **Location**, **Subscription**, and **Tags**.
3. Click **JSON View** (top right). This is the source of truth for any Azure resource.

> [!TIP]
> Whenever a resource is "weird" or behaving unexpectedly, check JSON View first. The portal sometimes hides fields that JSON shows.

## ⌨️ Activity 4 — Deploy a storage account

```bash
az storage account create \
  --name sttraining<your-initials>$RANDOM \
  --resource-group rg-training-foundations-<your-initials> \
  --location australiaeast \
  --sku Standard_LRS \
  --kind StorageV2 \
  --tags purpose=dsr-training app_name=training env=lab
```

> [!IMPORTANT]
> Storage account names must be **3–24 lowercase letters/digits, globally unique**. The `$RANDOM` adds a random number to avoid collisions.

## ⌨️ Activity 5 — Inspect the storage account

In the portal, find the new storage account. Note:

- **Performance:** Standard
- **Redundancy:** LRS (Locally Redundant — copies in one datacentre)
- **Account kind:** StorageV2 (the modern, recommended kind)
- **Endpoints:** Blob, File, Queue, Table — all four sub-services live in one account

Compare with what production uses (per the design doc):

| DSR account | Kind | Redundancy | Used for |
|---|---|---|---|
| `stanlnznfileprdrosi01` | StorageV2 | **ZRS** | NFS Premium for Rosetta |
| `stanlnznblobprdrosi01` | StorageV2 | **ZRS** | Blob for Rosetta storage groups |
| `stanlnznblobprdwod01` | StorageV2 | **ZRS** | Blob for WOD with legal hold |

## ⌨️ Activity 6 — Look at role assignments

1. In the storage account → **Access control (IAM)** → **Role assignments**.
2. You should see your account listed as **Owner** at the subscription level (inherited).

You'll learn more about RBAC in Module 3 — for now, just note where the menu is.

## ⌨️ Activity 7 — Tear down

```bash
az group delete --name rg-training-foundations-<your-initials> --yes --no-wait
```

✅ The whole RG and everything in it deletes asynchronously. No more cost.

---

## 🦾 Now your turn!

Without using the portal, deploy a **second** storage account into a new resource group named per DIA's naming standard, with all the right tags. Then list it using:

```bash
az storage account list --output table --query "[?tags.purpose=='dsr-training']"
```

✅ Your second account should show up with all tags applied.

Tear down both groups when done.

---

## ✅ Success checklist

- [ ] You can sign in and see your subscription
- [ ] You created a resource group following DIA's naming convention
- [ ] You deployed and inspected a Standard storage account
- [ ] You can read JSON view on any resource
- [ ] You torn down everything created

---

## 💰 Cost note

If you torn down at the end of the lab (and you should), the total cost is well under NZD $0.50.

---

➡️ **Next:** [Step 02 — Azure Portal foundations](step-02-portal-foundations.md)
