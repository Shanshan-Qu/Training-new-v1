# Step 20 — Defender for Storage (awareness)

_The "what does Cloud Security expect from us?" lab._ 🛡️ Awareness-only walkthrough of Defender for Storage: what it watches, the alerts it raises, where they appear, and the escalation path. Phase 4 closes here.

> [!NOTE]
> **Trainee duration:** 60 minutes
> **Instructor EDE:** 3.0 hours (1h prep + 1h delivery + 1h Q&A buffer)
> **Lab cost:** $0 — read-only walkthrough.
> **Prerequisites:** Steps 01–05 complete. Defender already enabled at sub level (DSR Cloud Security owns this).
> **Pairs with:** Module 4 of the DIA training plan (Observability & Reporting). **Ownership:** DSR Cloud Security owns Defender configuration and alert investigation. The Preservation Team's role is *recognise → escalate*.

---

## 📖 Session overview

Defender for Storage is the threat-detection plane sitting on top of Azure Storage. It analyses access patterns and signals (anomalous IPs, mass deletes, suspicious downloads) and raises alerts when something looks off. DSR has it enabled across all storage accounts. Your team is **not** the investigator — Cloud Security owns that — but you'll be the first to see an alert in your Workbook or to receive a "what's this access pattern on stanlnznblobprdwod01?" question. This lab makes you fluent enough to triage and escalate, no more.

