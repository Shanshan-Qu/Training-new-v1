# Step 01 — Azure foundations

_The "what is Azure made of, anyway?" lab._ 🧭 Builds the mental model every later lab depends on: tenant, subscription, resource group, region, RBAC, ARM, naming and tagging — all anchored to the DIA DSR landing zone you'll be operating.

> [!NOTE]
> **Trainee duration:** 90 minutes
> **Lab cost:** under NZD $0.50 — almost everything used here is free metadata. Deployable in your own Azure free-trial subscription.
> **Prerequisites:** Your personal Azure sandbox is set up — see [lab-environment-setup.md](lab-environment-setup.md). A modern browser. Your DIA work email allowed to sign in to a personal subscription.
> **Pairs with:** Module 1 of the DIA training plan (Foundations).

---

## 📖 Session overview

This session introduces the building blocks of Azure: how Microsoft organises a customer's cloud, where access lives, and how DIA's DSR landing zone is shaped. By the end, you'll be able to log in to your own Azure subscription, navigate the portal, and recognise the resource hierarchy you'll see every day in the DSR environment.

**What you'll learn**
- The difference between a **Microsoft Entra tenant**, a **subscription**, a **resource group**, and a **resource** — and how DIA uses each.
- What an **Azure region** is and why DSR uses **NZ North**.
- The basics of **Azure Resource Manager (ARM)** — the engine behind every click in the portal.
- How **Role-Based Access Control (RBAC)** decides who can do what.
- How DIA's **naming and tagging standards** (`rg-dia-anl-nzn-*`, `app_name=anl`) make the environment readable.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **Tenant** | Your organisation's identity boundary in Microsoft cloud (DIA has one, Microsoft has another). |
| **Subscription** | A billing container. DSR has three: DEV, UAT, PRD. |
| **Resource Group (RG)** | A folder for related resources. Everything inside one RG is usually deleted together. |
| **Resource** | A single thing — a VM, a storage account, a key vault. |
| **ARM** | Azure Resource Manager — the API everything in Azure goes through. |
| **RBAC** | Role-Based Access Control — who can read, write, or own each resource. |
| **Region** | A datacentre location. NZ North = Auckland. |
| **Tag** | A key-value label on a resource. DSR uses `app_name=anl`, `org_name=dia` for cost reporting and filtering. |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Describe Azure architecture and services](https://learn.microsoft.com/training/paths/azure-fundamentals-describe-azure-architecture-services/) | The big picture: regions, AZs, subscriptions, governance — the same concepts DSR uses. |
| [Azure resource manager and resource groups](https://learn.microsoft.com/training/modules/control-and-organize-with-azure-resource-manager/) | Every Portal click is ARM under the hood. |
| [Configure role-based access control (RBAC)](https://learn.microsoft.com/training/modules/secure-azure-resources-with-rbac/) | DSR uses RBAC to keep your team read-only on most things. |
| [Apply and manage tags on Azure resources](https://learn.microsoft.com/training/modules/control-azure-services-with-cli/) | DSR's `app_name=anl` tag is what makes the cost report split work. |

About **2 hours** of optional pre-reading. Not required to start the lab.

## 🧱 Foundational primer

- **Tenant ⊃ Subscription ⊃ Resource Group ⊃ Resource.** Memorise the nesting. Each level has its own access controls.
- **Subscriptions cost money, RGs don't.** A resource group is just an organisational folder; the resources inside it are billable.
- **Region matters.** NZ North (Auckland) keeps DIA data in-country, satisfies sovereignty, and is the closest datacentre to most DIA users. NZ North has no paired region — important when we discuss backup later.
- **Everything is ARM.** Whether you click in the portal, run an Azure CLI command, or apply a Terraform plan, it all becomes a JSON template sent to ARM.
- **Names are immutable.** Once a resource is created, its name is fixed. DIA enforces a naming convention so you can read a name and know what it is.
- **Tags are how cost gets split.** Without tags, Cost Management can only show "Azure cost = $X." With tags, it can show "ANL cost = $X."
- **Reader is your friend.** Most of the time, your Preservation Team account will hold `Reader` on production. That's intentional.

## ⌨️ Activity 1 — Sign in and explore the tenant

1. Go to <https://portal.azure.com> and sign in with your Azure account.
2. Top-right → click your name → **Switch directory**. Note the directory (tenant) name shown — this is the boundary for your identity.
3. Top-left search bar → type **Microsoft Entra ID** → open it. This is the identity service. (You don't need to change anything; just look.)
4. Left menu → **Overview**. Note the **Tenant ID** (a GUID). Every Azure resource ultimately lives inside one tenant.

> [!TIP]
> The tenant is also called the "directory". They are the same thing.

## ⌨️ Activity 2 — Explore subscriptions

1. Search bar → **Subscriptions** → open it.
2. You'll see at least your free-trial subscription. In production DIA has 3 subscriptions for ANL: DEV, UAT, PRD.
3. Click your subscription → **Overview**. Note the **Subscription ID** (a GUID).
4. Left menu → **Resource providers**. Each provider is a category of services Azure can give you (storage, networking, etc.). Note that they need to be **registered** before you can use them — registration is a one-off click that happens automatically the first time you create a resource of that kind.

## ⌨️ Activity 3 — Create your first resource group

1. Search bar → **Resource groups** → **+ Create**.
2. Subscription: your trial. Region: **Australia East** (closest available; in production DIA uses NZ North).
3. Name: `rg-labs-foundations-<your-initials>`.
4. Click **Review + create** → **Create**.
5. Click the new RG → **Tags** → add:
   - `purpose` = `dsr-training`
   - `owner` = your-email@dia.govt.nz
   - `app_name` = `lab`
6. Save.

> [!IMPORTANT]
> Always tag resources you create. The `purpose=dsr-training` + `owner=<you>` tags are what makes cleanup safe — you can search and delete only resources tagged with your email.

## ⌨️ Activity 4 — Read your role assignments

1. On the RG you just created → **Access control (IAM)** → **Role assignments**.
2. Find yourself in the list. You should see role **Owner** at the subscription scope.
3. Click your name → see the role detail. Owner = full read + write + delete + grant access to others.
4. Now mentally compare this to production: in DSR, your account would have **Reader** at the resource group level for most things. That means: can see, cannot change.

> [!TIP]
> RBAC is **inherited downward**. If you're Reader at subscription level, you're Reader on every RG, every resource, automatically. You can be granted *additional* access at lower scopes but cannot be granted *less*.

## ⌨️ Activity 5 — Decode a DIA resource name

DSR resource names follow a strict pattern:

```
rg-dia-anl-nzn-stor-prd
```

Break it down:

| Segment | Meaning |
|---|---|
| `rg-` | Resource group (resource type prefix) |
| `dia` | Organisation: Department of Internal Affairs |
| `anl` | Application: Archives & National Library |
| `nzn` | Region: NZ North |
| `stor` | Function: storage |
| `prd` | Environment: production |

**Practice:** decode these aloud (answers in the success checklist).

1. `vm-rosirep01-prd`
2. `stanlnznblobprdwod01`
3. `kv-anl-nzn-mgt-uat`

## 🦾 Now your turn!

In your `rg-labs-foundations-<your-initials>` resource group:

1. Create a **Storage account** with the smallest config: Standard, LRS, Hot tier, no geo-redundancy. Call it something with `lab` in the name (must be globally unique, all lowercase, no hyphens).
2. Add the same three tags you used on the RG.
3. In the storage account → **Overview** → click the **Resource ID** at the top right (the `/subscriptions/.../...` path). Notice the structure: `/subscriptions/<subId>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<name>`.
4. That string is what ARM uses for everything. Every tool, script, and API call resolves a resource by this path.

## ✅ Success checklist

- [ ] You can name the four levels of the Azure hierarchy in order (tenant → subscription → RG → resource).
- [ ] Your `rg-labs-foundations-<your-initials>` resource group exists with three tags.
- [ ] Your test storage account is created inside that RG.
- [ ] You can decode the three DSR names above.
  - *Answers: (1) Repository VM 01 in production; (2) ANL NZ North blob storage account for WOD in production; (3) Key Vault for ANL in NZ North management RG, UAT environment.*
- [ ] You can explain in one sentence why DIA tags resources with `app_name`.

## 📚 Self-serve refresher (for new starters / your future self)

- [Azure Fundamentals certification path (AZ-900)](https://learn.microsoft.com/certifications/azure-fundamentals/) — free, 8–12 hours, end-to-end recap of everything in this lab.
- [Resource naming best practices](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming) — why DIA's pattern looks the way it does.

## 💰 Cost note

- Resource group: free.
- Storage account empty: ~NZD $0.05/month minimum.
- All resources in this lab can be deleted together by deleting the resource group.

**Cleanup:** Search → Resource groups → click your RG → **Delete resource group** → type the name to confirm. Every resource inside disappears together.

---

➡️ **Next:** [Step 02 — Portal & Cloud Shell tour](step-02-portal-and-cloud-shell.md)
