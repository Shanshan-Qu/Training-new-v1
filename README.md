# DSR Training — Digital Preservation Team

Self-paced and instructor-led Azure training for the **Archives Library Digital Preservation Team** at the Department of Internal Affairs (DIA), New Zealand.

The curriculum is grounded in the **DIA DSR DPS Azure Application Landing Zone Design** and the team's BAU responsibilities: storage operations and the preservation applications (Rosetta + Whole-of-Domain archive).

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

The plan was revised on 11-May-2026 to align with the DP team's actual scope. Modules outside DP ownership — AZ-900-level foundations, immutability, Rosetta architecture, Oracle administration — have been removed; closely related observability topics have been consolidated.

### Phase 1 — Foundations
| # | Module | Lab time |
|---|---|---|
| 00 | [Lab environment setup](step-00-lab-environment-setup.md) | 60 min |
| 01 | [Portal & Cloud Shell tour](step-01-portal-and-cloud-shell.md) | 60 min |
| 02 | [Identity & access for the operator](step-02-identity-and-access.md) | 90 min |
| 03 | [Guardrails, governance & audit](step-03-governance-guardrails.md) | 60 min |
| 04 | [Networking primer (read-only view)](step-04-networking-primer.md) | 90 min |
| | **Phase 1 subtotal** | **~6 h** |

### Phase 2 — Storage
| # | Module | Lab time |
|---|---|---|
| 05 | [Storage accounts & tier management](step-05-storage-accounts-and-tiers.md) | 120 min |
| 06 | [Azure Files (DSR operational view)](step-06-azure-files.md) | 60 min |
| 07 | [Data protection posture](step-07-data-protection.md) | 90 min |
| 08 | [Blob inventory & capacity reporting](step-08-blob-inventory.md) | 90 min |
| | **Phase 2 subtotal** | **~6 h** |

### Phase 3 — Applications
| # | Module | Lab time |
|---|---|---|
| 09 | [WOD container operations](step-09-wod-container-ops.md) | 90 min |
| | **Phase 3 subtotal** | **~1 h 30 min** |

### Phase 4 — Observability & Cost
| # | Module | Lab time |
|---|---|---|
| 10 | [Operational Visibility & Alerts](step-10-operational-visibility.md) | 180 min |
| 11 | [Azure Monitor Workbooks](step-11-workbooks.md) | 90 min |
| 12 | [Cost Management for an application owner](step-12-cost-management.md) | 90 min |
| | **Phase 4 subtotal** | **~6 h** |

### Phase 5 — Capstones

> [!WARNING]
> **Phase 5 is on hold.** The three Capstones depend on real signals from a DSR non-production environment (Workbooks, cost data, incident triage against live AGW / Backup Center / Defender alerts). Hold scheduling until the customer has a non-prod environment available for trainees to test against. Until then, treat the content below as a draft — the queries and Workbook structure are ready, only the data source is missing.

| # | Module | Lab time |
|---|---|---|
| 13 | [Capstone: Weekly Health Report](step-13-capstone-weekly-health.md) | 180 min |
| 14 | [Capstone: Monthly Cost Report](step-14-capstone-monthly-cost.md) | 180 min |
| 15 | [Capstone: Incident triage tabletop](step-15-capstone-incident-triage.md) | 180 min |
| | **Phase 5 subtotal** | **~9 h** |

### Phase 6 — Infrastructure as Code

> [!WARNING]
> **Status: DRAFT.** Scope and depth pending discussion with the DIA Platform team. Goal is read-only fluency with the existing DSR Terraform repo, not authoring new modules.

| # | Module | Lab time |
|---|---|---|
| 16 | [Terraform & IaC for DSR (read-only)](step-16-terraform-iac.md) | 90 min |
| | **Phase 6 subtotal** | **~1 h 30 min** |

**Total hands-on lab time: ~30 hours** across 16 modules (excluding pre-reading and Step 00 setup). **Phase 5 (~9 h) is on hold pending DSR non-prod environment availability** — see warning above. **Phase 6 is draft pending DIA Platform team confirmation.**

---

## 🚦 How to start

1. **Set up your sandbox** — follow [Step 00 — Lab environment setup](step-00-lab-environment-setup.md). Don't skip this; every later lab assumes it. **Your resource group name must include your initials** to avoid collisions in the shared sandbox subscription.
2. Work through the phases in order: **Foundations → Storage → Applications → Observability & Cost → Capstones → Infrastructure as Code**.
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
