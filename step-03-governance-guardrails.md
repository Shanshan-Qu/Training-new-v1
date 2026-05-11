# Step 03 — Guardrails, governance & audit

> [!IMPORTANT]
> **STATUS: LITE VERSION (per Emma, 11-May-2026).** Platforms team owns guardrails. Reduce to **awareness-only** coverage: how to *read* resource locks, how to query the Activity Log for "who did what when". Drop policy authoring, Defender for Cloud compliance walkthrough, and Resource Graph audit packs.

_The "what stops me breaking production?" lab._ 🛡️ Builds the governance fluency the Preservation Team needs to operate inside DIA's landing zone safely — Azure Policy, resource locks, Activity Log audit, Defender for Cloud compliance, and the Resource Graph audit packs the team runs weekly.

> [!NOTE]
> **Trainee duration:** 90 minutes
> **Lab cost:** under NZD $0.50 — only one resource for lock testing.
> **Prerequisites:** Steps 00–02 (subscription, portal navigation, identity).
> **Pairs with:** Module 1 of the DIA training plan (Foundations) — addresses the "Visibility, governance & audit" feedback that made governance a first-class topic.

---

## 📖 Session overview

This session covers the guardrails Azure and DIA put in place so that mistakes don't escalate into outages. You'll learn what **Azure Policy** is and how to **read** existing policies (DIA's Cloud Governance team owns authoring). You'll learn how to inventory subscriptions, resources, and access for audit. You'll learn how to use the **Activity Log** to answer "who did what when" — and how to apply **resource locks** to protect resources from accidental delete.

The most important takeaway: **the Preservation Team reads governance signals; other DIA teams act on them.** This lab makes that boundary clear and gives you the queries you'll actually run.

**What you'll learn**
- The four governance pillars in Azure: **Policy**, **RBAC**, **Resource Locks**, **Defender for Cloud** — and who owns each at DIA.
- How to **read** existing Azure Policy assignments and compliance state without authoring policies.
- How to use **Resource Graph** to inventory resources and access for audit.
- How to read the **Activity Log** to answer "who did what when".
- How to apply and remove **resource locks** (`CanNotDelete`, `ReadOnly`).
- How to read the **Microsoft Defender for Cloud** compliance dashboard.
- The escalation path when a guardrail blocks something you legitimately need to do.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **Azure Policy** | Auto-enforce rules: "all storage accounts must use ZRS", "no resources without `app_name` tag". Either denies a deployment or audits it. |
| **Initiative** | A bundle of policies (e.g. Microsoft cloud security benchmark = ~200 policies). |
| **Effect** | What a policy does: `Deny`, `Audit`, `Append`, `Modify`, `DeployIfNotExists`. |
| **Resource lock** | `CanNotDelete` blocks delete; `ReadOnly` blocks delete + modify. Applies to all callers including Owners. |
| **Activity Log** | Every Azure control-plane operation (create/update/delete) — who, what, when. 90-day retention by default. |
| **MDC** | Microsoft Defender for Cloud — posture & threat detection across Azure. |
| **Compliance score** | % of policy controls passing in MDC's regulatory compliance view. |
| **Audit pack** | A saved set of Resource Graph / KQL queries you re-run on a schedule. |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Describe Azure governance and compliance features](https://learn.microsoft.com/training/modules/describe-features-tools-azure-for-governance-compliance/) | The big-picture map: Policy, Locks, Cost Mgmt, Defender. |
| [Build a cloud governance strategy on Azure](https://learn.microsoft.com/training/modules/build-cloud-governance-strategy-azure/) | Why governance exists and how it fits real-world ops. |
| [Manage resource locks](https://learn.microsoft.com/azure/azure-resource-manager/management/lock-resources) (article) | Hands-on with the CLI/portal lock commands. |

About **2 hours** of optional pre-reading.

## 🧱 Foundational primer

### The four governance pillars

| Pillar | What it does | Owner at DIA |
|---|---|---|
| **Azure Policy** | Auto-enforce or audit rules | Cloud Governance |
| **RBAC** (Step 02) | Who can do what | Cloud Governance + Platform |
| **Resource Locks** | Prevent accidental delete/modify | Resource owner — your team can apply |
| **Microsoft Defender for Cloud** | Posture & threat detection | Cloud Security |

### Likely policies you'll see effects of (you don't author these)

- "All storage accounts must have HTTPS-only enabled."
- "All resources must have `app_name`, `env`, `org_name` tags."
- "No public network access on storage in PRD."
- "All Recovery Services Vaults must use ZRS."
- "Deploy diagnostic settings to Log Analytics for every storage account."

If a deployment fails with **`RequestDisallowedByPolicy`**, the error names the policy assignment — that's your starting point.

### Activity Log = the audit trail

Every change writes who (`caller`), what (`operationName`), when (`eventTimestamp`), and result. DSR ships these to Log Analytics for >90-day retention. This is how you answer "who deleted that container?".

### Lock semantics

- `CanNotDelete`: read & modify allowed, delete denied.
- `ReadOnly`: only read allowed (often breaks management plane unexpectedly — use sparingly).
- Applied at **subscription**, **resource group**, or **resource** scope. Inherited downwards.

> [!IMPORTANT]
> Locks apply to **everyone**, including subscription Owners. To delete, you must remove the lock first, then delete. Production WOD storage has a `CanNotDelete` lock for legal-hold compliance.

---

## ⌨️ Activity 1 — Browse Azure Policy assignments

1. Portal → search **Policy** → open it.
2. Left blade → **Compliance**.
3. You'll see initiatives applied to your subscription. Click into **Microsoft cloud security benchmark** to see compliance state.
4. Click any policy → **Compliance details** → which resources comply, which don't, and why.

> [!IMPORTANT]
> Don't change anything here. The Preservation Team is not the policy owner. Read-only inspection only.

## ⌨️ Activity 2 — Inventory resources with Resource Graph

The audit pattern: "show me everything tagged with our app, where it sits, and what tags are missing."

```kql
Resources
| where tags['app_name'] == 'anl' or tags['purpose'] == 'dsr-training'
| project name, type, resourceGroup, subscriptionId, location, env=tags['env']
| order by type asc
```

Run in **Resource Graph Explorer**. ✅ Every resource scoped to your app, in one screen.

Bonus: find resources missing the `env` tag:

```kql
Resources
| where tags['app_name'] == 'anl'
| where isempty(tostring(tags['env']))
| project name, type, resourceGroup
```

## ⌨️ Activity 3 — Audit role assignments with Resource Graph

```kql
AuthorizationResources
| where type == "microsoft.authorization/roleassignments"
| extend principalId = tostring(properties.principalId),
         roleDefId = tostring(properties.roleDefinitionId),
         scope = tostring(properties.scope)
| project principalId, roleDefId, scope
```

Combine with `IdentityResources` to resolve principal IDs to display names (full query in the Appendix).

## ⌨️ Activity 4 — Read the Activity Log

1. Portal → your subscription → **Activity log**.
2. Filter: time = last 7 days, **Operation = Write Storage Account** (or any operation).
3. Click an entry → **JSON** tab → see `caller`, `operationName`, `eventTimestamp`, `status`.

In production, this is how DIA Cloud Governance proves "no one outside the Platform team modified the storage accounts last quarter."

You can also query it via KQL once shipped to Log Analytics (you'll do this in Step 12):

```kql
AzureActivity
| where TimeGenerated > ago(7d)
| where ResourceProvider == "Microsoft.Storage"
| where OperationNameValue endswith "/write" or OperationNameValue endswith "/delete"
| project TimeGenerated, Caller, OperationNameValue, ActivityStatusValue, _ResourceId
| order by TimeGenerated desc
```

## ⌨️ Activity 5 — Apply a resource lock

```bash
RG=rg-labs-foundations-<your-initials>
az group create -n $RG -l australiaeast --tags purpose=dsr-training app_name=training env=lab

az lock create \
  --name protect-from-delete \
  --resource-group $RG \
  --lock-type CanNotDelete \
  --notes "Lab: prove locks block delete"
```

Try to delete the RG:

```bash
az group delete -n $RG --yes
```

❌ Fails with a lock error. ✅ Lock works.

Remove the lock and retry:

```bash
az lock delete --name protect-from-delete --resource-group $RG
az group delete -n $RG --yes --no-wait
```

In production, the WOD storage account `stanlnznblobprdwod01` has a `CanNotDelete` lock applied because of legal-hold compliance. (Immutability policies themselves are owned by Cloud Governance and out of scope for this training.)

## ⌨️ Activity 6 — Read the Defender for Cloud compliance dashboard

1. Portal → search **Microsoft Defender for Cloud** → **Regulatory compliance**.
2. You'll see compliance against standards (e.g. **NZ ISM**, **ISO 27001**, **CIS Benchmark**).
3. Click any control → see which resources fail and the recommended fix.
4. Note the overall **secure score** at the top.

> [!IMPORTANT]
> The Preservation Team **reads** this dashboard for awareness. Acting on findings is owned by **DIA Cloud Security** — escalate via your normal incident channel. Do not "fix" findings yourself in production.

## ⌨️ Activity 7 — Watch a policy block something

In a sub with a "all resources must be tagged" policy, try to deploy a storage account without tags:

```bash
az storage account create \
  -n sttest$RANDOM \
  -g $RG \
  -l australiaeast --sku Standard_LRS
```

If the policy is assigned, this fails with **`RequestDisallowedByPolicy`**. The error names the policy assignment.

✅ This is the typical "why did my deploy fail?" experience. The fix: add the required tags. Try again with `--tags app_name=training env=lab purpose=dsr-training`.

## ⌨️ Activity 8 — Build your weekly audit pack

Save these as **saved queries** in Resource Graph Explorer:

1. **Untagged resources** in your application:

   ```kql
   Resources
   | where tags['app_name'] == 'anl'
   | where isempty(tostring(tags['env'])) or isempty(tostring(tags['org_name']))
   | project name, type, resourceGroup, missingTags = pack('env', tags['env'], 'org_name', tags['org_name'])
   ```

2. **High-privilege role assignments** at subscription scope (Owner / User Access Administrator):

   ```kql
   AuthorizationResources
   | where type == "microsoft.authorization/roleassignments"
   | extend scope = tostring(properties.scope), roleDefId = tostring(properties.roleDefinitionId)
   | where scope startswith "/subscriptions/"
   | where scope !contains "/resourcegroups/"
   ```

3. **Storage accounts without locks** in production resource groups:

   ```kql
   Resources
   | where type == "microsoft.storage/storageaccounts"
   | where resourceGroup contains "prd"
   | join kind=leftouter (
       resourcecontainers
       | where type == "microsoft.authorization/locks"
     ) on $left.id == $right.id
   | where isempty(name1)
   | project name, resourceGroup, subscriptionId
   ```

These three queries are your weekly governance health check.

## 🦾 Now your turn!

1. Find one Azure Policy assignment that targets storage accounts. Read its **definition JSON** and explain in plain English what it does.
2. Apply a `ReadOnly` lock to a test storage account, then try to upload a blob — capture the error message.
3. Run the **untagged resources** query against your training subscription. Fix any untagged resources by adding tags via CLI.
4. Open Defender for Cloud → pick one **High** severity recommendation against your training resources → write the escalation note you'd send to Cloud Security (don't act on it).
5. Find the Activity Log entry for any change you made today. Confirm `caller` matches your account.

## ✅ Success checklist

- [ ] You can name the four governance pillars and the DIA owner of each.
- [ ] You can read an existing Azure Policy assignment without changing it.
- [ ] You've applied and removed both `CanNotDelete` and `ReadOnly` locks.
- [ ] You've used the Activity Log to find who did what.
- [ ] You've run the three weekly audit-pack queries.
- [ ] You've read the Defender for Cloud regulatory compliance dashboard.
- [ ] You understand the boundary: **Preservation reads, others act**.
- [ ] You've **deleted** any lab locks and resource groups.

## 📚 Self-serve refresher

- [Azure Policy effects](https://learn.microsoft.com/azure/governance/policy/concepts/effects) — Deny vs Audit vs Modify vs DeployIfNotExists.
- [Resource Graph sample queries — governance](https://learn.microsoft.com/azure/governance/resource-graph/samples/starter#governance) — copy-paste audit templates.
- [Activity Log schema](https://learn.microsoft.com/azure/azure-monitor/essentials/activity-log-schema) — the exact field names for KQL.
- [Defender for Cloud overview](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-cloud-introduction) — what the secure score actually means.

## 🚨 Escalation paths

| You see | Escalate to |
|---|---|
| Policy unexpectedly denies a legitimate deploy | DIA Cloud Governance |
| MDC High severity on DSR storage | DIA Cloud Security |
| Lock missing where one is required (e.g. WOD) | DIA Platform + log incident |
| Activity Log shows unauthorised change | DIA Cloud Security + your team lead |

## 💰 Cost note

Under NZD $0.50 — locks, policies, Activity Log, MDC dashboard reading and Resource Graph queries are all free; one resource group with no resources costs nothing.

```bash
# Cleanup
az lock delete --name protect-from-delete --resource-group rg-labs-foundations-<your-initials> 2>/dev/null
az group delete -n rg-labs-foundations-<your-initials> --yes --no-wait
```

## 📎 Appendix — Full role-assignment audit query

```kql
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

⬅️ **Previous:** [Step 02 — Identity & access for the operator](step-02-identity-and-access.md)
➡️ **Next:** [Step 04 — Networking primer (read-only view)](step-04-networking-primer.md)
