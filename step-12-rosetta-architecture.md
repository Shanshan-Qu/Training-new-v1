# Step 12 — Rosetta architecture walkthrough (read-only)

_The "draw the picture from memory" lab._ 🏛️ Builds the operator's mental model of Rosetta on DSR: Application Gateway, App VMs, Index VMs, Oracle, file shares, blob storage, and how they connect. No deploys — recognising and reading.

> [!NOTE]
> **Trainee duration:** 90 minutes
> **Instructor EDE:** 3.5 hours (1h prep + 1.5h delivery + 1h Q&A buffer)
> **Lab cost:** $0 — all reading, no deploys.
> **Prerequisites:** Steps 01–11 complete.
> **Pairs with:** Module 3 of the DIA training plan (Application Operations) — anchored to the DSR DPS Azure Application Landing Zone Design v1.4.

---

## 📖 Session overview

Phase 3 starts here. You've built foundations and learned the storage estate; now we tour the application that consumes it. Rosetta is Ex Libris' digital preservation platform; in DSR it runs as a tier of Linux VMs behind an Application Gateway, talking to Oracle for metadata and to Azure Files / Blob for content. Your team operates it day-to-day — restarts, log reads, deposit triage, bill-of-health. This lab walks every component end-to-end, read-only.

**What you'll learn**
- The Rosetta tier names and roles: **Webapp / Index / Permanent / Repository / Workflow** (DSR uses these terms — verify your install).
- Where each tier runs (App Gateway → App tier VMs → Index VMs → Oracle).
- The two storage roles: **shared file share** (NFS Premium) for working areas, **blob storage** (Standard ZRS) for permanent IE objects.
- How a deposit flows through the tiers (the SIP → AIP → DIP journey).
- The endpoints, ports, and TLS / authentication between tiers.
- Where each component's logs live and how to read them.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **Rosetta** | Ex Libris' digital preservation system (the application). |
| **Application Gateway (AGW)** | Azure's L7 reverse proxy + WAF in front of the web tier. |
| **WAF** | Web Application Firewall — rules that block SQL injection, XSS, etc. |
| **SIP / AIP / DIP** | Submission / Archival / Dissemination Information Package — OAIS terms used by Rosetta. |
| **Webapp tier** | Rosetta UI + REST API. Public-facing (via AGW). |
| **Permanent / Repository tier** | Long-term IE storage controllers. |
| **Index tier** | Search index for IEs. |
| **Workflow tier** | Background processing (deposits, format characterisation). |
| **VMSS** | Virtual Machine Scale Set — used for tiers that scale horizontally. |
| **NSG** | Network Security Group — already covered in Step 04. |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Azure compute services overview](https://learn.microsoft.com/training/modules/intro-to-azure-compute/) | The VM/VMSS model under Rosetta. |
| [Application Gateway introduction](https://learn.microsoft.com/training/modules/intro-to-azure-application-gateway/) | The front door for Rosetta. |
| [Application Landing Zone (ALZ) reference](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/) | DSR is built on ALZ — you'll see the same patterns. |

Plus: skim the **DSR DPS Azure Application Landing Zone Design v1.4** document. It is the source of truth and you should have a copy bookmarked.

About **2 hours** of optional pre-reading.

## 🧱 Foundational primer

- **AGW is the only public ingress** for Rosetta. WAF is enabled.
- **Each tier has its own subnet**, with NSGs that allow only the tier-to-tier ports needed.
- **Oracle is private** — accessible only from app and workflow tiers via port 1521.
- **NFS file share is shared** between deposit/processing tiers; blob storage is the permanent landing pad.
- **Managed Identity is everywhere.** No SP secrets in Rosetta config; the VMs authenticate to Azure resources via MI.
- **Log Analytics is the central observability bus.** Every VM emits AzureDiagnostics + custom Rosetta logs.

## ⌨️ Activity 1 — Map the components in Resource Graph

```kql
resources
| where tags.app_name == "anl"
| where type in ("microsoft.compute/virtualmachines",
                 "microsoft.compute/virtualmachinescalesets",
                 "microsoft.network/applicationgateways",
                 "microsoft.dbforpostgresql/servers",
                 "microsoft.network/loadbalancers",
                 "microsoft.network/privateendpoints",
                 "microsoft.storage/storageaccounts",
                 "microsoft.keyvault/vaults")
| project name, type, tier = tags.tier, env = tags.environment, location, resourceGroup
| order by type, name
```

Run this against a DSR sub when you have Reader access. It produces the inventory you can match to the architecture diagram.

## ⌨️ Activity 2 — Trace a deposit (mental walkthrough)

Without clicking anything, walk the deposit path:

1. Curator drops a SIP into a deposit folder on the **shared NFS share** (`stanlnznfileprdrosi01:/deposit/`).
2. **Workflow tier** picks it up, validates structure, runs format identification.
3. SIP is converted to AIP and written to the **blob storage account** (`stanlnznblobprdrosi01:/aip/...`).
4. Metadata is written to **Oracle** (object record, technical metadata, fixity).
5. **Index tier** is notified, indexes the new IE.
6. Curator can now find the IE via the **Webapp UI** (HTTPS via AGW).

Keep this picture in your head — every triage call uses one or more steps from it.

## ⌨️ Activity 3 — Read App Gateway configuration

1. Search → AGW resource for ANL → **Listeners**. Note: HTTPS:443, hostname, certificate, frontend IP.
2. **Backend pools** — list of VMs/VMSS that serve traffic.
3. **HTTP settings** — backend port, protocol (typically HTTP within VNet), probe path.
4. **Rules** — listener → URL path map → backend pool wiring.
5. **WAF policy** (if attached to a separate WAF policy resource) — rules, exclusions, custom rules.

> [!TIP]
> When users say "Rosetta is throwing 502 Bad Gateway", AGW is your first stop. Backend health green = problem upstream; red = problem in the backend tier.

## ⌨️ Activity 4 — Read VM and VMSS configuration (read-only)

For one Rosetta VM:

1. VM blade → **Networking** — see NICs, NSGs, IP addresses (private only in DSR).
2. **Disks** — OS disk + data disks. Note disk SKU and size.
3. **Identity** — system-assigned MI; the Object ID is what you'd grant Storage Blob Data Reader to.
4. **Extensions** — Azure Monitor Agent, BackupExtension, etc.
5. **Insights** — performance metrics if Insights is on.

Don't restart, resize, or change anything. This is a *recognise* lab.

## ⌨️ Activity 5 — Read Oracle connectivity

1. Search → Private endpoint for Oracle (PE name pattern `pe-ora-prd-*`).
2. **DNS configuration** — the FQDN should resolve to a private IP in the Oracle subnet.
3. **Connection** — note the target service & resource group.
4. From a Rosetta VM (read-only operationally — usually you'd have a jumpbox), `nslookup ora.prd.anl.dia.govt.nz` returns the PE private IP.

We'll cover Oracle deeper in Step 14.

## ⌨️ Activity 6 — Locate logs

1. Log Analytics workspace for ANL (`law-anl-nzn-prd` or similar) → **Logs**.
2. Sample queries:

```kql
// Heartbeat from every Rosetta VM in last hour
Heartbeat
| where Computer startswith "vm-rosi"
| where TimeGenerated > ago(1h)
| summarize by Computer

// AGW backend health events
AzureDiagnostics
| where Category == "ApplicationGatewayAccessLog"
| where TimeGenerated > ago(1h)
| where backendStatusCode_d >= 500
| summarize count() by backendPoolName_s, bin(TimeGenerated, 5m)

// Custom Rosetta logs (Syslog facility)
Syslog
| where Computer startswith "vm-rosi"
| where Facility == "local0"
| where TimeGenerated > ago(1h)
| project TimeGenerated, Computer, SeverityLevel, SyslogMessage
```

These are the queries you live in during a triage call.

## ⌨️ Activity 7 — Sketch the picture

On paper or whiteboard, draw:

- Hub VNet (with AGW and ExpressRoute Gateway)
- ANL Spoke VNet (PRD)
- Subnets: AGW, App, Index, Workflow, Oracle, PE
- Storage accounts (file + blob)
- Oracle (Private Endpoint)

Compare your sketch to the actual ALZ design v1.4. The gaps in your drawing tell you where to focus next.

## 🦾 Now your turn!

1. Write a Resource Graph query that returns every Rosetta VM in PRD with its OS type, size, and last reboot timestamp.
2. From the AGW resource, find the WAF policy attached to it. What managed rule set version is it on?
3. Write the KQL for "show me every backend in unhealthy state in the last 24 hours".
4. Find Rosetta's Application Insights resource (if any) and identify its instrumentation key path. (DSR uses Log Analytics directly more than App Insights, but this is good to recognise.)

## ✅ Success checklist

- [ ] You can name the Rosetta tiers and what each does.
- [ ] You can sketch the network topology including Hub-Spoke and Private Endpoints.
- [ ] You can read AGW listener/backend/rule config and explain it.
- [ ] You can find the logs for a specific VM in Log Analytics.
- [ ] You know which storage account is for working files vs permanent IEs.
- [ ] You know not to change anything in production — this is a read-only operational role.

## 📚 Self-serve refresher

- DSR DPS Azure Application Landing Zone Design v1.4 (DIA SharePoint).
- [Application Gateway diagnostic logs](https://learn.microsoft.com/azure/application-gateway/application-gateway-diagnostics) — what's in each log category.

## 💰 Cost note

- This lab: $0. All read-only.

---

⬅️ **Previous:** [Step 11 — Blob inventory](step-11-blob-inventory.md)
➡️ **Next:** [Step 13 — Application Gateway + WAF for operators](step-13-app-gateway-waf.md)
