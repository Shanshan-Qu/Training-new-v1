Shanshan — the file system is still failing, so I can't save files for you to download. I'll deliver the markdown inline in batches you can copy directly into your repo. **Each fenced ` ```markdown ` block below = one `.md` file**, with the filename in the heading right above it.

This first batch covers the **repo README + Module 0 + Foundations stream (Modules 1–5)**. After this batch, just say "**continue**" and I'll send the next batch (Storage stream Modules 6–11). I'll keep going until all 27 are delivered.

---

## 📁 File 1 of N — `README.md`

````markdown
# DSR Training — Digital Preservation Team

Self-paced and instructor-led Azure training for the **Archives Library Digital Preservation Team** at the Department of Internal Affairs (DIA), New Zealand.

The training is grounded in the **DIA DSR DPS Azure Application Landing Zone Design v1.4** and follows the team's BAU responsibilities: storage operations and the preservation applications (Rosetta + Whole-of-Domain archive).

---

## 👥 Audience

- Technically capable; deep digital-preservation domain knowledge.
- Comfortable at the Linux/RHEL OS level.
- **No prior Azure experience** for most of the team.

This curriculum starts at zero Azure knowledge and finishes at "I can run our preservation system in production."

---

## 🗺️ Curriculum at a glance

| # | Module | Stream | Duration | Cost (NZD) |
|---|---|---|---|---|
| 00 | [Environment readiness](step-00-prereqs.md) | Pre-flight | 45 min | $0 |
| 01 | [Azure foundations](step-01-azure-foundations.md) | Foundations | 1.5h | < $0.50 |
| 02 | [Azure Portal foundations](step-02-portal-foundations.md) | Foundations | 1h | $0 |
| 03 | [Identity & access for the operator](step-03-identity-access.md) | Foundations | 1.25h | < $0.50 |
| 04 | [Guardrails, governance & audit](step-04-governance.md) | Foundations | 1.5h | < $0.50 |
| 05 | [Networking primer (read-only view)](step-05-networking.md) | Foundations | 1.5h | < $1 |
| 06 | [Storage accounts deep-dive](step-06-storage-accounts.md) | Storage | 1.5h | < $1 |
| 07 | [Reading the lifecycle policy](step-07-lifecycle.md) | Storage | 1h | < $1 |
| 08 | [Azure Files: NFS vs SMB](step-08-azure-files.md) | Storage | 2h | < $3 |
| 09 | [Data protection posture](step-09-data-protection.md) | Storage | 1.5h | < $1 |
| 10 | [Immutability & legal hold](step-10-immutability.md) | Storage | 1.25h | < $1 |
| 11 | [Blob inventory](step-11-blob-inventory.md) | Storage | 1.5h | < $1 |
| 12 | [Rosetta architecture walkthrough](step-12-rosetta-architecture.md) | Apps | 1.5h | $0 |
| 13 | [Application Gateway + WAF for operators](step-13-app-gateway.md) | Apps | 2h | < $4 |
| 14 | [Oracle on Azure (read-only ops view)](step-14-oracle.md) | Apps | 1.25h | $0 |
| 15 | [WOD container operations](step-15-wod-containers.md) | Apps | 1.5h | < $2 |
| 16 | [Azure Monitor — foundations (storage focus)](step-16-monitor-foundations.md) | Observability | 2h | $0 |
| 17 | [Azure Monitor — hands-on labs (storage focus)](step-17-monitor-hands-on.md) | Observability | 3h | < $1 |
| 18 | [Workbooks: built-in first, then custom](step-18-workbooks.md) | Observability | 2h | $0 |
| 19 | [Reading the lifecycle cost (light)](step-19-lifecycle-cost.md) | Cost | 1h | < $1 |
| 20 | [Cost Management with forecasting](step-20-cost-management.md) | Cost | 2h | $0 |
| 21 | [Backup Center — read-only operations](step-21-backup-center.md) | Read-only ops | 1h | $0 |
| 22 | [Defender for Storage — awareness only](step-22-defender.md) | Read-only ops | 45 min | $0 |
| 23 | [Reporting — scheduled & on-demand](step-23-reporting.md) | Capstone | 1.5h | $0 |
| 24 | [Healthy Report — assembled](step-24-healthy-report.md) | Capstone | 1.5h | $0 |
| 25 | [Cost Report — assembled (with forecast)](step-25-cost-report.md) | Capstone | 1.5h | $0 |
| 26 | [Incident triage tabletop](step-26-incident-triage.md) | Capstone | 2h | $0 |

**Optional stream:** Terraform on Azure — deferred. See [Microsoft Learn — Terraform on Azure](https://learn.microsoft.com/training/paths/terraform-fundamentals/).

---

## ⏱️ Time investment per trainee

| Activity | Hours |
|---|---|
| Self-paced lab time | ~38h |
| Instructor-led delivery | ~24h |
| MS Learn pre-work | ~6h |
| Q&A / review | ~3h |
| **Total EDE per trainee** | **~71h** |

Recommended pacing: **2 modules per week over ~10–12 weeks**.

---

## 💰 Total Azure cost per trainee

- **Pay-as-you-go:** ~NZD $20–30 end-to-end if you tear down between labs.
- **Free Azure trial:** $0 (NZD ~$300 credit, 30 days).
- **Visual Studio / Microsoft Partner benefit credits:** $0.

We recommend a **dedicated training subscription** so cleanup is easy and there's no risk to production.

---

## 🚦 How to start

1. Begin with **[Module 0 — Environment readiness](step-00-prereqs.md)**. Don't skip — every later lab assumes you've completed it.
2. Work through the foundations stream (Modules 1–5) before any storage or app module.
3. Each module is self-contained markdown — usable for live sessions, self-paced study, or as reference for new starters.

---

## 🛡️ Scope boundary

This curriculum covers **only** what the Digital Preservation Team owns. Topics owned by other DIA teams (Platform / Core Support / Cloud Networking / Cloud Security / Database / FinOps) are covered as **read-only / awareness only**, with clear escalation paths.

---

## 🤝 Contributing

Found a mistake? Have a better example? Open a pull request — this curriculum is meant to grow with the team.

````

---

## 📁 File 2 of N — `step-00-prereqs.md`

````markdown
# Step 00 — Environment readiness (pre-flight)

_Get your training environment set up before any hands-on session._ ✈️

> [!NOTE]
> **Duration:** 45 minutes
> **Lab cost:** $0 (only sign-ups and account verification)
> **Pairs with:** Module 0 of the training plan

---

## 📖 Session overview

This pre-flight session gets your personal Azure training environment ready so you can start the hands-on labs without delay. It walks you through getting an Azure subscription you can experiment in safely, installing the tools you'll use across the series, and confirming you have the right access to read the production DSR landing zone.

You don't need any Azure experience to complete this session — every step has a screenshot or a copy-paste command.

## 🎯 What you'll learn

- How to obtain a personal training Azure subscription (free trial or DIA-issued sandbox)
- How to sign in to the Azure portal and Cloud Shell
- How to install Azure CLI on your laptop (optional — Cloud Shell works in the browser)
- How to sign up for a Microsoft Learn account so pre-work links work
- How to confirm Reader access on the DSR DEV / UAT landing zone for read-only walkthroughs
- How to verify everything works using a one-line CLI command

## 📚 Before this session — MS Learn pre-work

None — this **is** the pre-work for everything else.

## 🔤 Acronyms used

- **CLI** = Command-Line Interface (you type commands instead of clicking)
- **CSP** = Cloud Solution Provider (the licensing model DIA uses for Azure)
- **NZD** = New Zealand Dollar
- **RBAC** = Role-Based Access Control (who can do what in Azure)

## ⏱️ EDE accounting

- Trainee self-paced: 45 min
- Instructor-led delivery: 30 min (group walk-through)
- Prep work: 0
- Q&A: 15 min
- **Total EDE per trainee: ~1.5h**

## 💰 Cost note

- $0 for setup itself.
- A free trial Azure subscription includes ~NZD $300 credit valid for 30 days — more than enough for this entire training series if you tear down between labs.

---

## ⌨️ Activity 1 — Get a training Azure subscription

You have three options. Pick one:

| Option | Best for | How |
|---|---|---|
| **A. Azure free trial** | Most trainees | Sign up at [azure.microsoft.com/free](https://azure.microsoft.com/free). Requires a phone number and credit card for verification (no charge unless you exceed the credit). Includes NZD ~$300 free credit for 30 days. |
| **B. DIA-issued sandbox** | If DIA has provisioned one for you | Ask your team lead. Use the credentials they provide. |
| **C. Visual Studio / MPN credits** | If you have a Visual Studio subscription | Already includes monthly Azure credit — sign in at [portal.azure.com](https://portal.azure.com) with your VS account. |

> [!IMPORTANT]
> Do **not** use the production DSR subscription for hands-on labs. Use a dedicated training subscription so cleanup is clean and there's no risk to production.

## ⌨️ Activity 2 — Sign in to the Azure portal

1. Go to [portal.azure.com](https://portal.azure.com).
2. Sign in with the account from Activity 1.
3. In the top right, click your profile picture and confirm:
   - The directory name (top of the menu).
   - The subscription name (Switch Directory if needed).

✅ You should see "Welcome to Microsoft Azure" and your subscription listed.

## ⌨️ Activity 3 — Open Cloud Shell

Cloud Shell is a free Linux shell in your browser — no install needed.

1. In the top bar, click the `>_` icon.
2. Choose **Bash** when prompted.
3. If asked to create storage for Cloud Shell, accept the defaults (it's free for the storage required).
4. Run:

   ```bash
   az account show --output table
   ```

✅ You should see your subscription name and ID.

## ⌨️ Activity 4 — (Optional) Install Azure CLI on your laptop

Skip this if you're happy using Cloud Shell.

- **macOS:** `brew install azure-cli`
- **Windows:** download from [aka.ms/installazurecliwindows](https://aka.ms/installazurecliwindows)
- **Linux (RHEL):** `sudo dnf install azure-cli`

After install, sign in:

```bash
az login
```

✅ A browser opens; sign in with your training account; the terminal shows your subscription.

## ⌨️ Activity 5 — Create a Microsoft Learn account

1. Go to [learn.microsoft.com](https://learn.microsoft.com).
2. Click **Sign in** in the top right.
3. Use your **work email** (not the training Azure account — these are separate).
4. Pick a display name and confirm.

✅ Pre-work links from later modules will now track your progress.

## ⌨️ Activity 6 — Confirm Reader access on DSR DEV / UAT

For modules 12–15 you'll need to *look at* the production-like Rosetta and WOD resources. You don't need to change anything — just read.

1. Email your team lead or DIA Core Support requesting:
   > **Reader** role on the `dsr-anl-dev` and `dsr-anl-uat` subscriptions for the duration of training.
2. Once granted, in [portal.azure.com](https://portal.azure.com) → top bar → switch to the DIA tenant → confirm you can see the DSR resource groups.

> [!TIP]
> If you don't have access yet by Module 12, you can still do those modules — you'll just use screenshots from training material instead of the live environment.

## ⌨️ Activity 7 — Final verification

In Cloud Shell, run:

```bash
az account list --output table
az group list --output table
```

✅ The first command lists your training subscription. The second shows resource groups (probably empty in your training sub — that's fine).

---

## 🦾 Now your turn!

Tag your training subscription so it's easy to identify later:

```bash
az tag create --resource-id /subscriptions/<your-sub-id> \
  --tags purpose=dsr-training owner=<your-email>
