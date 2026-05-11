# Step 20 — Backup Center read-only operations

> [!IMPORTANT]
> **STATUS: MERGED into [Step 17 — Operational Visibility & Alerts](step-17-azure-monitor.md) (per Emma, 11-May-2026).** Ownership of Backup Center may sit with **Service Reliability**, not Digital Preservation — to be confirmed with Emma. In the meantime, this content is folded into Step 17 as a read-only section. Do not deliver as a standalone session.

_The "did last night's backup actually run?" lab._ 🛡️ Builds the operator view of Backup Center — the single pane DSR uses to confirm RPO compliance across VMs, file shares, blobs, and Oracle backups.

> [!NOTE]
> **Trainee duration:** 90 minutes
> **Lab cost:** under NZD $2 — one tiny VM with a single backup point.
> **Prerequisites:** Steps 01–05 complete. Familiarity with Recovery Services Vaults helps.
> **Pairs with:** Module 4 of the DIA training plan (Observability & Reporting). **Backup ownership:** DSR Cloud Platform owns vault configuration and policy; the Preservation Team owns the *daily compliance read* and escalation.

---

## 📖 Session overview

Backup Center is the cross-resource console for Azure Backup. Every protected VM, file share, and Azure Database backup status surfaces here — with success/failure counts, RPO compliance, and policy summaries. Your team checks Backup Center first thing every morning. This lab walks you through enabling backup on a tiny VM, reading the backup result, querying backup data via KQL, and writing the escalation message that goes to Cloud Platform when something goes wrong.