**What you'll learn**
- What Defender for Storage **does and does not** detect.
- The alert types: **suspicious access, malware, sensitive data exposure**.
- Where alerts appear: **Defender for Cloud → Alerts**, plus the workspace.
- The escalation pattern to Cloud Security.
- Awareness of **Defender for Cloud Apps** (separate product, often confused).
- The cost model — Defender for Storage is per-storage-account.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **Defender for Cloud** | The umbrella security posture + threat-protection product. |
| **Defender for Storage** | The plan that protects storage accounts specifically. |
| **MDC** | Microsoft Defender for Cloud (acronym you'll see in tickets). |
| **Alert** | A scored event — Low, Medium, High severity. |
| **Recommendation** | A best-practice posture finding (e.g. "soft delete not enabled"). |
| **Threat protection** | The detection plane (alerts). Posture is the configuration plane (recommendations). |
| **Hash reputation** | Defender's malware-scan signature lookup at upload time. |
| **MAR** | Malware scanning. |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Plan Defender for Storage](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-storage-introduction) | The product overview. |
| [Storage threat alerts reference](https://learn.microsoft.com/azure/defender-for-cloud/alerts-reference#alerts-azurestorage) | Every alert type explained. |
| [Respond to alerts](https://learn.microsoft.com/azure/defender-for-cloud/managing-and-responding-alerts) | The default workflow Cloud Security uses. |

About **1 hour** of optional pre-reading.

## 🧱 Foundational primer

- **DSR has Defender for Storage at the *Activity Monitoring* tier on all production accounts.** Malware scanning is not enabled by default (extra cost) — confirm per account in DSR.
- **Defender does NOT inspect blob *content*** unless the malware-scanning add-on is enabled. By default it analyses control-plane and data-plane logs only.
- **Alerts surface in two places**: Defender for Cloud → Alerts, and the workspace's `SecurityAlert` table.
- **You will not action alerts.** Cloud Security has a 24/7 SOC. You forward + provide context.
- **The recommendations side is also valuable.** Posture findings show "your storage account isn't following best practice" (no public access, no SAS-with-no-expiry, etc.). DSR addresses these via Policy.
- **Defender raises false positives.** A penetration test or a legitimate bulk-download by an archivist will look anomalous. Context from the application owner (you) is what closes the alert correctly.

## ⌨️ Activity 1 — Confirm Defender is enabled

1. Subscription → **Defender for Cloud → Environment settings → click your sub**.
2. Storage row → ON. Plan: **Activity Monitoring** (or Activity + Malware).
3. Note the per-account override option — but DSR runs uniform.

## ⌨️ Activity 2 — Read the alerts blade

1. Defender for Cloud → **Alerts**. Filter by Resource type = Storage.
2. Live alerts (if any) show severity, alert title, affected resource.
3. Click into one (real or simulated) → see Description, Affected resources, Take action tab, Investigation graph.

If your sub has no alerts, the page is empty — that's the normal state for a healthy sub.

## ⌨️ Activity 3 — Generate a sample alert

Defender supports a built-in sample alert generator.

1. Subscription → **Defender for Cloud → Workload protections → Storage account threat protection → Sample alerts**.
2. Or via CLI:

```bash
az security alert simulate -n "Sample-Storage-DataExfiltration" \
  --location australiaeast
```

3. Wait ~2 min. Refresh the Alerts blade. You see a simulated "Suspicious data extraction" alert.

## ⌨️ Activity 4 — Read the alert details

Sample alert → click in.

- **Description:** what triggered + recommended action.
- **Take action:** suggested mitigation (e.g. revoke SAS, block IP).
- **Investigation graph:** the entities involved (account, IP, identity).
- **Tags:** `MITRE ATT&CK` mapping (e.g. T1530 — data from cloud storage object).

## ⌨️ Activity 5 — Find alerts in KQL

```kql
SecurityAlert
| where TimeGenerated > ago(7d)
| where ResourceType has "storage"
| project TimeGenerated, AlertName, AlertSeverity, ResourceId,
          Status, ProviderName, ExtendedProperties
| order by TimeGenerated desc
```

Pin this query as a tile on the **ANL Health** Workbook from Step 17. Empty result = healthy day; any rows = read and act.

## ⌨️ Activity 6 — Recognise vs. investigate

| Symptom | Your action |
|---|---|
| Sample alert in test sub | Dismiss after walkthrough. |
| `Suspicious sign-in to a storage account` on prd | Forward to Cloud Security with: account name, time, calling IP, your context (was a curator running bulk export?). |
| `Anomalous access from an exit node of an active TOR network` | Page Cloud Security immediately. Do not action yourself. |
| `Phishing content hosted on a storage account` | Page Cloud Security; pull AccessTier of affected blobs. |
| `Public access of a storage account` recommendation | Cloud Security action; you can provide whether your container needed public access (it shouldn't). |

The pattern: **provide context; do not remediate**.

## ⌨️ Activity 7 — Escalation message template

```
Subject: ANL — Defender alert <severity> on <storage account>

Alert ID: <Defender alert ID>
Severity: <Low/Medium/High>
First seen: <UTC timestamp>
Account: <stanlnznblobprdwod01>
Container(s) impacted: <if known>

Application context:
  - Account purpose: <e.g. Web On-Demand WARC archive>
  - Owner: <Preservation Team>
  - Recent legitimate bulk-access events: <e.g. "Tuesday's archivist export of 2024 corpus" / "none">
  - Known maintenance windows: <if any>

Recommendation: please investigate as <category, e.g. "data exfiltration"> per the Defender alert details.
```

Provide this in the ServiceNow / DSR Sec ticket. The clearer your context, the faster Cloud Security closes the alert correctly.

## ⌨️ Activity 8 — Posture recommendations (read-only)

1. Defender for Cloud → **Recommendations**. Filter Resource type = Storage.
2. Common findings:
   - "Storage account public access should be disallowed."
   - "Storage account should restrict network access using virtual network rules."
   - "Storage account should require Secure Transfer."
3. For each, look at affected resources. Are any in your scope? Confirm yes/no with the actual storage account view.

This is informational — Cloud Security drives remediation via Policy or direct config.

## 🦾 Now your turn!

1. Find the Microsoft Defender for Cloud workbook gallery — which one is built for storage threat protection?
2. Pin a `SecurityAlert` tile to your **ANL Health** Workbook.
3. Read the DSR Cloud Security playbook section on storage alert response — what's the SLA for High-severity alerts? Where do you escalate after hours?
4. Find the `Microsoft.Security/locations/alerts` REST endpoint — could you write an automation that posts new alerts to a Teams channel? (You'd ask Cloud Security before doing it.)

## ✅ Success checklist

- [ ] You can describe what Defender for Storage detects (and what it doesn't).
- [ ] You can find an alert in Defender for Cloud and in `SecurityAlert` via KQL.
- [ ] You've simulated a sample alert and read it end-to-end.
- [ ] You can write a clean escalation context message.
- [ ] You know who owns alert investigation (Cloud Security) and who provides context (you).

## 📚 Self-serve refresher

- [Defender for Storage alerts reference](https://learn.microsoft.com/azure/defender-for-cloud/alerts-reference#alerts-azurestorage) — every alert.
- [MITRE ATT&CK for cloud](https://attack.mitre.org/matrices/enterprise/cloud/) — the framework alerts map to.

## 💰 Cost note

- Defender for Storage Activity Monitoring: ~NZD $0.05 per 100K transactions on the protected account. Big accounts add up.
- Malware scanning add-on: separate per-GB charge. DSR does not enable on archive accounts; weigh per account.
- This lab: $0 (sample alerts are free).

---

⬅️ **Previous:** [Step 19 — Backup Center read-only operations](step-19-backup-center.md)
➡️ **Next:** [Step 21 — Capstone: Weekly Health Report](step-21-capstone-weekly-health.md) (Phase 5 begins)
