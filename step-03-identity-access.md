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
