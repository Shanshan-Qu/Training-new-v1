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

> [!IMPORTANT]
> **Plan revised 11-May-2026** based on Emma's feedback. Several modules have been **dropped**, **trimmed**, or **merged** to better match the DP team's actual scope. See the Status column below; open each module file for delivery notes.

### Foundations
| # | Module | Status |
|---|---|---|
| 01 | ~~Azure foundations~~ | **DROPPED** — AZ-900 already done (file archived: [archived-step-01](archived-step-01-azure-foundations.md)) |
| 02 | [Portal & Cloud Shell tour](step-02-portal-and-cloud-shell.md) | Active |
| 03 | [Identity & access for the operator](step-03-identity-and-access.md) | Active |
| 04 | [Guardrails, governance & audit](step-04-governance-guardrails.md) | **Lite version** — awareness only (Platforms owns) |
| 05 | [Networking primer (read-only view)](step-05-networking-primer.md) | Active |

### Storage
| # | Module | Status |
|---|---|---|
| 06 | [Storage accounts deep-dive](step-06-storage-accounts-deep-dive.md) | **Expanded** — absorbs Step 07 lifecycle content |
| 07 | ~~Blob lifecycle management~~ | **Merged into Step 06** — Archive tier dropped ([merged-step-07](merged-step-07-blob-lifecycle.md)) |
| 08 | [Cold tier retrieval & rehydration cost](step-08-cold-retrieval.md) | Active |
| 09 | [Azure Files (NFS & SMB)](step-09-azure-files.md) | **Trimmed** — team already familiar |
| 10 | [Data protection posture](step-10-data-protection.md) | Active |
| 11 | ~~Immutability & legal hold~~ | **DROPPED** — not in use at DIA ([archived-step-11](archived-step-11-immutability.md)) |
| 12 | [Blob inventory & capacity reporting](step-12-blob-inventory.md) | Active |

### Applications
| # | Module | Status |
|---|---|---|
| 13 | ~~Rosetta architecture walkthrough~~ | **DROPPED** — not required ([archived-step-13](archived-step-13-rosetta-architecture.md)) |
| 14 | [Application Gateway + WAF for operators](step-14-app-gateway-waf.md) | **Trimmed** — Rosetta-down triage only (Service Reliability owns) |
| 15 | ~~Oracle on Azure~~ | **DROPPED** — not required ([archived-step-15](archived-step-15-oracle-on-azure.md)) |
| 16 | [WOD container operations](step-16-wod-container-ops.md) | **TBC** — confirm scope with Emma |

### Observability & Cost
| # | Module | Status |
|---|---|---|
| 17 | [Operational Visibility & Alerts](step-17-azure-monitor.md) | **Expanded & merged** — Azure Monitor + Backup Center + Defender awareness |
| 18 | [Azure Monitor Workbooks](step-18-workbooks.md) | Active |
| 19 | [Cost Management for an application owner](step-19-cost-management.md) | Active |

### Read-only operations
| # | Module | Status |
|---|---|---|
| 20 | ~~Backup Center read-only operations~~ | **Merged into Step 17** — ownership TBC (may belong to Service Reliability) ([merged-step-20](merged-step-20-backup-center.md)) |
| 21 | ~~Defender for Storage (awareness)~~ | **Merged into Step 17** — single awareness slide ([merged-step-21](merged-step-21-defender-storage.md)) |

### Capstones
| # | Module | Status |
|---|---|---|
| 22 | [Capstone: Weekly Health Report](step-22-capstone-weekly-health.md) | **Reshaped** — co-design workshop (team defines KPIs live) |
| 23 | [Capstone: Monthly Cost Report](step-23-capstone-monthly-cost.md) | Active |
| 24 | [Capstone: Incident triage tabletop](step-24-capstone-incident-triage.md) | Active |

---

## 🚦 How to start

1. **Set up your sandbox** — follow [Step 00 — Lab environment setup](step-00-lab-environment-setup.md). Don't skip this; every later lab assumes it. **Your resource group name must include your initials** to avoid collisions in the shared sandbox subscription.
2. Work through the **Foundations** stream (Modules 02–05) first — every later lab assumes it.
3. Follow stream order: Storage → Applications → Observability & Cost → Capstones.
4. Each module is self-contained markdown — usable for live sessions, self-paced study, or reference for new starters.

Recommended pacing: **2 modules per week over ~12 weeks**.

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