**What you'll learn**
- The Backup Center hierarchy: **Vaults → Protected items → Backup items → Recovery points**.
- How **Azure Backup** differs from snapshots: vault-stored, retention-policy-driven, restorable cross-region.
- How to **read the daily report**: green/red, RPO breach, retention drift.
- The KQL views over `AddonAzureBackupJobs` and `AddonAzureBackupPolicy`.
- The escalation pattern: what your team actions vs what Cloud Platform actions.
- **Soft delete on the vault** — the safety net against ransomware-style backup deletion.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **Recovery Services Vault (RSV)** | The container for VM/Files backups. |
| **Backup vault** | Newer container for blobs, PostgreSQL, AKS. (Different from RSV.) |
| **Protected item** | A resource the vault is backing up. |
| **Recovery point** | A single backup snapshot, usable for restore. |
| **RPO** | Recovery Point Objective — max acceptable data loss in time. DSR target: 24h for most tiers. |
| **RTO** | Recovery Time Objective — max acceptable downtime. Per-tier; out of scope here. |
| **Soft delete (vault)** | Deleted backups retained 14 days, then permanently removed. |
| **Cross-region restore** | Restore in a paired region. NZ North has no paired region for cross-region restore. |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Introduction to Azure Backup](https://learn.microsoft.com/training/modules/intro-to-azure-backup/) | The mental model. |
| [Monitor Azure Backup with Backup Center](https://learn.microsoft.com/azure/backup/backup-center-overview) | The exact console you'll use daily. |
| [Backup reports](https://learn.microsoft.com/azure/backup/configure-reports) | The KQL views that drive the daily compliance report. |

About **2 hours** of optional pre-reading.

## 🧱 Foundational primer

- **DSR Cloud Platform owns vault configuration.** The Preservation Team has Reader on the vault and acts on what they see.
- **Soft delete on vault is enabled** in DSR. Even if a malicious actor "deleted" a backup, the data is recoverable for 14 days.
- **NZ North has no paired region.** Cross-region restore is not configured. ZRS storage on the vault provides zone redundancy.
- **Per-tier policy varies.** Web tier daily; Index tier daily; Oracle has its own RMAN cycle (Step 15) running in parallel with VM-level Azure Backup.
- **The compliance read = "every protected item has a recovery point in the last 24h"**. Anything else is an exception worth raising.
- **You don't trigger backups manually** in DSR — policies drive them. You read, you don't write.

## ⌨️ Activity 1 — Open Backup Center

1. Search → **Backup center**. (One stop for everything backup-related.)
2. Tabs:
   - **Backup items** — all protected resources across all vaults you can read.
   - **Backup jobs** — every job, success/fail.
   - **Backup policies** — policy assignments.
   - **Vaults** — RSV + Backup vaults.
3. Filter by Subscription/RG to scope to your sub.

## ⌨️ Activity 2 — Enable backup on a small lab VM

Use the lab VM from Step 16 (or any small VM).

1. VM → **Backup**. Or Backup Center → **+ Backup** → Azure resource → Virtual machine.
2. Recovery Services Vault: + Create. Name: `rsv-lab-<initials>`. RG: lab RG. Region: same as VM.
3. Policy: DefaultPolicy (daily, 30-day retention).
4. Enable backup.
5. **Backup → Backup now**. Pick "for 7 days". Submit.
6. Wait ~30–60 min. The first backup run can take 30+ min.

## ⌨️ Activity 3 — Read the result

1. Backup Center → **Backup jobs**. Filter time = last 24h.
2. You should see your backup with status `In progress` then `Completed`.
3. Click into the job → see start/end time, data transferred, recovery point ID.

## ⌨️ Activity 4 — Read the daily compliance picture (KQL)

DSR enables Backup reports — these populate the workspace tables.

```kql
// Jobs in last 24h with status
AddonAzureBackupJobs
| where TimeGenerated > ago(24h)
| where JobOperation == "Backup"
| project TimeGenerated, BackupItemUniqueId, JobStatus,
          JobDurationInSecs, DataTransferredInMB
| summarize success = countif(JobStatus == "Completed"),
            failed = countif(JobStatus == "Failed")
```

```kql
// Items with no recovery point in last 26h (RPO breach)
AddonAzureBackupProtectedInstance
| where TimeGenerated > ago(2d)
| summarize lastRP = max(LatestRecoveryPoint_s) by BackupItemUniqueId
| extend ageHours = datetime_diff('hour', now(), todatetime(lastRP))
| where ageHours > 26
| order by ageHours desc
```

These are the queries Backup Workbook tiles run over.

## ⌨️ Activity 5 — Walk a recovery point

1. Backup Center → **Backup items** → click your VM → **Recovery points**.
2. Pick the one created in Activity 2.
3. Read: snapshot type (Application-consistent), retention, location.
4. Don't restore — but click **File recovery → Mount** for the walkthrough. Cancel before mounting.

## ⌨️ Activity 6 — Soft delete on the vault

1. Vault → **Properties → Security settings**.
2. Confirm **Soft delete**: Enabled.
3. Confirm **Always-on soft delete** (cannot be disabled by an admin without 14-day delay): Enabled in DSR prod.
4. Note: if a backup item is "deleted", it actually goes to soft-deleted state for 14 days.

This is the ransomware safety net.

## ⌨️ Activity 7 — Backup vault (blob, PostgreSQL)

Different surface, same idea. DSR uses Backup vaults for the blob storage accounts.

1. Backup Center → **+ Backup** → Azure resource → Blobs.
2. (You don't need to enable in lab — costs add up. Walk through the wizard, then cancel.)
3. Note operational backup vs vaulted backup — DSR uses operational only on most blobs, with versioning + soft delete as the primary protection.

## ⌨️ Activity 8 — Escalation message template

When you spot a backup compliance breach:

```
Subject: ANL Backup compliance — <resource name> <severity>

What:
  - <resource name> last successful recovery point: <timestamp>
  - Time since last RP: <X hours>
  - Policy expects: <Y hours>

When detected: <UTC timestamp>

Where: <vault name> in <subscription> / <RG>

What I've checked:
  - Backup job log shows <error message or "no recent attempt">
  - VM heartbeat: <healthy/missing>
  - Source resource state: <Running/Stopped/Deallocated>

Impact:
  - RPO breach: <yes/no>
  - Data at risk: <type and approx volume>

Action requested: please investigate vault job-runner / source resource.
Owner: DSR Cloud Platform.
```

This template is reused; keep it on hand.

## 🦾 Now your turn!

1. From Backup Center, find every protected item across the sub. How many distinct policies are in use?
2. Write the KQL for "show every backup job that took longer than 4 hours in the last 7 days, with the resource name". Long-running jobs hint at storage performance issues.
3. Find the **vault's diagnostic settings** — confirm `AzureBackupReport` is being sent to your Log Analytics workspace.
4. Read the DSR Backup Policy table: which Rosetta tier has the shortest retention? Which has the longest?

## ✅ Success checklist

- [ ] You can name the four Backup Center tabs and what each shows.
- [ ] You've enabled backup on a VM and waited for the first recovery point.
- [ ] You can read the daily compliance KQL and understand what an RPO breach looks like.
- [ ] You can identify a vault's soft delete setting.
- [ ] You can write a clean escalation message for a backup compliance issue.
- [ ] You've **deleted** the lab VM's backup and the lab vault to avoid ongoing storage cost.

## 📚 Self-serve refresher

- [Backup Center query reference](https://learn.microsoft.com/azure/backup/backup-azure-monitoring-built-in-monitor) — every table.
- [Configure Backup reports](https://learn.microsoft.com/azure/backup/configure-reports) — when you write your own queries.

## 💰 Cost note

- VM backup: ~NZD $5/month per VM + $0.12/GB instance/month + storage of recovery points.
- Lab cost: < $2 if you delete promptly (within a day or two).

```bash
# Cleanup
# Step 1: stop protection on the VM (keep or delete data)
az backup protection disable -g rg-labs-foundations-<your-initials> \
   -v rsv-lab-<initials> -c <container-name> -i <item-name> --delete-backup-data true --yes
# Step 2: delete the vault (only succeeds once all backup data is removed)
az backup vault delete -g rg-labs-foundations-<your-initials> -n rsv-lab-<initials> --yes
```

---

⬅️ **Previous:** [Step 19 — Cost Management for an application owner](step-19-cost-management.md)
➡️ **Next:** [Step 21 — Defender for Storage (awareness)](step-21-defender-storage.md)
