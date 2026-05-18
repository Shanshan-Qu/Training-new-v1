# Step 06 — Azure Files for DSR (operational view)

> [!NOTE]
> **Trimmed lab.** The team already knows SMB/NFS protocols, POSIX permissions, ACLs, and how to mount a share. This lab covers only the DSR-specific operations: reading `stalnznznfileprdrosi01/02` configuration, throttling diagnostics, and the mount-troubleshooting runbook.

_The "shared filesystems for Rosetta" lab._ 📁 Operational view of Azure Files as DSR runs it.

> [!NOTE]
> **Trainee duration:** 60 minutes
> **Lab cost:** $0 — read-only against production + a metrics chart on the lab storage account from Step 05.
> **Prerequisites:** Steps 00–05 complete. Reader on the DSR production subscription. The production Azure Files storage accounts (`stalnznznfileprdrosi01` and `stalnznznfileprdrosi02`) must already be provisioned and accessible — this lab is read-only against them and does **not** cover provisioning a new FileStorage account or share.
> **Pairs with:** Module 2 of the DIA training plan (Storage).

---

## 📖 Session overview

Rosetta uses Azure Files for shared deposit / IE staging directories. Two file shares live on dedicated FileStorage accounts (`stalnznznfileprdrosi01` and `02`). Your team needs to be able to: **read share configuration**, **diagnose throttling**, **work the mount-troubleshooting runbook**, and **confirm soft-delete posture**. Protocol theory, POSIX permissions, and provisioning a Premium share from scratch are **out of scope** — already in your day-job toolkit.