```

Replace `<your-sub-id>` with your training subscription ID (from `az account show`).

---

## ✅ Success checklist

- [ ] You have a training Azure subscription (not production)
- [ ] You can sign in to [portal.azure.com](https://portal.azure.com)
- [ ] You can open Cloud Shell and `az account show` works
- [ ] You have a Microsoft Learn account
- [ ] You've requested Reader on DSR DEV / UAT (acknowledgement is enough — actual grant can come later)
- [ ] Your subscription is tagged `purpose=dsr-training`

---

## 💰 Cost note

Setup itself is free. Your free trial credit is untouched — you'll start using it from Module 1 onwards.

---

➡️ **Next:** [Step 01 — Azure foundations](step-01-azure-foundations.md)
````

---

## 📁 File 3 of N — `step-01-azure-foundations.md`

````markdown
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
````

---

## 📁 File 4 of N — `step-02-portal-foundations.md`

````markdown
# Step 02 — Azure Portal foundations

_The "I can find anything in the portal" lab._ 🧭

> [!NOTE]
> **Duration:** 60 minutes
> **Lab cost:** $0 (read-only navigation)
> **Pairs with:** Module 2 of the training plan

---

## 📖 Session overview

This session is dedicated to **navigating the Azure portal fluently**. The portal is your daily window into the DSR environment — being slow in the portal slows down every other operation. This lab teaches search, filtering, JSON view, Resource Graph queries, Cloud Shell, and the keyboard shortcuts that make portal work feel less like wading through a swamp.

It's a light hands-on session — no resources are deployed, so it's free to repeat as many times as you like.

## 🎯 What you'll learn

- How to navigate the portal efficiently — search, pinning, dashboards
- How to read JSON view on any resource (every resource has one)
- How to write **Resource Graph** queries to find resources across all subscriptions
- How to use **Cloud Shell** for quick CLI without leaving the browser
- How to set up portal favourites and keyboard shortcuts

## 📚 Before this session — MS Learn pre-work

| Module | Time | Why this matters |
|---|---|---|
| [Configure Azure resources with the Azure portal](https://learn.microsoft.com/training/modules/configure-azure-resources-portal/) | 30 min | The standard portal-foundations module |
| [Query the Azure Resource Graph](https://learn.microsoft.com/training/modules/intro-to-governance/) (Resource Graph section) | 20 min | The query tool that scales across subscriptions |

## 🔤 Acronyms used

- **KQL** = Kusto Query Language — used in Resource Graph and Log Analytics
- **MFA** = Multi-Factor Authentication

## ⏱️ EDE accounting

- Trainee self-paced: 60 min
- Instructor-led delivery: 1h
- Prep work: 50 min
- Q&A: 20 min
- **Total EDE per trainee: ~3h**

## 💰 Cost note

$0. No resources deployed.

---

## 🧱 Foundational primer

The Azure portal has 5 areas you'll use daily:

1. **Top search bar** — type any resource name, RG, service, or keyword. Use this 80% of the time.
2. **Left navigation** — pinned services. Customise it.
3. **Resource blade** — the right-hand panel that opens when you click a resource. Has tabs (Overview, Activity log, Tags, IAM, etc.) along its left edge.
4. **Cloud Shell** — `>_` icon top-right. Browser-based Bash/PowerShell.
5. **Dashboards** — pinned tiles on a custom home page. Useful for ops views.

---

## ⌨️ Activity 1 — Master the search bar

1. Sign in to [portal.azure.com](https://portal.azure.com).
2. Click the search bar (or press `/`).
3. Type the name of any resource, e.g. `rg-training` — see resources, services, marketplace items, and documentation matches.
4. Use **filter chips** at the top of the results (Resources / Services / Marketplace / Documentation).

> [!TIP]
> Type `_` followed by anything to search Activity Log entries. Type a UUID to jump to that exact resource.

## ⌨️ Activity 2 — Pin and customise the left nav

1. Click the **hamburger menu** (top left).
2. Click any service to add to the left nav, e.g. **Storage accounts**, **Resource groups**, **Subscriptions**, **All resources**, **Cost Management**.
3. Drag to reorder.

✅ Your most-used services are now one click away.

## ⌨️ Activity 3 — Resource blade tour

1. Search for any storage account (or use the one you created in Module 1, then re-create if you torn it down).
2. Click into it.
3. Walk through these tabs in the left edge of the blade:
   - **Overview** — at-a-glance state
   - **Activity log** — who did what and when (audit trail)
   - **Access control (IAM)** — role assignments
   - **Tags** — DIA's tagging
   - **Diagnose and solve problems** — guided troubleshooting
   - **Properties** / **Locks** — protection settings
   - **Export template** — get the ARM/Bicep code for this resource
4. Click **JSON View** (top right of Overview). This is the source of truth.

## ⌨️ Activity 4 — Resource Graph: find resources across subscriptions

Resource Graph is a free service that lets you query every resource in every subscription you have access to, in seconds.

1. Search **Resource Graph Explorer** in the top bar.
2. Run this query:

   ```kusto
   Resources
   | where type == "microsoft.storage/storageaccounts"
   | project name, resourceGroup, location, sku.name, tags
   | order by name asc
   ```

3. Click **Run query**.

✅ You see every storage account you have access to. In production, this is how you'd inventory the 4 DSR storage accounts in one query.

Try this one — find resources missing the `app_name` tag:

```kusto
Resources
| where tags['app_name'] == ""
| project name, type, resourceGroup
```

## ⌨️ Activity 5 — Cloud Shell

1. Click the `>_` icon (top right).
2. Choose **Bash**.
3. Run:

   ```bash
   az resource list \
     --query "[?type=='Microsoft.Storage/storageAccounts']" \
     --output table
   ```

4. Try the same with PowerShell — top dropdown → switch to PowerShell.

> [!TIP]
> Cloud Shell has 5GB of free home directory storage. Drop scripts in `~/clouddrive/` and they persist between sessions.

## ⌨️ Activity 6 — Pin a tile to a dashboard

1. Search **All resources** in the top bar.
2. Right-click any tile or chart → **Pin to dashboard** → choose **My dashboard**.
3. Top bar → **Dashboard** → see your pinned items.

You'll use this in Module 24 (Healthy Report) to build the Rosetta Operations dashboard.

## ⌨️ Activity 7 — Keyboard shortcuts

| Shortcut | What it does |
|---|---|
| `G+/` | Open search |
| `G+N` | Notifications |
| `G+H` | Home |
| `G+D` | Dashboard |
| `G+B` | Cloud Shell |
| `Esc` | Close current blade |

Practice opening a resource and pressing `Esc` to close.

---

## 🦾 Now your turn!

Write a Resource Graph query that returns:

- All storage accounts in any subscription you have access to
- Showing name, region, redundancy SKU, and the `env` tag
- Sorted by environment, then name

Save it as a **Saved query** in Resource Graph Explorer so you can re-run it weekly.

Hint:

```kusto
Resources
| where type == "microsoft.storage/storageaccounts"
| project name, location, sku=sku.name, env=tags['env']
| order by env asc, name asc
```

---

## ✅ Success checklist

- [ ] You can find any resource within 5 seconds via search
- [ ] You've customised your left nav with your most-used services
- [ ] You've used JSON View on at least one resource
- [ ] You've run at least 2 Resource Graph queries
- [ ] You've used Cloud Shell to list resources
- [ ] You've pinned at least one tile to a dashboard

---

## 💰 Cost note

$0. Browsing the portal is free.

---

➡️ **Next:** [Step 03 — Identity & access for the operator](step-03-identity-access.md)
````

---

## 📁 File 5 of N — `step-03-identity-access.md`

````markdown
# Step 03 — Identity & access for the operator

_The "who can do what" lab._ 🔐

> [!NOTE]
> **Duration:** 75 minutes
> **Lab cost:** < NZD $0.50
> **Pairs with:** Module 3 of the training plan

---

## 📖 Session overview

This session covers Azure identity and access from an operator's perspective — not a designer's. You'll learn the difference between sign-in identities (users, groups, service principals, managed identities), the difference between built-in and custom roles, and the principle of **least privilege** that governs how the Preservation Team should be granted access.

The most important takeaway: **you should be Reader on production, not Contributor or Owner**. We'll cover what that means and why.

## 🎯 What you'll learn

- The four kinds of identities in Azure: **users**, **groups**, **service principals**, **managed identities**
- The four most-used roles: **Owner**, **Contributor**, **Reader**, and **Storage Blob Data Reader**
- How **scope** works (sub → RG → resource) and why a Reader at sub level is a Reader at every resource
- How to view your own role assignments
- How to read someone else's role assignments (for audit)
- The Preservation Team's recommended access pattern in DSR

## 📚 Before this session — MS Learn pre-work

| Module | Time | Why this matters |
|---|---|---|
| [Describe Azure identity, access, and security](https://learn.microsoft.com/training/modules/describe-azure-identity-access-security/) | 30 min | RBAC fundamentals |
| [Configure Azure role-based access control](https://learn.microsoft.com/training/modules/configure-role-based-access-control/) | 35 min | Hands-on with role assignments |

## 🔤 Acronyms used

- **MI** = Managed Identity. An identity that Azure manages for you — no passwords to rotate.
- **PIM** = Privileged Identity Management. Just-in-time elevation for privileged roles.
- **RBAC** = Role-Based Access Control
- **SP** = Service Principal. An app or automation identity (not a human).

## ⏱️ EDE accounting

- Trainee self-paced: 75 min
- Instructor-led delivery: 1.25h
- Prep work: 1h
- Q&A: 30 min
- **Total EDE per trainee: ~4h**

## 💰 Cost note

< NZD $0.50 (one storage account for hands-on, torn down at end).

---

## 🧱 Foundational primer

### The four identity kinds

| Kind | What it is | Use in DSR |
|---|---|---|
| **User** | A human with an Entra ID sign-in | Every Preservation Team member |
| **Group** | A collection of users | Use groups for role assignments — never assign roles to individual users in production |
| **Service Principal (SP)** | An app/automation identity with a secret or certificate | CI/CD pipelines, Terraform |
| **Managed Identity (MI)** | An identity Azure manages — no secrets | The Rosetta delivery VMs use a system-assigned MI to authenticate to blob storage |

### The four most-used roles

| Role | Can do |
|---|---|
| **Owner** | Everything, including managing access (giving roles to others) |
| **Contributor** | Everything *except* managing access |
| **Reader** | View only — cannot change anything |
| **Storage Blob Data Reader** | Read blob *data* (the actual files) — Reader does not include this |

> [!IMPORTANT]
> **Reader** lets you see a storage account's configuration. It does NOT let you read the blobs inside. To read blobs, you need **Storage Blob Data Reader** (a separate role). This catches everyone the first time.

### Scope

Roles are assigned at a scope:

```
Subscription (broadest)
  └── Resource Group
        └── Resource (narrowest)
