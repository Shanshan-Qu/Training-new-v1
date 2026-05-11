# DSR Training — Digital Preservation Team

Self-paced and instructor-led Azure training for the **Archives Library Digital Preservation Team** at the Department of Internal Affairs (DIA), New Zealand.

The curriculum is grounded in the **DIA DSR DPS Azure Application Landing Zone Design** and the team's BAU responsibilities: storage operations and the preservation applications (Rosetta + Whole-of-Domain archive).

Full timing / EDE accounting is maintained in a separate internal Word document (not in this repo).

> [!IMPORTANT]
> **Before you start any module**, every trainee must request a DIA-issued sandbox subscription and create their own personally-named resource group. Read [Step 00 — Lab environment setup](step-00-lab-environment-setup.md) and complete its checklist first. **Your RG name must include your initials** (e.g. `rg-labs-foundations-sq`) so it doesn't collide with other trainees in the shared sandbox.

---

## 👥 Audience

- Technically capable; deep digital-preservation domain knowledge.
- Comfortable at the Linux/RHEL OS level.
- **No prior Azure experience** assumed.

Starts at zero Azure knowledge; finishes at "I can operate our preservation system in production."

---

## 🗺️ Curriculum

The plan was revised on 11-May-2026 to align with the DP team's actual scope (per Emma's feedback). Modules outside DP ownership — AZ-900-level foundations, immutability, Rosetta architecture, Oracle administration — have been removed; closely related observability topics have been consolidated.

### Phase 1 — Foundations
| # | Module | Notes |
|---|---|---|
| 00 | [Lab environment setup](step-00-lab-environment-setup.md) | Sandbox subscription + personal RG (`rg-labs-foundations-<initials>`) |
| 01 | [Portal & Cloud Shell tour](step-01-portal-and-cloud-shell.md) | |
| 02 | [Identity & access for the operator](step-02-identity-and-access.md) | |
| 03 | [Guardrails, governance & audit](step-03-governance-guardrails.md) | Lite — awareness only (Platforms team owns) |
| 04 | [Networking primer (read-only view)](step-04-networking-primer.md) | |

### Phase 2 — Storage
| # | Module | Notes |
|---|---|---|
| 05 | [Storage accounts deep-dive](step-05-storage-accounts-deep-dive.md) | Includes Hot/Cool/Cold lifecycle (Archive tier excluded — not used at DIA) |
| 06 | [Cold tier retrieval & rehydration cost](step-06-cold-retrieval.md) | |
| 07 | [Azure Files (NFS & SMB)](step-07-azure-files.md) | Trimmed — DSR-specific items only |
| 08 | [Data protection posture](step-08-data-protection.md) | |
| 09 | [Blob inventory & capacity reporting](step-09-blob-inventory.md) | |

### Phase 3 — Applications
| # | Module | Notes |
|---|---|---|
| 10 | [Application Gateway + WAF for operators](step-10-app-gateway-waf.md) | Trimmed — Rosetta-down triage only (Service Reliability owns the rest) |
| 11 | [WOD container operations](step-11-wod-container-ops.md) | **TBC** — confirm scope with Emma before scheduling |

### Phase 4 — Observability & Cost
| # | Module | Notes |
|---|---|---|
| 12 | [Operational Visibility & Alerts](step-12-operational-visibility.md) | Azure Monitor + Backup Center + Defender for Storage awareness, consolidated |
| 13 | [Azure Monitor Workbooks](step-13-workbooks.md) | |
| 14 | [Cost Management for an application owner](step-14-cost-management.md) | |

### Phase 5 — Capstones
| # | Module | Notes |
|---|---|---|
| 15 | [Capstone: Weekly Health Report](step-15-capstone-weekly-health.md) | Co-design workshop — team defines KPIs live |
| 16 | [Capstone: Monthly Cost Report](step-16-capstone-monthly-cost.md) | |
| 17 | [Capstone: Incident triage tabletop](step-17-capstone-incident-triage.md) | |

---

## 🚦 How to start

1. **Set up your sandbox** — follow [Step 00 — Lab environment setup](step-00-lab-environment-setup.md). Don't skip this; every later lab assumes it. **Your resource group name must include your initials** to avoid collisions in the shared sandbox subscription.
2. Work through the phases in order: **Foundations → Storage → Applications → Observability & Cost → Capstones**.
3. Each module is self-contained markdown — usable for live sessions, self-paced study, or reference for new starters.

Recommended pacing: **2 modules per week over ~9 weeks**.

---

## 💰 Cost expectations

- **Pay-as-you-go:** ~NZD $20–30 end-to-end if you tear down between labs.
- **Free Azure trial:** $0 (NZD ~$300 credit, 30 days).
- **Visual Studio / MPN credits:** $0.

Each module lists its own lab cost and a teardown command. Always tear down at the end of a session.

---

## 🛡️ Scope boundary

This curriculum covers **only** what the Digital Preservation Team owns. Topics owned by other DIA teams (Platform / Core Support / Cloud Networking / Cloud Security / Database / FinOps) are covered as **read-only / awareness only**, with clear escalation paths.

---

## 🤝 Contributing

Found a mistake? Have a better example? Open a pull request — this curriculum is meant to grow with the team.