**What you'll learn**
- How to read `stalnznznfileprdrosi01/02` config from Resource Graph.
- The Azure Files metrics that matter: `Transactions` (with `SuccessThrottlingError`) and `SuccessE2ELatency`.
- The KQL query for "is Rosetta slow because the share is being throttled?"
- The mount-troubleshooting runbook (4 common failure modes).
- How to verify soft-delete is configured on production shares.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **FileStorage account kind** | The specialised storage account kind that hosts Premium file shares. DSR's `stalnznznfileprd*` accounts. |
| **Provisioned size** | On Premium, you pre-pay X GiB; **IOPS scale with size** (baseline = `3000 + 4 × GiB`). |
| **Throttling** | Exceeding provisioned IOPS / throughput. Shows as latency spikes and `SuccessThrottlingError` on the `Transactions` metric. |
| **Soft delete** | Recover deleted shares / files within a retention window. DSR = 14 days on production. |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for DSR |
|---|---|
| [Premium file shares performance](https://learn.microsoft.com/azure/storage/files/understand-performance) | IOPS / throughput / burst — what to watch on metrics. |
| [Troubleshoot Azure Files mounting](https://learn.microsoft.com/azure/storage/files/storage-troubleshoot-linux-file-connection-problems) | The exact runbook for "share won't mount". |

About **30 minutes** of optional pre-reading.

## 🧱 Foundational primer

- **DSR pattern:** Premium FileStorage, **NFS v4.1**, ZRS, Private Endpoint only, public network access disabled, 14-day share-soft-delete.
- **IOPS baseline formula:** `3,000 + 4 × GiB`. Burst: `10,000 + 200 × GiB` for up to 1 hour. Bigger share = more IOPS.
- **Throttling is latency, not always errors.** Watch `Transactions` (filter ResponseType=`SuccessThrottlingError`) **and** `SuccessE2ELatency` together.
- **Quota changes go through change control** at DIA — but recognising "we're throttling, we need more GiB" is yours.

---

## ⌨️ Activity 1 — Read production file share config

Resource Graph Explorer:

```kql
resources
| where type == "microsoft.storage/storageaccounts"
| where kind == "FileStorage"
| where name has "stalnznznfileprd"
| project name,
          sku = sku.name,
          location, resourceGroup, subscriptionId
```

Expect: `kind = FileStorage`, `sku = Premium_ZRS`.

Then the shares inside:

```kql
resources
| where type == "microsoft.storage/storageaccounts/fileservices/shares"
| where name has "stalnznznfileprd"
| project name,
          shareQuota = properties.shareQuota,
          enabledProtocols = properties.enabledProtocols,
          rootSquash = properties.rootSquash
```

Confirm: `enabledProtocols = NFS`, `rootSquash = RootSquash` (production hardening).

## ⌨️ Activity 2 — Read share metrics

Pick one of the production file storage accounts in the portal (you have Reader):

1. Account → **Metrics**.
2. Metric: `Transactions`. Aggregation: Sum. Add filter: **ResponseType = `SuccessThrottlingError`**.
3. Chart `SuccessE2ELatency` alongside.
4. Chart `Egress` — bytes read from the share — to correlate with deposit batches.

> [!IMPORTANT]
> Throttling spikes that coincide with the nightly deposit batch are usually expected. Throttling outside that window is your "Rosetta is slow" signal.

## ⌨️ Activity 3 — KQL: throttling on real account

```kql
AzureMetrics
| where Resource has "stalnznznfileprd"
| where MetricName == "Transactions"
| where parse_json(Properties).ResponseType == "SuccessThrottlingError"
| summarize throttled = sum(Total) by Resource, bin(TimeGenerated, 5m)
| order by TimeGenerated desc
```

Run against the production Log Analytics workspace. Any 5-minute bucket with >0 throttled transactions is worth a glance.

## ⌨️ Activity 4 — Walk the mount-troubleshooting runbook

Memorise this table — it's the first thing you check on a "Rosetta can't reach the share" ticket. **Do not** attempt to mount anything yourself in this lab — that's protocol practice the team already has.

| Symptom | Likely cause | First action |
|---|---|---|
| Permission denied at mount | Storage firewall blocking client IP | Check Networking → IP allowlist / Private Endpoint. |
| No route to host | Private Endpoint DNS not resolving to private IP | `nslookup <sa>.file.core.windows.net` from the client. |
| Stale file handle | NFS server restarted (rare) | `umount -f` then re-mount. |
| Slow performance | Throttling / share undersized | Run Activity 2/3; consider quota increase via change control. |

## ⌨️ Activity 5 — Verify soft-delete posture

```kql
resources
| where type == "microsoft.storage/storageaccounts/fileservices"
| where id has "stalnznznfileprd"
| project id,
          shareDeleteRetentionEnabled = properties.shareDeleteRetentionPolicy.enabled,
          retentionDays = properties.shareDeleteRetentionPolicy.days
```

Expect `enabled = true`, `days = 14` on every production file service. If not, that's a finding for Cloud Governance.

## 🦾 Now your turn!

1. Pick a 30-day window in `AzureMetrics`. Plot throttled transactions for `stalnznznfileprdrosi01` and `stalnznznfileprdrosi02`. Are the spikes correlated (shared workload) or independent (per-share workload)?
2. From Resource Graph, write a query that flags any FileStorage account in DSR where soft-delete is **not** set to 14 days.
3. Write a one-paragraph escalation note for "Rosetta team reports share is slow at 14:00 daily" — what you'd put in the ticket, which metrics, who you'd cc.

## ✅ Success checklist

- [ ] You can read `stalnznznfileprdrosi01/02` config from Resource Graph without help.
- [ ] You can pull up throttling metrics for a production file share in under a minute.
- [ ] You can recite the 4-row mount-troubleshooting table from memory.
- [ ] You can verify soft-delete posture across all DSR file accounts in one KQL query.
- [ ] You know quota changes go through change control, not direct portal edits.

## 🚨 Escalation paths

| You see | Escalate to |
|---|---|
| Persistent throttling outside the nightly deposit window | DIA Platform + Rosetta application owner |
| Mount fails after working previously, DNS resolves correctly | DIA Platform (Private Endpoint / firewall change?) |
| Soft-delete missing or <14 days on production share | DIA Cloud Governance |
| File share quota needs to increase | DIA Platform via change control |

## 📚 Self-serve refresher

- [Performance and scale targets](https://learn.microsoft.com/azure/storage/files/storage-files-scale-targets) — current IOPS/throughput formulas.
- [Troubleshoot Azure Files performance](https://learn.microsoft.com/azure/storage/files/storage-troubleshooting-files-performance) — when latency spikes.

## 💰 Cost note

This lab is **$0** — all activities are read-only against production + a metrics chart on the lab account you already have from Step 05. No Premium share deployment (which would cost ~NZD $20/month minimum prepaid).

---

⬅️ **Previous:** [Step 05 — Storage accounts & tier management](step-05-storage-accounts-and-tiers.md)
➡️ **Next:** [Step 07 — Data protection posture](step-07-data-protection.md)
