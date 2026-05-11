# Step 03 — Guardrails, governance & audit (lite)

> [!NOTE]
> **This is the lite version (per Emma, 11-May-2026).** Platforms team owns guardrails at DIA. Scope is **awareness-only**: read existing resource locks, query the Activity Log for "who did what when", recognise when a policy has blocked a deployment. Policy authoring, Defender for Cloud compliance, and weekly Resource Graph audit packs are **out of scope** for the DP team.

_The "what stops me breaking production?" lab._ 🛡️ Awareness-only coverage of the guardrails Platforms applies to the DSR landing zone, and the two operations the DP team actually performs: read the Activity Log, apply/remove a resource lock.

> [!NOTE]
> **Trainee duration:** 45 minutes
> **Lab cost:** under NZD $0.50 — only one resource for lock testing.
> **Prerequisites:** Steps 00–02 (subscription, portal navigation, identity).
> **Pairs with:** Module 1 of the DIA training plan (Foundations).

---

## 📖 Session overview

Azure and DIA put guardrails in place so that mistakes don't escalate into outages. The Preservation Team does **not** author these guardrails — Cloud Governance and Platforms do — but you do need to recognise them when they bite, and you do need two specific skills: **reading the Activity Log** ("who did what when") and **applying a resource lock** to protect something you own.

The most important takeaway: **the Preservation Team reads governance signals; other DIA teams act on them.**

**What you'll learn**
- The four governance pillars in Azure: **Policy**, **RBAC**, **Resource Locks**, **Defender for Cloud** — and who owns each at DIA (awareness only).
- How to recognise the `RequestDisallowedByPolicy` error and where to escalate.
- How to read the **Activity Log** to answer "who did what when".
- How to apply and remove **resource locks** (`CanNotDelete`, `ReadOnly`).
- The escalation path when a guardrail blocks something you legitimately need to do.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **Azure Policy** | Auto-enforce rules: "all storage accounts must use ZRS", "no resources without `app_name` tag". Either denies a deployment or audits it. |
| **Effect** | What a policy does: `Deny`, `Audit`, `Append`, `Modify`, `DeployIfNotExists`. |
| **Resource lock** | `CanNotDelete` blocks delete; `ReadOnly` blocks delete + modify. Applies to all callers including Owners. |
| **Activity Log** | Every Azure control-plane operation (create/update/delete) — who, what, when. 90-day retention by default. |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Describe Azure governance and compliance features](https://learn.microsoft.com/training/modules/describe-features-tools-azure-for-governance-compliance/) | The big-picture map: Policy, Locks, Cost Mgmt, Defender. |
| [Manage resource locks](https://learn.microsoft.com/azure/azure-resource-manager/management/lock-resources) (article) | Hands-on with the CLI/portal lock commands. |

About **45 minutes** of optional pre-reading.

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

If a deployment fails with **`RequestDisallowedByPolicy`**, the error names the policy assignment — that's your starting point. **Don't try to "fix" the policy; escalate to Cloud Governance.**

### Activity Log = the audit trail

Every change writes who (`caller`), what (`operationName`), when (`eventTimestamp`), and result. DSR ships these to Log Analytics for >90-day retention. This is how you answer "who deleted that container?".

### Lock semantics

- `CanNotDelete`: read & modify allowed, delete denied.
- `ReadOnly`: only read allowed (often breaks management plane unexpectedly — use sparingly).
- Applied at **subscription**, **resource group**, or **resource** scope. Inherited downwards.

> [!IMPORTANT]
> Locks apply to **everyone**, including subscription Owners. To delete, you must remove the lock first, then delete. Production WOD storage has a `CanNotDelete` lock for legal-hold compliance.

---

## ⌨️ Activity 1 — Browse Azure Policy assignments (read only)

1. Portal → search **Policy** → open it.
2. Left blade → **Compliance**.
3. You'll see initiatives applied to your subscription. Click into **Microsoft cloud security benchmark** to see compliance state.
4. Click any policy → **Compliance details** → which resources comply, which don't, and why.

> [!IMPORTANT]
> Don't change anything here. The Preservation Team is not the policy owner. Read-only inspection only.

## ⌨️ Activity 2 — Read the Activity Log

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

## ⌨️ Activity 3 — Apply a resource lock

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

## 🦾 Now your turn!

1. Find one Azure Policy assignment that targets storage accounts. Read its **definition JSON** and explain in plain English what it does — no changes.
2. Apply a `ReadOnly` lock to a test storage account, then try to upload a blob — capture the error message.
3. Find the Activity Log entry for any change you made today. Confirm `caller` matches your account.

## ✅ Success checklist

- [ ] You can name the four governance pillars and the DIA owner of each.
- [ ] You can read an existing Azure Policy assignment without changing it.
- [ ] You've applied and removed both `CanNotDelete` and `ReadOnly` locks.
- [ ] You've used the Activity Log to find who did what.
- [ ] You understand the boundary: **Preservation reads, others act**.
- [ ] You've **deleted** any lab locks and resource groups.

## 📚 Self-serve refresher

- [Azure Policy effects](https://learn.microsoft.com/azure/governance/policy/concepts/effects) — Deny vs Audit vs Modify vs DeployIfNotExists.
- [Activity Log schema](https://learn.microsoft.com/azure/azure-monitor/essentials/activity-log-schema) — the exact field names for KQL.

## 🚨 Escalation paths

| You see | Escalate to |
|---|---|
| Policy unexpectedly denies a legitimate deploy (`RequestDisallowedByPolicy`) | DIA Cloud Governance |
| Lock missing where one is required (e.g. WOD) | DIA Platform + log incident |
| Activity Log shows unauthorised change | DIA Cloud Security + your team lead |
| Defender for Cloud alert on DSR storage | DIA Cloud Security (you do **not** investigate) |

## 💰 Cost note

Under NZD $0.50 — locks, policies, Activity Log are all free; one resource group with no resources costs nothing.

```bash
# Cleanup
az lock delete --name protect-from-delete --resource-group rg-labs-foundations-<your-initials> 2>/dev/null
az group delete -n rg-labs-foundations-<your-initials> --yes --no-wait
```

---

⬅️ **Previous:** [Step 02 — Identity & access for the operator](step-02-identity-and-access.md)
➡️ **Next:** [Step 04 — Networking primer (read-only view)](step-04-networking-primer.md)
