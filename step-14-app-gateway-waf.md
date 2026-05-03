# Step 14 — Application Gateway + WAF for operators

_The "front door of Rosetta" lab._ 🚪 Builds operator-level fluency in Application Gateway and WAF: backend health, certificate management, blocked-request review, and how to triage "Rosetta is down" tickets without write access.

> [!NOTE]
> **Trainee duration:** 120 minutes
> **Instructor EDE:** 4.0 hours (1h prep + 2h delivery + 1h Q&A buffer)
> **Lab cost:** under NZD $5 — a small AGW v2 + WAF for an hour. **Delete promptly after.**
> **Prerequisites:** Steps 01–04, 12 complete.
> **Pairs with:** Module 3 of the DIA training plan (Application Operations).

---

## 📖 Session overview

Application Gateway is the most-watched component in DSR after Rosetta itself. It's the first place you look when end users report problems, and the second place you look when WAF starts blocking legitimate traffic. This lab deploys a tiny AGW + WAF combo, walks the listener/rule/backend wiring, exercises blocked-request review, and explains certificate rotation — the read-only operational view you'll use on production.

**What you'll learn**
- How a request flows through AGW: **Listener → Rule → Backend pool → HTTP setting → backend VM**.
- How **probes** decide whether a backend is healthy.
- How **WAF** rules work (managed rule set vs custom rules) and how to read **blocked-request** logs.
- How **certificates** attach to listeners (Key Vault integration) and how rotation works.
- The two AGW SKUs (Standard_v2, WAF_v2) and which DSR uses.
- How to triage "Rosetta is down" through AGW alone.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **AGW** | Application Gateway. L7 reverse proxy with WAF, SSL termination, URL routing. |
| **Listener** | A combo of frontend IP + port + protocol + hostname that AGW listens on. |
| **Backend pool** | A list of backend targets (VMs, VMSS, FQDN, App Service) that AGW forwards to. |
| **HTTP setting** | The backend-side protocol/port/probe configuration. |
| **Rule** | Wires listener → URL path map → backend pool + HTTP setting. |
| **WAF** | Web Application Firewall. OWASP-style rules + custom rules. |
| **Managed rule set (CRS)** | Microsoft-curated OWASP rules (CRS 3.1, 3.2, etc.). |
| **Detection vs Prevention** | WAF mode. Detection logs only; Prevention logs and blocks. |
| **Block list / Allow list** | Custom WAF rules. Block specific IPs, allow exceptions. |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Application Gateway introduction](https://learn.microsoft.com/training/modules/intro-to-azure-application-gateway/) | The component's own mental model. |
| [Web Application Firewall on Application Gateway](https://learn.microsoft.com/training/modules/introduction-azure-web-application-firewall/) | The protection in front of Rosetta. |
| [Application Gateway diagnostic logs](https://learn.microsoft.com/azure/application-gateway/application-gateway-diagnostics) | The exact fields in the logs you'll query. |

About **2 hours** of optional pre-reading.

## 🧱 Foundational primer

- **AGW v2 is auto-scaling**, region-redundant. DSR uses AGW v2 with WAF.
- **One listener per host:port combination.** TLS listener requires a certificate (uploaded directly or referenced from Key Vault).
- **Probes are lightweight HTTP/HTTPS GETs** at a path you specify (`/healthz` or similar). If 5 consecutive probes fail, the backend is marked unhealthy.
- **WAF in Prevention mode logs and blocks; Detection mode logs only.** DSR runs Prevention.
- **WAF rules can have false positives.** Rosetta admin paths sometimes look like attacks; you may need to tune exclusions.
- **Certificates are stored in Key Vault** with Managed Identity access from AGW. Rotation = update Key Vault; AGW picks up the new version.
- **AGW logs are the audit trail.** Access log = every request; Performance log = AGW health; Firewall log = WAF actions.

## ⌨️ Activity 1 — Deploy a tiny AGW

1. Portal → **Application gateways → + Create**.
2. RG: `rg-labs-foundations-<your-initials>`. Region: Australia East.
3. Tier: **WAF V2**. Autoscale: minimum 1, max 2.
4. Virtual network: `vnet-labs-net` (from Step 05). Subnet: needs to be its own — create one called `snet-agw` if it doesn't exist (CIDR `10.99.3.0/24`).
5. Frontend → Public IP. Create new.
6. Backend → For now, "Don't add a target" — we'll add a public FQDN as a test.
7. Configuration → Listener: HTTP:80 (no cert needed for the lab). HTTP setting: HTTP:80.
8. Rule: connect listener → backend.
9. **WAF policy**: create new, OWASP 3.2, Detection mode.
10. Review + create. Wait ~5 min.

> [!IMPORTANT]
> AGW costs ~NZD $0.30/hour while running. **Set a calendar reminder to delete after the lab — $7+/day if forgotten.**

## ⌨️ Activity 2 — Configure a backend

1. AGW → **Backend pools → click your default pool → Edit**.
2. Add target: FQDN, value `httpbin.org` (a public test endpoint).
3. Save.
4. Wait ~30 seconds. **Backend health** blade → status should be Healthy (probe is `/`).

## ⌨️ Activity 3 — Hit it

1. AGW → Overview → copy the public IP.
2. Browser: `http://<public IP>/`. You should see httpbin's homepage.
3. Refresh a few times. Each refresh is a request through AGW.

## ⌨️ Activity 4 — Read access logs

1. **Diagnostic settings (under Monitoring) → + Add**.
2. Send to Log Analytics workspace (your trial workspace). Categories: ApplicationGatewayAccessLog, FirewallLog, PerformanceLog.
3. Wait ~5 min for logs to start arriving.
4. Log Analytics → run:

```kql
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS"
| where Category == "ApplicationGatewayAccessLog"
| project TimeGenerated, clientIP_s, requestUri_s,
          httpStatus_d, responseLatency_d, userAgent_s
| order by TimeGenerated desc
| take 50
```

You see every request you've made.

## ⌨️ Activity 5 — Trigger a WAF block

1. Browser: `http://<public IP>/?id=' OR 1=1--` (a classic SQL injection pattern).
2. With WAF in **Detection mode**, the request is *logged* but not blocked.
3. Switch WAF to **Prevention** mode (WAF policy → Policy settings → Prevention → Save).
4. Re-fire the URL. You should get **403 Forbidden** from AGW.
5. Query the firewall log:

```kql
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS"
| where Category == "ApplicationGatewayFirewallLog"
| project TimeGenerated, clientIp_s, requestUri_s,
          ruleId_s, action_s, message_s
| order by TimeGenerated desc
| take 20
```

You'll see the rule ID that fired (e.g. `942100` — SQL injection detection) and `action_s = Blocked`.

## ⌨️ Activity 6 — Add a custom WAF rule

1. WAF policy → **Custom rules → + Add custom rule**.
2. Name: `BlockMyTestIP`. Priority: 1. Action: Block.
3. Match condition: RemoteAddr matches your IP (find your IP at `whatismyip.com`).
4. Save. Wait ~30 sec.
5. Try to reload `http://<public IP>/`. **403** even on the homepage. Now you can no longer reach the AGW from your IP.
6. Disable the custom rule to restore access.

This is the operational pattern for "block this scraper" tickets.

## ⌨️ Activity 7 — Certificate rotation pattern (read-only)

DSR uses Key Vault-stored certificates with AGW. The pattern:

1. Cert is uploaded to Key Vault by the Cloud Networking team.
2. AGW listener references the cert by Key Vault URI.
3. AGW's Managed Identity has Key Vault Secrets User role on the vault.
4. To rotate: upload a new version of the cert to Key Vault. AGW picks it up within 4 hours.

For the lab, walk through:
1. Open the AGW listener → see the certificate field.
2. (Don't change it — TLS isn't enabled in this lab.)
3. Read the AGW's Managed Identity → confirm what role assignments it has.

## ⌨️ Activity 8 — Triage "Rosetta is down" — the read

When users say Rosetta is down, walk this checklist:

| Step | Where | What to look for |
|---|---|---|
| 1. AGW frontend reachable? | Browser test | DNS resolves; AGW responds with at least 502/503. |
| 2. Backend health | AGW → Backend health | Pool entries Healthy / Unhealthy. |
| 3. Recent firewall blocks? | KQL on FirewallLog | Sudden spike in `Blocked`. |
| 4. Backend latency? | KQL on AccessLog | `responseLatency_d` distribution. |
| 5. Backend VM heartbeat? | KQL on Heartbeat | Computer name matches Rosetta VMs. |
| 6. Storage / Oracle reachable? | Storage / DB metrics | Check next steps in Step 15. |

You don't fix anything — you escalate. But you escalate with the right data.

## 🦾 Now your turn!

1. Use AGW's **Backend health probe** to add a probe with a custom path (`/status`). Save and observe.
2. Author a custom WAF rule that *allows* a specific IP that would otherwise be blocked. (Higher priority than the default rules.)
3. Find the AGW's autoscale metrics — instance count, throughput. Where do they live?
4. Write the KQL for "show me the top 10 most blocked rule IDs in the last 7 days".

## ✅ Success checklist

- [ ] You've deployed and used a small AGW.
- [ ] You can read access, performance, and firewall logs in KQL.
- [ ] You've triggered a WAF block and identified the rule ID.
- [ ] You can write a custom WAF rule (block + allow).
- [ ] You can describe how AGW certificates are stored and rotated via Key Vault.
- [ ] You can run the "Rosetta is down" triage checklist end-to-end.
- [ ] You've **deleted** the AGW.

## 📚 Self-serve refresher

- [AGW backend health troubleshooting](https://learn.microsoft.com/azure/application-gateway/application-gateway-backend-health-troubleshooting) — when probes go red.
- [WAF managed rule set reference](https://learn.microsoft.com/azure/web-application-firewall/ag/application-gateway-crs-rulegroups-rules) — every rule explained.

## 💰 Cost note

- AGW v2 + WAF: ~NZD $0.30/hour fixed + small per-GB data fee.
- **Cleanup is essential.** ~$220/month if left running.

```bash
# Cleanup
az network application-gateway delete -g rg-labs-foundations-<your-initials> -n <your AGW name>
az network public-ip delete -g rg-labs-foundations-<your-initials> -n <your AGW public IP>
```

---

⬅️ **Previous:** [Step 13 — Rosetta architecture walkthrough](step-13-rosetta-architecture.md)
➡️ **Next:** [Step 15 — Oracle on Azure (read-only ops)](step-15-oracle-on-azure.md)
