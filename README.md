# DSR Training — Digital Preservation Team

Self-paced and instructor-led Azure training for the **Archives Library Digital Preservation Team** at the Department of Internal Affairs (DIA), New Zealand.

The curriculum is grounded in the **DIA DSR DPS Azure Application Landing Zone Design** and the team's BAU responsibilities: storage operations and the preservation applications (Rosetta + Whole-of-Domain archive).

For full timing/EDE accounting, see [DIA-DPS-Training-Plan-and-EDE-Hours.docx](DIA-DPS-Training-Plan-and-EDE-Hours.docx).

---

## 👥 Audience

- Technically capable; deep digital-preservation domain knowledge.
- Comfortable at the Linux/RHEL OS level.
- **No prior Azure experience** assumed.

Starts at zero Azure knowledge; finishes at "I can operate our preservation system in production."

---

## 🗺️ Curriculum

### Foundations
| # | Module |
|---|---|
| 01 | [Azure foundations](step-01-azure-foundations.md) |
| 02 | [Portal & Cloud Shell tour](step-02-portal-and-cloud-shell.md) |
| 03 | [Identity & access for the operator](step-03-identity-and-access.md) |
| 04 | [Networking primer (read-only view)](step-04-networking-primer.md) |

### Storage
| # | Module |
|---|---|
| 05 | [Storage accounts deep-dive](step-05-storage-accounts-deep-dive.md) |
| 06 | [Blob lifecycle management](step-06-blob-lifecycle.md) |
| 07 | [Cold tier retrieval & rehydration cost](step-07-cold-retrieval.md) |
| 08 | [Azure Files (NFS & SMB)](step-08-azure-files.md) |
| 09 | [Data protection posture](step-09-data-protection.md) |
| 10 | [Immutability & legal hold](step-10-immutability.md) |
| 11 | [Blob inventory & capacity reporting](step-11-blob-inventory.md) |

### Applications
| # | Module |
|---|---|
| 12 | [Rosetta architecture walkthrough (read-only)](step-12-rosetta-architecture.md) |
| 13 | [Application Gateway + WAF for operators](step-13-app-gateway-waf.md) |
| 14 | [Oracle on Azure (read-only ops view)](step-14-oracle-on-azure.md) |
| 15 | [WOD container operations](step-15-wod-container-ops.md) |

### Observability & Cost
| # | Module |
|---|---|
| 16 | [Azure Monitor: Logs, Metrics & Alerts](step-16-log-analytics-kql.md) |
| 17 | [Azure Monitor Workbooks](step-17-workbooks.md) |
| 18 | [Cost Management for an application owner](step-18-cost-management.md) |

### Read-only operations
| # | Module |
|---|---|
| 19 | [Backup Center read-only operations](step-19-backup-center.md) |
| 20 | [Defender for Storage (awareness)](step-20-defender-storage.md) |

### Capstones
| # | Module |
|---|---|
| 21 | [Capstone: Weekly Health Report](step-21-capstone-weekly-health.md) |
| 22 | [Capstone: Monthly Cost Report](step-22-capstone-monthly-cost.md) |
| 23 | [Capstone: Incident triage tabletop](step-23-capstone-incident-triage.md) |

---

## 🚦 How to start

1. Set up a personal training Azure subscription (Azure free trial, DIA-issued sandbox, or Visual Studio credits). **Do not use the production DSR subscription for hands-on labs.**
2. Work through the **Foundations** stream (Modules 01–04) first — every later lab assumes it.
3. Follow stream order: Storage → Applications → Observability & Cost → Read-only ops → Capstones.
4. Each module is self-contained markdown — usable for live sessions, self-paced study, or reference for new starters.

Recommended pacing: **2 modules per week over ~10–12 weeks**.

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