```

A Reader at subscription level is Reader on every RG and every resource inside. Always assign at the **narrowest scope that works**.

### The Preservation Team's recommended access pattern

| Resource | DEV | UAT | PRD |
|---|---|---|---|
| Storage accounts | Reader + Storage Blob Data Reader | Reader + Storage Blob Data Reader | **Reader** + **Storage Blob Data Reader** (read-only) |
| VMs | Reader | Reader | Reader |
| Recovery Services Vault | Reader | Reader | **Reader** (for Backup Center visibility) |
| Log Analytics Workspaces | Log Analytics Reader | Log Analytics Reader | Log Analytics Reader |

The team **does not** get Contributor on production. Changes go via DIA Platform Team.

---

## ⌨️ Activity 1 — Find your own role assignments

In Cloud Shell:

```bash
az role assignment list --assignee $(az account show --query user.name -o tsv) --output table
```

You should see at least one role on your training subscription.

## ⌨️ Activity 2 — Create test resources to play with roles

```bash
az group create -n rg-training-iam-<your-initials> -l australiaeast \
  --tags purpose=dsr-training

az storage account create \
  -n sttraining<your-initials>$RANDOM \
  -g rg-training-iam-<your-initials> \
  -l australiaeast --sku Standard_LRS --kind StorageV2 \
  --tags purpose=dsr-training
