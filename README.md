# DSR Training — Digital Preservation Team

Self-paced and instructor-led Azure training for the **Archives Library Digital Preservation Team** at the Department of Internal Affairs (DIA), New Zealand.

The training is grounded in the **DIA DSR DPS Azure Application Landing Zone Design v1.4** and follows the team's BAU responsibilities: storage operations and the preservation applications (Rosetta + Whole-of-Domain archive).

---

## 👥 Audience

- Technically capable; deep digital-preservation domain knowledge.
- Comfortable at the Linux/RHEL OS level.
- **No prior Azure experience** for most of the team.

This curriculum starts at zero Azure knowledge and finishes at "I can run our preservation system in production."

---

## 🗺️ Curriculum at a glance

| # | Module | Stream | Duration | Cost (NZD) |
|---|---|---|---|---|
| 00 | [Environment readiness](step-00-prereqs.md) | Pre-flight | 45 min | $0 |
| 01 | [Azure foundations](step-01-azure-foundations.md) | Foundations | 1.5h | < $0.50 |
| 02 | [Azure Portal foundations](step-02-portal-foundations.md) | Foundations | 1h | $0 |
| 03 | [Identity & access for the operator](step-03-identity-access.md) | Foundations | 1.25h | < $0.50 |
| 04 | [Guardrails, governance & audit](step-04-governance.md) | Foundations | 1.5h | < $0.50 |
| 05 | [Networking primer (read-only view)](step-05-networking.md) | Foundations | 1.5h | < $1 |
| 06 | [Storage accounts deep-dive](step-06-storage-accounts.md) | Storage | 1.5h | < $1 |
| 07 | [Reading the lifecycle policy](step-07-lifecycle.md) | Storage | 1h | < $1 |
| 08 | [Azure Files: NFS vs SMB](step-08-azure-files.md) | Storage | 2h | < $3 |
| 09 | [Data protection posture](step-09-data-protection.md) | Storage | 1.5h | < $1 |
| 10 | [Immutability & legal hold](step-10-immutability.md) | Storage | 1.25h | < $1 |
| 11 | [Blob inventory](step-11-blob-inventory.md) | Storage | 1.5h | < $1 |
| 12 | [Rosetta architecture walkthrough](step-12-rosetta-architecture.md) | Apps | 1.5h | $0 |
| 13 | [Application Gateway + WAF for operators](step-13-app-gateway.md) | Apps | 2h | < $4 |
| 14 | [Oracle on Azure (read-only ops view)](step-14-oracle.md) | Apps | 1.25h | $0 |
| 15 | [WOD container operations](step-15-wod-containers.md) | Apps | 1.5h | < $2 |
| 16 | [Azure Monitor — foundations (storage focus)](step-16-monitor-foundations.md) | Observability | 2h | $0 |
| 17 | [Azure Monitor — hands-on labs (storage focus)](step-17-monitor-hands-on.md) | Observability | 3h | < $1 |
| 18 | [Workbooks: built-in first, then custom](step-18-workbooks.md) | Observability | 2h | $0 |
| 19 | [Reading the lifecycle cost (light)](step-19-lifecycle-cost.md) | Cost | 1h | < $1 |
| 20 | [Cost Management with forecasting](step-20-cost-management.md) | Cost | 2h | $0 |
| 21 | [Backup Center — read-only operations](step-21-backup-center.md) | Read-only ops | 1h | $0 |
| 22 | [Defender for Storage — awareness only](step-22-defender.md) | Read-only ops | 45 min | $0 |
| 23 | [Reporting — scheduled & on-demand](step-23-reporting.md) | Capstone | 1.5h | $0 |
| 24 | [Healthy Report — assembled](step-24-healthy-report.md) | Capstone | 1.5h | $0 |
| 25 | [Cost Report — assembled (with forecast)](step-25-cost-report.md) | Capstone | 1.5h | $0 |
| 26 | [Incident triage tabletop](step-26-incident-triage.md) | Capstone | 2h | $0 |

**Optional stream:** Terraform on Azure — deferred. See [Microsoft Learn — Terraform on Azure](https://learn.microsoft.com/training/paths/terraform-fundamentals/).

---

## ⏱️ Time investment per trainee

| Activity | Hours |
|---|---|
| Self-paced lab time | ~38h |
| Instructor-led delivery | ~24h |
| MS Learn pre-work | ~6h |
| Q&A / review | ~3h |
| **Total EDE per trainee** | **~71h** |

Recommended pacing: **2 modules per week over ~10–12 weeks**.

---

## 💰 Total Azure cost per trainee

- **Pay-as-you-go:** ~NZD $20–30 end-to-end if you tear down between labs.
- **Free Azure trial:** $0 (NZD ~$300 credit, 30 days).
- **Visual Studio / Microsoft Partner benefit credits:** $0.

We recommend a **dedicated training subscription** so cleanup is easy and there's no risk to production.

---

## 🚦 How to start

1. Begin with **[Module 0 — Environment readiness](step-00-prereqs.md)**. Don't skip — every later lab assumes you've completed it.
2. Work through the foundations stream (Modules 1–5) before any storage or app module.
3. Each module is self-contained markdown — usable for live sessions, self-paced study, or as reference for new starters.

---

## 🛡️ Scope boundary

This curriculum covers **only** what the Digital Preservation Team owns. Topics owned by other DIA teams (Platform / Core Support / Cloud Networking / Cloud Security / Database / FinOps) are covered as **read-only / awareness only**, with clear escalation paths.

---

## 🤝 Contributing

Found a mistake? Have a better example? Open a pull request — this curriculum is meant to grow with the team.
