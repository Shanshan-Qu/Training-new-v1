# Step 09 — Azure Files (NFS & SMB)

> [!IMPORTANT]
> **STATUS: TRIMMED (per Emma, 11-May-2026).** Team is already familiar with POSIX vs ACL and SMB/NFS basics. Cut the generic protocol primer and the permissions theory. Keep only DSR-specific items: `stanlnznfileprdrosi01/02` configuration, mount troubleshooting, throttling/IOPS limits, performance-tier read.

_The "shared filesystems for Rosetta" lab._ 📁 Builds the working knowledge of Azure Files the team needs to operate `stanlnznfileprdrosi01/02` — NFS vs SMB, mounting, POSIX/Kerberos permissions, performance, and throttling.

> [!NOTE]
> **Trainee duration:** 120 minutes
> **Lab cost:** under NZD $5 — one Premium FileStorage account (NFS or SMB) for the duration of the lab. **Delete promptly after.**
> **Prerequisites:** Steps 01–05 complete. A Linux VM you can SSH to (or a small Cloud Shell mount).
> **Pairs with:** Module 2 of the DIA training plan (Storage).

---

## 📖 Session overview

Rosetta uses Azure Files for shared deposit / IE staging directories. Two file shares live on dedicated FileStorage accounts (`stanlnznfileprdrosi01` and `02`). Your team needs to be able to: read share configuration, recognise NFS vs SMB shares, recognise the **performance tier**, troubleshoot mount problems with read-only diagnostics, and understand **throttling** and IOPS limits. We won't deploy Rosetta itself — we'll deploy a lab share, mount it, and exercise the same operations Rosetta does.