```

Capture the storage account name (e.g. `sttrainingsq1234`) — you'll use it next.

## ⌨️ Activity 3 — Try to read a blob without the data role

1. In the portal → your storage account → **Containers** → **+ Container** → name `test`.
2. Upload any small file.
3. Click the file → **View/edit**. You should see "AuthorizationPermissionMismatch" or similar — even though you're Owner.

This is the catch: **Owner does not include data-plane access by default**.

## ⌨️ Activity 4 — Grant yourself Storage Blob Data Reader

```bash
az role assignment create \
  --assignee $(az account show --query user.name -o tsv) \
  --role "Storage Blob Data Reader" \
  --scope $(az storage account show -n sttraining<your-initials><nnnn> -g rg-training-iam-<your-initials> --query id -o tsv)
```

Wait ~60 seconds for the role to propagate, then refresh the portal and try viewing the blob again. ✅ It should work.

## ⌨️ Activity 5 — View role assignments at scope

In the portal:

1. Go to your storage account → **Access control (IAM)** → **Role assignments**.
2. See your account listed with **Storage Blob Data Reader** (scope: this resource) and **Owner** (scope: subscription, inherited).

This is exactly the pattern you'd use to audit "who has access to production storage" — except at sub level instead of resource level.

## ⌨️ Activity 6 — Read role assignments using CLI (audit pattern)

```bash
# At subscription scope
az role assignment list --output table

