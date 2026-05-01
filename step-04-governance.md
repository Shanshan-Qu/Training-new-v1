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
