# Step 02 — Identity & access for the operator

_The "who can do what" lab._ 🔐 Builds the access mindset Preservation needs: how RBAC works, the difference between user, service principal, and managed identity, and how to audit access in DSR — without ever needing to grant or change permissions.

> [!NOTE]
> **Trainee duration:** 90 minutes
> **Lab cost:** under NZD $0.50 — only a small test storage account is reused.
> **Prerequisites:** Steps 00 + 01 complete; a test storage account in your `rg-labs-foundations-<your-initials>`.
> **Pairs with:** Module 1 of the DIA training plan (Foundations) — addresses the "RBAC visibility for audit" requirement.

---

## 📖 Session overview

Your team will mostly hold Reader-level access in DSR. Knowing how access works — and how to *audit* it — matters more than knowing how to grant it. This lab teaches you to identify who has access to what, recognise the three identity types you'll see, and use the Access Review tooling that DIA's audit team relies on.

**What you'll learn**
- The difference between a **user**, a **service principal (SP)**, and a **managed identity (MI)**.
- How **RBAC** scopes inherit (subscription → RG → resource).
- The most common roles you'll encounter (Owner, Contributor, Reader, Storage Blob Data Reader, etc.) and what they actually allow.
- How to **review who has access** to a resource — without changing anything.
- How to use **Privileged Identity Management (PIM)** (read-only view) for just-in-time access in DSR.
- How to read **Activity Log** to see who did what, when.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **User** | A human identity in Microsoft Entra (ex-Azure AD). |
| **Service Principal (SP)** | A non-human identity for an app or script. Has a client ID and a secret/certificate. |
| **Managed Identity (MI)** | A special kind of SP that Azure manages for you — no secrets to handle. Used by VMs and apps to authenticate to other Azure services. |
| **RBAC** | Role-Based Access Control. Three pieces: who (identity), what (role), where (scope). |
| **Scope** | Where a role applies. Can be subscription, resource group, or single resource. |
| **Role definition** | The list of allowed actions (e.g. `Microsoft.Storage/storageAccounts/read`). |
| **Role assignment** | Linking an identity to a role at a scope. |
| **PIM** | Privileged Identity Management — gives just-in-time elevated access with approval and time limit. |
| **Activity Log** | The audit trail of management-plane actions (create, modify, delete) on every resource. |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Configure role-based access control (RBAC)](https://learn.microsoft.com/training/modules/secure-azure-resources-with-rbac/) | The core access model in DSR. |
| [Microsoft Entra built-in roles](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles) | The role catalogue — bookmark this. |
| [Application and service principal objects in Microsoft Entra](https://learn.microsoft.com/entra/identity-platform/app-objects-and-service-principals) | DSR has SPs for every automation pipeline. |
| [Plan a Privileged Identity Management deployment](https://learn.microsoft.com/entra/id-governance/privileged-identity-management/pim-deployment-plan) | DIA uses PIM for sensitive subscriptions. |

About **2 hours** of optional pre-reading.

## 🧱 Foundational primer

- **Three identity types**: User (a person), Service Principal (an app/script), Managed Identity (an SP that Azure manages — preferred). DSR uses Managed Identity wherever possible (e.g. Rosetta VMs reading blob storage).
- **Scope is a path**, just like a file system. A role at `/subscriptions/<id>/resourceGroups/rg-foo` applies to every resource under that RG.
- **Roles are additive.** If you're Reader at sub level and Contributor on one RG, you have Contributor on that RG.
- **Built-in roles are usually enough.** Custom roles exist but are rare and need governance approval at DIA.
- **"Owner" is a security concern.** It implies full control plus the ability to grant access to others. Limit it.
- **Data plane vs management plane.** You can have permission to manage a storage account (`Storage Account Contributor`) but no permission to read its data — those are different roles. Critical distinction.
- **Activity Log keeps 90 days** in the portal. For longer retention, DSR exports it to Log Analytics (extended retention per the workspace policy) and Splunk.

## ⌨️ Activity 1 — Inspect role assignments at three scopes

1. Portal → **Subscriptions** → click your subscription → **Access control (IAM)** → **Role assignments** tab.
2. Note who has what at the subscription level. You should see yourself as **Owner**.
3. Click any role to see its definition (the list of allowed actions).
4. Now navigate down: **Resource groups** → your `rg-labs-foundations-*` → **IAM** → **Role assignments**. You'll see the same Owner role inherited from the subscription, plus any RG-specific assignments.
5. Finally: your storage account → **IAM** → **Role assignments**. Same inheritance pattern.

> [!TIP]
> The "Scope" column tells you *where* the role was granted. Inherited assignments show the parent scope; direct assignments show the resource itself.

## ⌨️ Activity 2 — The Storage Blob Data roles

This is where the management-plane vs data-plane distinction matters.

1. Open your storage account → **IAM** → click **+ Add → Add role assignment**.
2. In the role search, type "Storage Blob". Notice three groups:
   - **Storage Blob Data Owner / Contributor / Reader** — these grant access to the *data* (read/write blobs).
   - **Storage Account Contributor** — this grants access to *manage* the storage account (change firewall, create containers) but **does not** let you read blob data.
3. Cancel out (don't actually assign anything).

> [!IMPORTANT]
> In DSR, your team will typically have **Reader** at RG/resource level and **Storage Blob Data Reader** at the storage account level. That means you can see config but not read preserved content — which is what compliance requires.

## ⌨️ Activity 3 — Create a service principal (read-only)

We're going to create a service principal so you can see what one looks like, then immediately remove it.

1. Cloud Shell → run:

```bash
az ad sp create-for-rbac --name "sp-lab-readonly-<your-initials>" \
  --role Reader \
  --scopes /subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-labs-foundations-<your-initials>
```

2. Note the output: `appId` (client ID), `password` (secret — only shown once), `tenant`.
3. In Portal → **Microsoft Entra ID → App registrations** → search for your `sp-lab-readonly-*`. You'll see the SP listed.
4. Open it → **Certificates & secrets** → see the secret you just created (with an expiry).

> [!TIP]
> SP secrets are sensitive. In DSR they're stored only in Key Vault. **Never** check them into Git or paste them into chat.

## ⌨️ Activity 4 — Use Managed Identity instead

Managed Identity is preferred over SP because it removes the secret management burden.

1. Portal → search **Storage account** → your test account → **Settings → Identity**.
2. Under **System assigned** → toggle Status to **On** → Save. (This creates a managed identity for the storage account itself — unusual for storage but a quick demo.)
3. After a few seconds, the storage account has an Object ID. That's its identity.

In DSR production, the **Rosetta VMs** have system-assigned MIs and use them to read from `stanlnznblobprdrosi01` — no secrets, no rotation, no leaks.

## ⌨️ Activity 5 — Read the Activity Log

1. Open your storage account → **Activity log** (left menu).
2. You should see entries for your role-assignment actions, the MI enable, and the original resource creation.
3. Click any entry → see who, what, when, and the JSON of the operation.
4. Filter by **Operation name** = "Create or update role assignment" — this is the search you'd use to spot RBAC changes in DSR.

## ⌨️ Activity 6 — Audit-ready RBAC report (Resource Graph)

This is what DIA's audit team will ask for.

1. Resource Graph Explorer → run:

```kql
authorizationresources
| where type == "microsoft.authorization/roleassignments"
| extend principalId = tostring(properties.principalId),
         roleDefinitionId = tostring(properties.roleDefinitionId),
         scope = tostring(properties.scope)
| where scope contains "rg-labs-foundations"
| project principalId, roleDefinitionId, scope
```

2. This lists every role assignment touching your lab RG.
3. Save the query → name it `Audit — RBAC on lab RG`.
4. Pin to dashboard.

In production, the same query (with a different scope filter) gives DIA's audit team a one-click "who has access to ANL production?" report.

## ⌨️ Activity 7 — Clean up the SP

1. Cloud Shell:

```bash
# Remove the SP
az ad sp delete --id $(az ad sp list --display-name "sp-lab-readonly-<your-initials>" --query "[0].id" -o tsv)
```

2. Verify in Portal → **Entra ID → App registrations** → it's gone.
3. Also turn off the storage account's MI: storage account → **Identity → System assigned → Status: Off → Save**.

## 🦾 Now your turn!

1. In Resource Graph, write a query that returns every role assignment in your subscription, joined to the role definition name (hint: join with `authorizationresources | where type == "microsoft.authorization/roledefinitions"`).
2. From Activity Log, find the exact timestamp at which you enabled the MI on the storage account. What was the operation name?
3. Imagine you're asked: "List every Owner role assignment in your lab RG." Write the Resource Graph query.

## ✅ Success checklist

- [ ] You can name the three identity types (User, SP, MI).
- [ ] You can explain the difference between Storage Account Contributor and Storage Blob Data Contributor.
- [ ] You know which role(s) your DSR account will hold and at which scope.
- [ ] You've run a Resource Graph query returning role assignments and saved it.
- [ ] Your test SP is deleted; the storage account's MI is disabled.

## 📚 Self-serve refresher

- [Common RBAC questions](https://learn.microsoft.com/azure/role-based-access-control/troubleshooting) — what to check when "I can't access this" happens.
- [Resource Graph sample queries — RBAC](https://learn.microsoft.com/azure/governance/resource-graph/samples/starter#azure-rbac) — pre-baked audit queries.

## 💰 Cost note

- Service principals: free.
- Managed identities: free.
- Activity Log: free for 90 days.
- This lab: $0 incremental.

---

⬅️ **Previous:** [Step 01 — Portal & Cloud Shell tour](step-01-portal-and-cloud-shell.md)
➡️ **Next:** [Step 03 — Guardrails, governance & audit](step-03-governance-guardrails.md)