# Just one resource group
az role assignment list --resource-group rg-training-iam-<your-initials> --output table

# Just one resource
az role assignment list \
  --scope $(az storage account show -n sttraining<your-initials><nnnn> -g rg-training-iam-<your-initials> --query id -o tsv) \
  --output table
```

In production, this is how DIA Cloud Governance answers "who has access to the WOD storage account?"

## ⌨️ Activity 7 — Inspect a managed identity (read-only)

If you have Reader on DSR DEV:

1. Navigate to a Rosetta delivery VM (e.g. starting with `rosidel`).
2. Left blade → **Identity** → see **System assigned** = On.
3. Click **Azure role assignments** → see what roles this MI has on what storage accounts.

This is how `rosidel*` VMs authenticate to blob storage without storing keys.

## ⌨️ Activity 8 — Tear down

```bash
az group delete -n rg-training-iam-<your-initials> --yes --no-wait
```

---

## 🦾 Now your turn!

Write a CLI command that:

- Lists all role assignments in your training subscription
- Filters to **only** custom roles (not built-in)
- Outputs name, role, and scope

Hint:

```bash
az role assignment list --query "[?roleDefinitionName!='Reader' && roleDefinitionName!='Contributor' && roleDefinitionName!='Owner']" --output table
```

In production, this is exactly the audit query — "show me anyone with non-standard access."

---

## ✅ Success checklist

- [ ] You can list your own role assignments
- [ ] You understand why Owner doesn't grant blob data access
- [ ] You've granted yourself Storage Blob Data Reader at resource scope
- [ ] You've audited role assignments at sub, RG, and resource scope
- [ ] You've inspected a managed identity (read-only)
- [ ] You torn down test resources

---

## 💰 Cost note

< NZD $0.50 if torn down promptly.

---

➡️ **Next:** [Step 04 — Guardrails, governance & audit](step-04-governance.md)
````

---

## 📁 File 6 of N — `step-04-governance.md`

````markdown
# Step 04 — Guardrails, governance & audit

_The "what stops me breaking production?" lab._ 🛡️

> [!NOTE]
> **Duration:** 90 minutes
> **Lab cost:** < NZD $0.50
> **Pairs with:** Module 4 of the training plan (a NEW module added to address Emma's feedback)

---

## 📖 Session overview

This session covers the guardrails Azure and DIA put in place to make sure mistakes don't escalate into outages. You'll learn what **Azure Policy** is and how it enforces standards (without you needing to write any policies — DIA's Cloud Governance team owns those). You'll learn how to view subscription, resource, and access inventories for audit. You'll learn how to use the **Activity Log** to answer "who did what when".

This module exists because Emma asked for it: visibility, governance, audit — three things the Preservation Team needs to be comfortable operating production safely.

## 🎯 What you'll learn

- What **Azure Policy** is and how to read existing policies (without writing them)
- How to use **Resource Graph** to inventory resources, subscriptions, and access for audit
- How to read the **Activity Log** to answer "who did what when"
- How to read the **Microsoft Defender for Cloud** compliance dashboard (read-only)
- How to apply **resource locks** to protect resources from accidental delete
- The escalation path when a guardrail blocks something you legitimately need to do

## 📚 Before this session — MS Learn pre-work

| Module | Time | Why this matters |
|---|---|---|
| [Describe Azure governance and compliance](https://learn.microsoft.com/training/modules/azure-management-fundamentals/describe-features-tools-azure-for-governance-compliance/) | 30 min | The big picture — Policy, Locks, Cost Mgmt, Defender |
| [Build a cloud governance strategy on Azure](https://learn.microsoft.com/training/modules/build-cloud-governance-strategy-azure/) | 45 min | Why governance exists, how it fits real-world ops |

## 🔤 Acronyms used

- **MDC** = Microsoft Defender for Cloud
- **PIM** = Privileged Identity Management

## ⏱️ EDE accounting

- Trainee self-paced: 90 min
- Instructor-led delivery: 1.5h
- Prep work: 1.25h
- Q&A: 30 min
- **Total EDE per trainee: ~5h**

## 💰 Cost note

< NZD $0.50 (one resource for lock testing, torn down at end).

---

## 🧱 Foundational primer

### The four governance pillars in Azure

| Pillar | What it does | Owner in DIA |
|---|---|---|
| **Azure Policy** | Auto-enforce rules: "all storage accounts must use ZRS", "no resources without `app_name` tag", etc. | DIA Cloud Governance |
| **RBAC** (covered in Module 3) | Who can do what | DIA Cloud Governance + Platform |
| **Resource Locks** | Prevent accidental delete or change | Resource owner — you can apply these |
| **Microsoft Defender for Cloud** | Posture & threat detection | DIA Cloud Security |

### What policies probably exist on DSR (read-only)

You don't write these. But you'll see their effects when, say, a Contributor request is denied:

- "All storage accounts must have HTTPS-only enabled"
- "All resources must have `app_name` and `env` tags"
- "No public network access on storage in PRD"
- "All Recovery Services Vaults must use ZRS"

If a deployment fails with "RequestDisallowedByPolicy", check the Policy assignment named in the error.

### Activity Log: who did what when

Every change in Azure writes an Activity Log entry — who, what, when, success/fail. Retained 90 days by default. DSR ships these to Log Analytics for longer retention.

---

## ⌨️ Activity 1 — Browse Azure Policy assignments

1. Portal → search **Policy** → open it.
2. Left blade → **Compliance**.
3. You'll see policy initiatives applied to your subscription. Click into **Microsoft cloud security benchmark** (the default one) to see compliance state.
4. Click any policy → see **Compliance details** → which resources comply, which don't, and why.

> [!IMPORTANT]
> Don't change anything here. The Preservation Team is not the policy owner. Read-only inspection only.

## ⌨️ Activity 2 — Inventory resources with Resource Graph

The audit pattern: "show me everything tagged with our app and where it sits."

```kusto
Resources
| where tags['app_name'] == 'anl' or tags['purpose'] == 'dsr-training'
| project name, type, resourceGroup, subscriptionId, location, env=tags['env']
| order by type asc
```

Run this in **Resource Graph Explorer**. ✅ You see every resource scoped to your app.

## ⌨️ Activity 3 — Audit access using Resource Graph

```kusto
AuthorizationResources
| where type == "microsoft.authorization/roleassignments"
| extend principalId = tostring(properties.principalId),
         roleDefId = tostring(properties.roleDefinitionId),
         scope = tostring(properties.scope)
