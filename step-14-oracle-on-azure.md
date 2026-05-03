# Step 14 — Oracle on Azure (read-only ops view)

_The "I can recognise an Oracle problem when I see one" lab._ 🐘 Builds operational awareness of Rosetta's Oracle backend on Azure VMs: Data Guard sync, RMAN backup status, port 1521 connectivity checks — without ever issuing a SQL command.

> [!NOTE]
> **Trainee duration:** 75 minutes
> **Instructor EDE:** 3.25 hours (1h prep + 1.25h delivery + 1h Q&A buffer)
> **Lab cost:** $0 — all reading; no Oracle deploy.
> **Prerequisites:** Steps 01–04, 12 complete.
> **Pairs with:** Module 3 of the DIA training plan (Application Operations). **Note:** Oracle administration (SQL, schema, RMAN config) is owned by **DIA Database Team**. The Preservation Team owns *connectivity awareness* and recognises the symptoms of Oracle-related Rosetta problems.

---

## 📖 Session overview

Rosetta uses Oracle for catalogue, technical metadata, and workflow state. In DSR, Oracle runs as VMs with Data Guard for HA and RMAN for backup. Your team won't run SQL or restore databases — that's the DBAs — but you *will* be the first to see "Rosetta says deposit is stuck" when Oracle replication lags or a connection pool exhausts. This lab makes you fluent enough to triage, escalate accurately, and read the right metric/log.

