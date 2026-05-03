# Step 24 — Capstone: Incident triage tabletop

_The "did everything you learned actually stick?" lab._ 🏆 The final Capstone — four scenarios run as a tabletop exercise. No new content, just the ability to navigate the tools you've built across all 22 prior steps under realistic time pressure.

> [!NOTE]
> **Trainee duration:** 180 minutes (four scenarios × ~45 min each)
> **Instructor EDE:** 6.0 hours (1.5h prep + 3h delivery + 1.5h Q&A buffer — debrief time matters)
> **Lab cost:** $0 — read-only across the existing estate.
> **Prerequisites:** Steps 01–22 complete. Confidence in KQL, the Workbooks, and Backup Center.
> **Pairs with:** Module 3 + Module 4 of the DIA training plan (Application Operations + Observability). **Closes the programme.**

---

## 📖 Session overview

You've spent 22 steps absorbing content. This Capstone tests whether it sticks under pressure. Four scenarios, run at the whiteboard with the actual Azure portals open, simulated as real incidents. Each scenario:

1. The **trigger** (a Teams ping, a paged alert, a user complaint).
2. **Triage** — first 10 minutes of read-only investigation.
3. **Communication** — the Teams update or stakeholder email.
4. **Escalation** — who you hand off to and what context you include.
5. **Debrief** — what would you do differently next time.

The goal is fluency, not completeness. You won't fix anything; you'll know exactly what to read and who to call.

**What you'll exercise**
- Navigating the **Weekly Health Workbook** under stress.
- Writing **clean escalation messages** to DBA, Cloud Platform, Cloud Security.
- **Recognising patterns** — index regression vs. AGW vs. storage vs. backup.
- **Reporting up** — short status messages to leadership during an incident.
- **Post-incident notes** that feed the runbook.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **Tabletop** | A walkthrough exercise, not a real incident. Live tools, simulated trigger. |
| **Triage** | The first read of the situation. Goal: confirm scope and direct to the right responder. |
| **Bridge** | A Teams call set up for the incident. Multiple owners join. |
| **Comms cadence** | How often you update stakeholders. Default: every 30 min during a Sev1, hourly during a Sev2. |
| **PIR** | Post-Incident Review. Comes after the fix, separate meeting. |

## 📚 Prepare in advance

- Have the **Weekly Health Workbook** (Step 22) open in one tab.
- Have **Backup Center** (Step 20) open in another.
- Have **Defender for Cloud → Alerts** (Step 21) open in a third.
- Have the **DSR Application Operations runbook** open in a fourth.
- Whiteboard or shared notes doc for the tabletop facilitator.

## 🧱 Foundational principles

- **Read before you call.** Most "incidents" resolve themselves within five minutes if you have the right tile open.
- **Confirm scope before paging.** "Is this the Index tier or every tier?" matters for who you wake.
- **Communicate often, briefly.** Three short updates beat one long one.
- **Don't fix what isn't yours.** Restraint is a skill. Document, escalate, hand off.
- **Always start a notes doc.** Timeline + actions + owners. Fuels the PIR later.

---

## 🎯 Scenario 1 — Snapshot job failed overnight

> **Trigger (08:55 NZT, Friday):** Backup Center email alert: "ANL — Backup job failed for `vm-rosi-prd-03`."

### What you actually do

1. Open Backup Center → filter to `vm-rosi-prd-03` → read the failed job. Note:
   - Failure error code.
   - Timestamp of last successful backup.
   - Whether the next scheduled run will retry automatically.
2. Open the Weekly Health Workbook → backup compliance tile. Is this an isolated failure or part of a pattern (multiple VMs)?
3. Check the VM itself: heartbeat in the last hour? Disk space? Recent reboot?
4. **Decide:** isolated, retried automatically, low-impact → log + observe. Pattern across multiple VMs → page Cloud Platform.

### Communication

In your Team's "DSR Ops" Teams channel:
> 🔵 ANL Ops: `vm-rosi-prd-03` overnight backup failed (`<error code>`). Last successful: `<timestamp>`. Auto-retry next cycle. Monitoring.

### Escalation criteria

Page Cloud Platform if any of: same VM fails twice in a row; 3+ VMs fail same night; soft delete on the vault is somehow disabled.

### Debrief discussion

- What's the runbook entry for this exact error code?
- Should this become a Workbook tile threshold so it pages instead of emails?

---

## 🎯 Scenario 2 — AGW backend goes red

> **Trigger (14:12 NZT, Tuesday):** User report — "Rosetta UI is throwing 502 Bad Gateway."

### What you actually do

1. Open AGW resource → **Backend health**. Identify which backend pool is unhealthy.
2. Workbook tile: AGW unhealthy backends over time. Is this just now, or trending?
3. KQL: `AzureDiagnostics | where ResourceType == "APPLICATIONGATEWAYS" | where Category == "ApplicationGatewayAccessLog" | where TimeGenerated > ago(30m) | summarize count() by backendStatusCode_d`. Confirm scale of the 502 wave.
4. Check the affected VMs: heartbeat? CPU? Disk? Recent restart?
5. NSG / route changes in last 24h (Activity Log)?
6. **Decide:** AGW config issue → Cloud Networking. Backend VM issue → Cloud Platform + you read VM logs. App-tier issue → app team but you raise.

### Communication