| project principalId, roleDefId, scope
```

This lists every role assignment you can see. Combine with `IdentityResources` to resolve principal IDs to names — the full query is in the appendix below.

## ⌨️ Activity 4 — Read the Activity Log for audit

1. Portal → your subscription → **Activity log**.
2. Filter: time = last 7 days, **Operation = Write Storage Account** (or any operation).
3. Click an entry → see the JSON: who (`caller`), what (`operationName`), when (`eventTimestamp`), success/fail.

In production, this is how DIA Cloud Governance proves "no one outside the Platform team modified the storage accounts last quarter."

## ⌨️ Activity 5 — Apply a resource lock

Locks prevent accidental delete or modify, even by Owners.

```bash
az group create -n rg-training-locks-<your-initials> -l australiaeast \
  --tags purpose=dsr-training

az lock create \
  --name protect-from-delete \
  --resource-group rg-training-locks-<your-initials> \
  --lock-type CanNotDelete
```

Now try to delete the RG:

```bash
az group delete -n rg-training-locks-<your-initials> --yes
```

✅ It fails with a lock error.

Remove the lock and retry:

```bash
az lock delete --name protect-from-delete --resource-group rg-training-locks-<your-initials>
az group delete -n rg-training-locks-<your-initials> --yes --no-wait
```

In production, the WOD storage account `stanlnznblobprdwod01` would have a `CanNotDelete` lock applied because of legal hold compliance.

## ⌨️ Activity 6 — Read the Defender for Cloud compliance dashboard

1. Portal → search **Microsoft Defender for Cloud** → **Regulatory compliance**.
2. You'll see compliance against standards (e.g. **NZ ISM**, **ISO 27001**, **CIS Benchmark**).
3. Click any control → see which resources fail and the recommended fix.

> [!IMPORTANT]
> The Preservation Team **reads** this dashboard for awareness. Acting on findings is owned by **DIA Cloud Security** — escalate via your normal incident channel.

## ⌨️ Activity 7 — Watch a policy block something

In a sub with the "all resources must be tagged" policy, try to deploy a storage account without tags:

```bash
az storage account create \
  -n sttest$RANDOM \
  -g rg-training-locks-<your-initials> \
  -l australiaeast --sku Standard_LRS