**What you'll learn**
- The two **share protocols**: SMB (Windows-style) and NFS v4.1 (Linux-style).
- The **performance tiers**: Standard (transaction-priced) and Premium (provisioned IOPS / GiB).
- How **provisioned size** drives **IOPS, throughput, burst** on Premium.
- The two **identity / auth options**: storage-account key vs Active Directory (Kerberos / Entra Domain Services).
- POSIX vs ACL permission models.
- How to read **throttling metrics** and what to do about them.
- The DSR pattern: NFS v4.1 Premium with Private Endpoint.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **SMB** | Server Message Block — the Windows file-sharing protocol. Works on Linux too. |
| **NFS v4.1** | Network File System — Unix/Linux file-sharing protocol. DSR uses this for Rosetta. |
| **FileStorage account kind** | A specialised storage account kind that supports Premium file shares. |
| **Provisioned size** | On Premium, you pre-pay for X GiB; IOPS/throughput scale with X. |
| **IOPS** | I/O operations per second — how many reads/writes per second the share allows. |
| **Burst credits** | Allowance to exceed baseline IOPS for short periods. |
| **Throttling** | When you exceed IOPS/throughput, the share returns errors or slows responses (HTTP 503, NFS slow). |
| **Mount** | Attaching the file share to a VM so it appears as a local folder. |
| **POSIX** | Standard Unix file permissions (owner/group/other × read/write/execute). |
| **Kerberos** | The Microsoft-Entra/AD authentication protocol — required for SMB-with-AD shares. |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Azure Files overview](https://learn.microsoft.com/azure/storage/files/storage-files-introduction) | The model behind every share. |
| [NFS file shares in Azure Files](https://learn.microsoft.com/azure/storage/files/files-nfs-protocol) | The protocol DSR uses for Rosetta. |
| [Premium file shares performance](https://learn.microsoft.com/azure/storage/files/understand-performance) | IOPS / throughput / burst — what to watch on metrics. |
| [Troubleshoot Azure Files mounting](https://learn.microsoft.com/azure/storage/files/storage-troubleshoot-linux-file-connection-problems) | The exact runbook for "share won't mount". |

About **2.5 hours** of optional pre-reading.

## 🧱 Foundational primer

- **Premium NFS shares** require a **FileStorage** kind storage account, **NFS v4.1** protocol, and **disabled "secure transfer required"** (encrypts in transit at the network layer instead).
- **NFS shares are Linux-only** at mount time; SMB shares mount on Windows or Linux.
- **Premium IOPS = 3,000 + (4 × GiB) baseline**, with **10,000 + (200 × GiB) burst** for up to 1 hour. Bigger share = more IOPS.
- **Standard tier shares are billed by transactions and storage**, like blob storage. Cheaper at low traffic, surprisingly expensive at high traffic.
- **NFS uses POSIX permissions** (uid/gid). The mounting client must have UID/GID that match what Rosetta expects.
- **SMB with Entra Domain Services** uses Kerberos. The VM must be domain-joined.
- **Private endpoint is mandatory in DSR.** Public access on the file share = audit finding.
- **Throttling shows up as latency** — not always errors. Watch the **`Transactions`** and **`SuccessE2ELatency`** metrics together.

## ⌨️ Activity 1 — Read the production file share config

Resource Graph:

```kql
resources
| where type == "microsoft.storage/storageaccounts"
| where kind == "FileStorage"
| project name,
          sku = sku.name,
          provisionedIOPS = properties.fileShareProperties,
          location, resourceGroup, subscriptionId
```

You'll see DSR's two `stanlnznfileprdrosi01/02` accounts (when you have Reader on the prod sub). Note: kind = FileStorage, sku = Premium_LRS or Premium_ZRS.

To read the share inside:

```kql
resources
| where type == "microsoft.storage/storageaccounts/fileservices/shares"
| where name has "stanlnznfileprd"
| project name,
          shareQuota = properties.shareQuota,
          enabledProtocols = properties.enabledProtocols,
          rootSquash = properties.rootSquash
```

## ⌨️ Activity 2 — Create your own NFS Premium file share

1. Portal → **Storage accounts → + Create**.
2. RG: `rg-labs-foundations-<your-initials>`. Name: `stlabnfsprm<initials><nn>`.
3. Region: Australia East. **Performance: Premium**. Premium account type: **File shares**.
4. Replication: ZRS.
5. **Advanced** tab → **Secure transfer required: Disabled** (required for NFS).
6. **Networking → Public network access: Enabled from selected networks** + add your IP. (Production = Disabled with Private Endpoint only.)
7. Create.

8. Open the new account → **File shares → + File share**.
9. Name: `lab-nfs`. Provisioned capacity: 100 GiB (minimum on Premium).
10. **Protocol: NFS**. **Root squash: NoRootSquash** (lab only — production uses RootSquash).
11. Create.

> [!IMPORTANT]
> The 100 GiB minimum on Premium is **prepaid**. It costs about NZD $20/month. **Delete the account at the end of this lab.**

## ⌨️ Activity 3 — Mount the share from a Linux client

You need a Linux VM. Cheapest option: deploy a tiny Standard B1s Ubuntu VM in the **same VNet** as a Private Endpoint (or use public access + your IP for this lab).

```bash
# On the Linux VM
sudo apt update && sudo apt install -y nfs-common
sudo mkdir -p /mnt/lab-nfs

SA=<your storage account>
sudo mount -t nfs -o vers=4,minorversion=1,sec=sys $SA.file.core.windows.net:/$SA/lab-nfs /mnt/lab-nfs

# Test
sudo touch /mnt/lab-nfs/hello.txt
ls -l /mnt/lab-nfs
```

If the mount fails, the most common causes are:

| Symptom | Likely cause | Fix |
|---|---|---|
| Permission denied | Storage firewall blocking client IP | Check Networking → IP allowlist. |
| No route to host | Private Endpoint DNS not resolving | Check DNS resolves to private IP. |
| Stale file handle | NFS server restarted | `umount -f` then re-mount. |
| Slow performance | Throttling / undersized share | Check Metrics → Provisioned IOPS exhaustion. |

## ⌨️ Activity 4 — Read share metrics

1. Storage account → **Metrics**.
2. Metric: `Transactions`. Aggregation: Sum. Add filter: ResponseType = `SuccessThrottlingError`.
3. If the count is non-zero, you're being throttled — the workload exceeds provisioned IOPS.
4. Also chart `SuccessE2ELatency` and `Egress` (data read from share).

In production this is your first read for "Rosetta is slow today" tickets.

## ⌨️ Activity 5 — Inspect throttling on a real account (read-only on DSR)

```kql
AzureMetrics
| where Resource has "stanlnznfileprd"
| where MetricName == "Transactions"
| where parse_json(Properties).ResponseType == "SuccessThrottlingError"
| summarize throttled = sum(Total) by Resource, bin(TimeGenerated, 5m)
| order by TimeGenerated desc
```

Throttling spikes correlate to deposit batches or backup runs — often expected, but the workbook in Step 18 will surface them.

## ⌨️ Activity 6 — Read POSIX permissions

On the mounted share:

```bash
ls -ln /mnt/lab-nfs        # numeric uid/gid
sudo chown 1001:1001 /mnt/lab-nfs/hello.txt
ls -ln /mnt/lab-nfs/hello.txt
```

In production, the Rosetta service account runs as a specific UID (e.g. 60000). The share must have files owned by 60000 or readable by 60000's group. UID mismatches are a common cause of "Rosetta can't see the file".

## ⌨️ Activity 7 — Increase share quota and watch IOPS rise

1. File share → **Quota → Increase to 500 GiB**.
2. Note in the portal: provisioned IOPS goes from 3,400 → 5,000; burst from 10,200 → 110,000.
3. This is how you scale capacity for Rosetta during deposit-heavy days — the actual change is small and reversible.

> [!TIP]
> You can also resize via Cloud Shell: `az storage share update --name lab-nfs --account-name $SA --quota 500`. In DSR, all resize operations go through change control — but the *recognise* step is yours.

## 🦾 Now your turn!

1. Run a small load test: copy 10,000 small files into the share with `dd` or a simple loop, then watch the metric `Transactions` and `SuccessE2ELatency`. Was throttling triggered?
2. From Resource Graph, write a query that returns every Premium FileStorage account in the subscription and its provisioned size.
3. Mount the same share on a *second* Linux client (or via Cloud Shell). What happens to file locks if both write the same file? (NFS v4.1 supports byte-range locks — try `flock`.)
4. Find the share's **soft delete** setting. Is it enabled? (DSR has 14-day soft delete on production file shares.)

## ✅ Success checklist

- [ ] You can name the two share protocols and which DSR uses for Rosetta.
- [ ] You can name the formula for Premium baseline IOPS (`3000 + 4 × GiB`).
- [ ] You can mount an NFS v4.1 share from Linux.
- [ ] You can read throttling metrics and explain "Rosetta is slow" with data.
- [ ] You understand POSIX UID/GID and how the wrong UID causes access errors.
- [ ] You've cleaned up — Premium accounts are not free to leave running.

## 📚 Self-serve refresher

- [Mount NFS file share on Linux](https://learn.microsoft.com/azure/storage/files/storage-files-how-to-mount-nfs-shares) — the mount runbook.
- [Performance and scale targets](https://learn.microsoft.com/azure/storage/files/storage-files-scale-targets) — current IOPS/throughput formulas.
- [Troubleshoot Azure Files performance](https://learn.microsoft.com/azure/storage/files/storage-troubleshooting-files-performance) — when latency spikes.

## 💰 Cost note

- Premium FileStorage minimum: 100 GiB ≈ NZD $20/month.
- A few hours of Premium share for the lab: ~NZD $0.10 if deleted promptly.
- **Cleanup is critical here.** Delete the storage account itself, not just the share — Premium accounts are billed even with empty shares.

```bash
# Cleanup
az storage account delete -g rg-labs-foundations-<your-initials> -n <your account> --yes
```

---

⬅️ **Previous:** [Step 08 — Cold tier retrieval & rehydration cost](step-08-cold-retrieval.md)
➡️ **Next:** [Step 10 — Data protection posture](step-10-data-protection.md)