**What you'll learn**
- The Oracle-on-Azure pattern: **Primary VM + Standby VM with Data Guard async sync**.
- How **port 1521** connectivity works between Rosetta and Oracle (NSG rule, private endpoint or direct VM IP).
- Where **RMAN backup status** is logged and how to read "did last night's backup succeed?".
- How to **test connectivity from a Rosetta VM** without SQL access (TCP test, DNS check).
- The signals that point to an Oracle problem (vs. an app problem).

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **Oracle DB** | The relational database engine. Rosetta uses it for catalogue + workflow. |
| **Data Guard** | Oracle's HA / DR replication. Primary writes; Standby applies redo logs. |
| **Standby database** | The replica. Read-only or "physical standby" in DSR. |
| **RMAN** | Recovery Manager — Oracle's backup tool. Writes to a backup destination (often Azure Files or Blob). |
| **TNS Listener** | Oracle service that listens for incoming connections. Default port: 1521. |
| **TNS connect string** | The client-side address Rosetta uses (e.g. `(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=ora.prd.anl)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=ROSPRD)))`). |
| **Wallet / TDE** | Encryption-at-rest for the database. Wallet is the credential store. |
| **DBA / DBE** | Database Administrator / Database Engineer (DIA's database team). |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Oracle workloads on Azure overview](https://learn.microsoft.com/azure/virtual-machines/workloads/oracle/oracle-overview) | The reference architecture DSR uses. |
| [Architecture: Oracle on Azure VMs with Data Guard](https://learn.microsoft.com/azure/virtual-machines/workloads/oracle/oracle-reference-architecture) | The exact shape of the DSR Oracle estate. |
| [Backing up Oracle databases on Azure](https://learn.microsoft.com/azure/virtual-machines/workloads/oracle/oracle-database-backup-azure-storage) | RMAN to Azure Files / Blob — what DSR uses. |

About **1.5 hours** of optional pre-reading.

## 🧱 Foundational primer

- **DSR runs Oracle on Linux VMs**, not Oracle Database Service for Azure (different product, separate consideration).
- **Two-node Data Guard**: primary in one zone, standby in another (within NZ North).
- **Async replication** — small lag is normal (seconds). Larger lag (minutes) is a problem worth escalating.
- **Port 1521** is the only port Rosetta uses to reach Oracle. NSG rule allows app subnet → oracle subnet on 1521 only.
- **RMAN backups** run nightly to a dedicated Azure Files share or Blob container, retained per the data retention policy.
- **You read; DBAs write.** Your team gets Reader on the Oracle VMs and the backup share. You can see metrics and logs but not SQL data.
- **The DBA team's runbook owns failover.** You raise the ticket; they execute.

## ⌨️ Activity 1 — Map the Oracle resources

```kql
resources
| where tags.app_name == "anl"
| where type == "microsoft.compute/virtualmachines"
| where name has "ora" or tags.tier == "database"
| project name, location, tier = tags.tier, env = tags.environment, resourceGroup
```

Should return the primary and standby Oracle VMs (e.g. `vm-ora-prd-01`, `vm-ora-prd-02`).

## ⌨️ Activity 2 — Test connectivity to Oracle on port 1521 (read-only)

From any Rosetta VM (or your Cloud Shell with VNet integration):

```bash
# Resolve Oracle FQDN
nslookup ora.prd.anl.dia.govt.nz

# TCP test on port 1521
nc -vz ora.prd.anl.dia.govt.nz 1521
# expected: succeeded
```

If `nc` fails, the issue is one of:
1. NSG (Step 04 rule blocking app → oracle on 1521).
2. Oracle TNS Listener not running.
3. DNS pointing to a stale IP.

This three-bullet checklist is your initial triage when "Rosetta can't reach DB".

## ⌨️ Activity 3 — Read RMAN backup status (log only)

DSR pipes RMAN logs to Log Analytics via Azure Monitor Agent. Sample query:

```kql
Syslog
| where Computer startswith "vm-ora-prd"
| where Facility == "user" and ProcessName == "rman"
| where TimeGenerated > ago(36h)
| project TimeGenerated, Computer, SeverityLevel, SyslogMessage
| order by TimeGenerated desc
```

What "good" looks like — a successful run shows lines like `RMAN-08023: backed up 1 file(s)` followed by `RMAN-00569: list of objects backed up` and exit code 0.

What "bad" looks like — an `ORA-19xxx` error line, or an early exit with no completion stamp.

## ⌨️ Activity 4 — Read Data Guard sync lag (metric)

Oracle exports a metric for redo apply lag via the OEM agent or a custom log:

```kql
// Custom log table fed by a small script on the standby VM
Oracle_DG_Lag_CL
| where Computer == "vm-ora-prd-02"
| where TimeGenerated > ago(1h)
| project TimeGenerated, lagSeconds_d
| render timechart
```

Acceptable: <10s. Concern: 10–60s for >5 minutes. Escalate: >60s sustained.

(If your environment doesn't expose this yet, the DBA team owns the script. Recognise that the metric exists.)

## ⌨️ Activity 5 — Read VM-level metrics

1. Oracle VM → **Metrics** → Percentage CPU, OS Disk QD, Network In/Out.
2. Spike on disk queue depth often correlates with RMAN runs (expected) or runaway query (escalate to DBA).
3. CPU > 80% sustained on an Oracle VM is rare and usually a query-plan regression.

## ⌨️ Activity 6 — Recognise an Oracle problem vs an app problem

| Symptom | Likely cause |
|---|---|
| Rosetta UI loads but search returns nothing | Index tier (Step 12), not Oracle. |
| Rosetta UI shows "Database connection failed" banner | Oracle / connectivity. Run Activity 2. |
| Deposits stuck in "ingesting" | Could be either — check Workflow tier logs first. |
| Sudden flood of "ORA-12541" in app logs | TNS Listener down on Oracle host — page DBA. |
| Sudden flood of "ORA-01017" | Auth/credentials — likely a wallet problem; DBA. |
| Sudden flood of "ORA-00060" | Deadlock — DBA. |

Memorise the top 4. The rest you can look up.

## ⌨️ Activity 7 — Escalation pattern

When you escalate to DBA, include:

1. **What** — the symptom and screenshot/log line.
2. **When** — first observed timestamp (use UTC).
3. **Where** — primary or standby, environment.
4. **What you've checked** — connectivity (Activity 2), VM metrics (Activity 5), recent app changes.
5. **Impact** — number of users affected, business consequence.

A clear escalation gets a fast response. A vague "Oracle's broken" wastes a round trip.

## 🦾 Now your turn!

1. Write the KQL for "show every RMAN backup outcome in the last 7 days, with success/fail count by day".
2. Find the **Backup Center** entry for the Oracle backup vault (covered fully in Step 19) — does it list the Oracle VMs?
3. From a Rosetta VM, write a one-liner that confirms DNS, port 1521 reachable, *and* logs the result with a timestamp.
4. Read the **DSR Oracle Data Guard runbook** (DIA SharePoint) and identify the failover trigger conditions. Don't memorise the steps — just know where to find them.

## ✅ Success checklist

- [ ] You can name the Data Guard role of each Oracle VM in DSR.
- [ ] You can test connectivity to Oracle from a Rosetta VM (DNS + TCP).
- [ ] You can read RMAN logs and tell success from failure.
- [ ] You can recognise the top-4 ORA error codes and what they mean.
- [ ] You know which problems escalate to DBA and which don't.
- [ ] You can write a clean escalation message.

## 📚 Self-serve refresher

- [Common ORA error codes](https://docs.oracle.com/en/error-help/db/) — Oracle's official error reference.
- [Azure Monitor for VMs](https://learn.microsoft.com/azure/azure-monitor/vm/vminsights-overview) — VM-side metrics.

## 💰 Cost note

- This lab: $0 — all read-only.

---

⬅️ **Previous:** [Step 13 — Application Gateway + WAF](step-13-app-gateway-waf.md)
➡️ **Next:** [Step 15 — WOD container operations](step-15-wod-container-ops.md)