```

If your sub has the policy assigned, this fails with `RequestDisallowedByPolicy`. The error message names the policy.

✅ This is the typical "why did my deploy fail?" experience. The fix: add the required tags.

---

## 🦾 Now your turn!

Build a Resource Graph "audit pack" query that returns a single result set with:

- All ANL resources missing `env` tag
- All Owner role assignments at sub level (high-privilege)
- All resources without locks in the production resource group

Save it as a **Saved query** in Resource Graph Explorer. This is your weekly audit checklist.

---

## ✅ Success checklist

- [ ] You've browsed Azure Policy compliance
- [ ] You've run a Resource Graph inventory query
- [ ] You've used Activity Log to find who did what
- [ ] You've applied and removed a resource lock
- [ ] You've read the Defender for Cloud compliance dashboard
- [ ] You've understood the escalation path: Preservation reads, others act

---

## 💰 Cost note

< NZD $0.50.

---

## 📎 Appendix — Full role assignment audit query

```kusto
AuthorizationResources
| where type == "microsoft.authorization/roleassignments"
| extend principalId = tostring(properties.principalId)
| extend roleDefId = tostring(properties.roleDefinitionId)
| extend scope = tostring(properties.scope)
| join kind=leftouter (
    AuthorizationResources
    | where type == "microsoft.authorization/roledefinitions"
    | extend roleDefId = tostring(id)
    | project roleDefId, roleName=tostring(properties.roleName)
) on roleDefId
| project principalId, roleName, scope
| order by scope asc
```

---

➡️ **Next:** [Step 05 — Networking primer (read-only view)](step-05-networking.md)
````

---

## 📁 File 7 of N — `step-05-networking.md`

````markdown
# Step 05 — Networking primer (read-only view)

_The "I can read the network without breaking it" lab._ 🌐

> [!NOTE]
> **Duration:** 90 minutes
> **Lab cost:** < NZD $1 (a small VNet + Private Endpoint, torn down at end)
> **Pairs with:** Module 5 of the training plan
> **Networking is owned by DIA Cloud Networking** — this lab is read-only and diagnostic only.

---

## 📖 Session overview

The Digital Preservation Team doesn't configure DSR networking — that's the Cloud Networking team's job. But you do need to *read* the network when troubleshooting Rosetta connectivity (e.g. "the deposit VM can't reach the database"). This session teaches you the network shapes you'll encounter in DSR (VNets, subnets, NSGs, private endpoints, hub-and-spoke, ExpressRoute) and how to interpret them — without giving you the right to change them.

## 🎯 What you'll learn

- What a **VNet** is and how subnets divide it
- How **NSGs** filter traffic and how to read NSG rules
- What **Private Endpoints** and **Private DNS Zones** are and why DSR uses them
- The **Hub-and-Spoke** topology DSR uses
- How **ExpressRoute** connects DIA on-premises to Azure
- How to use **Network Watcher** for read-only diagnostics
- The escalation path when you suspect a network issue

## 📚 Before this session — MS Learn pre-work

| Module | Time | Why this matters |
|---|---|---|
| [Describe Azure networking](https://learn.microsoft.com/training/modules/describe-azure-networking-services/) | 30 min | VNets, subnets, public/private endpoints |
| [Configure Azure virtual networks](https://learn.microsoft.com/training/modules/configure-virtual-networks/) | 45 min | Hands-on with NSGs, peering |
| [Introduction to Azure Private Endpoints](https://learn.microsoft.com/training/modules/introduction-azure-private-link/) | 30 min | Why DSR uses Private Endpoints everywhere |

## 🔤 Acronyms used

- **DNS** = Domain Name System
- **ER** = ExpressRoute. A private fibre link from DIA to Azure.
- **NSG** = Network Security Group. Stateful firewall rules attached to a subnet or NIC.
- **PE** = Private Endpoint. A network interface in your VNet that gives a service a private IP.
- **VNet** = Virtual Network

## ⏱️ EDE accounting

- Trainee self-paced: 90 min
- Instructor-led delivery: 1.5h
- Prep work: 1.75h
- Q&A: 30 min
- **Total EDE per trainee: ~5h**

## 💰 Cost note

- Small VNet: free
- Private Endpoint: ~NZD $0.30/day
- Test storage account: ~NZD $0.03/day
- Tear down within the lab — total < NZD $1.

---

## 🧱 Foundational primer

### Hub-and-Spoke (DSR uses this)

```
┌─────────────────────┐
│  Hub VNet (shared)  │ ← ExpressRoute gateway, Azure Firewall, shared DNS
└──────────┬──────────┘
           │ peering
   ┌───────┴───────┐
   │               │