```
Subject: ANL — Rosetta 502s — investigating

Status: investigating
Started: 14:12 NZT
Scope: ~80% of UI requests returning 502 from AGW; backend pool "rosetta-app-pool" 2/3 unhealthy.
What we know: VMs heartbeating but probe failing on /healthz.
Next step: VM-level checks; will update by 14:30.
```

### Escalation criteria

VMs are down → Cloud Platform. Probe path returns 5xx but VM seems alive → application-tier owner. Pattern across multiple AGW listeners → Cloud Networking.

### Debrief discussion

- Which Workbook tile would have caught this 5 minutes earlier?
- Is the probe path documented somewhere a non-app-team operator can find?

---

## 🎯 Scenario 3 — Cold-tier retrieval cost spike

> **Trigger (Monday 09:00):** Cost Management anomaly alert — "ANL daily cost up 43% on Sunday."

### What you actually do

1. Cost Management → daily breakdown for Sunday. What service?
2. Likely culprit: **Storage → Read transactions and/or Cold/Archive retrieval**.
3. Open the storage account in question → Insights → transactions by tier.
4. KQL: `StorageBlobLogs | where TimeGenerated >= datetime(<sunday>) | where StatusCode < 300 | summarize MB = sum(ResponseBodySize)/1024/1024 by Uri | top 20 by MB desc`. Identify the URIs.
5. Match URIs to a blob tier (Step 12 inventory join). If they were Cold/Archive, that explains the spike.
6. Talk to the application owner / curator: was a planned bulk export run? If yes, expected. If no, investigate.

### Communication

```
ANL — Cost spike Sunday (≈+43% / +NZD $X)
Cause: Cold-tier blob retrieval volume on `stanlnznblobprdwod01`.
Top URIs: <list 5>.
Next: confirming with the curator team whether this was a planned export.
```

### Escalation criteria

Curator confirms unplanned → Cloud Security investigation (Defender alerts? Anomalous access?). Curator confirms planned → close the ticket as expected; document for the Monthly Cost Report commentary.

### Debrief discussion

- Should bulk-export jobs be announced in the Ops channel ahead of time?
- Could lifecycle rules (Step 07) have moved this content out of Archive earlier so retrieval was cheaper?

---

## 🎯 Scenario 4 — Hold removal request, with a twist

> **Trigger (Wednesday 10:30):** Email from Records Team — "Please remove the legal hold on container `<x>` so we can run lifecycle on it."

### What you actually do

1. Open the container → Properties → Legal hold tags.
2. Confirm what hold tag is in place, and whether the **time-based retention policy** is also set.
3. Cross-reference the **Records Team request** with the **Information Management policy** — does the request meet the documented criteria?
4. Recognise that **legal hold removal is a write action** outside Preservation Team scope — you don't action it directly.
5. Confirm the request is from an authorised role (Records Team Lead or General Counsel delegate, per the policy).
6. Forward to the operator who owns hold management (DSR Cloud Platform with Information Management approval) with full context.

### Communication

```
Subject: ANL — Legal hold removal request — <container>

Originator: <name, role>
Request received: <timestamp>
Container: <stanlnznblobprdwod01:archived-2018>
Hold tag(s): <list>
Time-based retention: <set / not set / locked / unlocked>
Approval verification: <pending / received / cited>

Recommendation: ready for action by Cloud Platform once Information Management sign-off attached.
```

### Escalation criteria

Approval is missing → return to originator. Container also has a **Locked** time-based retention policy → policy change is a separate, slower path; flag this in your reply.

### Debrief discussion

- Where is the Information Management policy documented?
- Should the originator have used the formal change request process?

---

## ⌨️ Activity — Run the four scenarios end-to-end

Facilitator (instructor) plays the trigger; trainees take turns leading. Aim ~45 min per scenario:

- 5 min — read the trigger, ask clarifying questions.
- 15 min — triage with the live tools.
- 5 min — draft the comms message.
- 5 min — define the escalation.
- 15 min — debrief with the room.

Alternate the trainee leading each scenario to spread the practice.

## 🦾 Now your turn!

1. Pick a fifth scenario from the DSR runbook (or invent one) and run it solo.
2. Write a short post-incident note for one of the scenarios above. What's missing from the runbook? Submit a PR.
3. Time yourself on Scenario 2 with no notes. Goal: <10 min from trigger to comms.
4. Build a one-pager **incident response cheat sheet** — top six "open this tab first" actions, taped to your desk.

## ✅ Success checklist

- [ ] You can run all four scenarios in <45 min each.
- [ ] You can write a clean comms message under 5 min.
- [ ] You can identify the right escalation owner without checking the runbook.
- [ ] You've added at least one improvement back into the runbook.
- [ ] You can pass the programme: name every step, every owner, every Workbook, every saved KQL function.

## 📚 Self-serve refresher

- DSR Application Operations runbook, Section 7 ("Incident Response").
- [Microsoft incident response framework](https://learn.microsoft.com/security/operations/incident-response-overview) — for the broader structure.

## 🎓 Programme close

You've completed the 23-step Foundation programme. From here:

- Pair with a teammate on a real production incident (shadow first, lead second).
- Pick up one open runbook gap and fill it.
- Help train the next cohort — teaching what you've learned solidifies it.
- Keep your **personal sandbox subscription** active for trying new Azure features safely.

Ka pai.

---

⬅️ **Previous:** [Step 23 — Capstone: Monthly Cost Report](step-23-capstone-monthly-cost.md)
➡️ **Next:** _Programme complete._ Move on to advanced specialisation (your choice — security, FinOps, automation, application engineering)._