┌──┴───────┐  ┌────┴─────┐
│ Spoke A  │  │ Spoke B  │  ← ANL workloads (one spoke per env)
│ (DEV)    │  │ (UAT)    │
└──────────┘  └──────────┘
```

DSR has spokes per env: `vnet-anl-dev`, `vnet-anl-uat`, `vnet-anl-prd`. Each is peered to the central hub.

### Subnets (DSR has many — read the design doc table)

| Subnet | What lives there |
|---|---|
| `snet-app-prd` | Rosetta application VMs |
| `snet-ora-prd` | Oracle DB VMs (`rosidbs01/02`) |
| `snet-pe-prd` | Private Endpoints (storage accounts, Key Vault) |
| `snet-agw-prd` | Application Gateway |
| `snet-bastion` | Azure Bastion (jump host) |

### NSG rules (read-only — don't modify)

DSR's `nsg-snet-ora-prd` allows TCP 1521 from `snet-app-prd` only — that's how Rosetta app VMs reach Oracle. If you see this, it's intentional.

### Private Endpoints

Instead of a storage account being on the public internet, a Private Endpoint puts it inside the VNet at a private IP. The DNS zone `privatelink.blob.core.windows.net` rewrites the public hostname to the private IP, so applications don't need to change.

---

## ⌨️ Activity 1 — Inspect the DSR VNet (read-only)

If you have Reader on DSR DEV:

1. Portal → search **Virtual networks** → `vnet-anl-dev`.
2. Note: Address space, Subnets list, Peerings (peers to the hub).
3. Click any subnet → see NSG attached, route table, service endpoints, delegations.

## ⌨️ Activity 2 — Read NSG rules

1. Portal → search **Network security groups** → `nsg-snet-ora-prd` (or DEV equivalent).
2. **Inbound security rules** — see allow/deny rules.
3. Look for the rule allowing TCP 1521 from `snet-app-prd`.
4. Click any rule → see priority, source, destination, action.

## ⌨️ Activity 3 — Trace a network path with Network Watcher

Network Watcher is free and read-only diagnostics.

1. Portal → search **Network Watcher**.
2. Left blade → **Connection troubleshoot**.
3. Source: pick any DSR VM. Destination: pick a storage account FQDN like `stanlnznblobprdrosi01.blob.core.windows.net`.
4. Run the test. ✅ You'll see hop-by-hop reachability.

> [!TIP]
> If a Rosetta VM can't reach storage, this is the first place you look. Output tells you exactly which hop blocks the traffic.

## ⌨️ Activity 4 — Inspect a Private Endpoint

1. Portal → search **Private endpoints**.
2. Pick one in DSR DEV (e.g. `pe-stanlnznfileprdrosi01`).
3. Note: Private IP, Network interface, DNS configuration, Connection state.
4. Click **DNS configuration** → see the FQDN being mapped to a private IP.

## ⌨️ Activity 5 — Hands-on: deploy a tiny VNet + PE in your own sub

```bash
# Resource group
az group create -n rg-training-net-<your-initials> -l australiaeast \
  --tags purpose=dsr-training

# VNet with two subnets
az network vnet create \
  -g rg-training-net-<your-initials> \
  -n vnet-training \
  --address-prefix 10.50.0.0/16 \
  --subnet-name snet-app --subnet-prefix 10.50.1.0/24

az network vnet subnet create \
  -g rg-training-net-<your-initials> \
  --vnet-name vnet-training \
  -n snet-pe \
  --address-prefix 10.50.2.0/24 \
  --disable-private-endpoint-network-policies true

# Storage account to put a PE on
SA=sttraining<your-initials>$RANDOM
az storage account create \
  -n $SA -g rg-training-net-<your-initials> \
  -l australiaeast --sku Standard_LRS --kind StorageV2

# Create Private Endpoint
az network private-endpoint create \
  -g rg-training-net-<your-initials> \
  -n pe-$SA \
  --vnet-name vnet-training --subnet snet-pe \
  --private-connection-resource-id $(az storage account show -n $SA -g rg-training-net-<your-initials> --query id -o tsv) \
  --group-id blob \
  --connection-name myconn
```

Inspect the PE in the portal — you'll see it has a private IP from `10.50.2.0/24`.

## ⌨️ Activity 6 — View effective routes (advanced)

1. Portal → any VM → **Networking** → **Effective routes**.
2. See the actual routing table the OS uses.
3. Identify the route to `0.0.0.0/0` — in DSR, this points to the Azure Firewall in the hub.

## ⌨️ Activity 7 — Tear down

```bash
az group delete -n rg-training-net-<your-initials> --yes --no-wait
```

---

## 🦾 Now your turn!

Use Network Watcher → **NSG diagnostics** to test whether a hypothetical packet from `snet-app-prd` to `snet-ora-prd` on TCP 1521 would be allowed. Document the rule that allows it.

If you don't have access to DSR DEV, simulate it on your training VNet by creating two subnets and an NSG with one rule.

---

## ✅ Success checklist

- [ ] You can read the structure of a VNet (subnets, peerings)
- [ ] You can find the NSG rule allowing Rosetta app → Oracle
- [ ] You've used Network Watcher to trace a connection
- [ ] You've inspected a Private Endpoint
- [ ] You've deployed your own VNet + PE
- [ ] You understand: networking changes go via DIA Cloud Networking, not you

---

## 🚨 Escalation path

If you suspect a network issue causing Rosetta failure:

1. Capture the Network Watcher Connection Troubleshoot output (screenshot).
2. Note the NSG rule that's blocking (if any).
3. Open a ticket with **DIA Cloud Networking**, attaching the diagnostics.
4. Do **not** modify NSG rules yourself.

---

## 💰 Cost note

< NZD $1 if torn down promptly.

---

➡️ **Next:** [Step 06 — Storage accounts deep-dive](step-06-storage-accounts.md)
````

---

## 📊 Progress

**Delivered in this batch:** README + 6 module files (Modules 0–5, Foundations stream complete).

**Remaining:** 21 modules — Storage (6) + Apps (4) + Observability (3) + Cost (2) + Read-only ops (2) + Capstones (4).

Reply "**continue**" and I'll send the next batch (**Storage stream — Modules 6–11**, ~6 files). I'll keep going batch by batch until all 27 are delivered.

If you'd like a different batch order (e.g. capstones first, or Observability first), just say so.