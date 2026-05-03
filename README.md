# DSR Training — Digital Preservation Team

Self-paced and instructor-led Azure training for the **Archives Library Digital Preservation Team** at the Department of Internal Affairs (DIA), New Zealand.

The curriculum is grounded in the **DIA DSR DPS Azure Application Landing Zone Design** and the team's BAU responsibilities: storage operations and the preservation applications (Rosetta + Whole-of-Domain archive).

For full timing/EDE accounting, see [DIA-DPS-Training-Plan-and-EDE-Hours.docx](DIA-DPS-Training-Plan-and-EDE-Hours.docx).

> [!IMPORTANT]
> **Before Step 01**, every trainee must set up their own personal Azure sandbox. Read [lab-environment-setup.md](lab-environment-setup.md) and complete its checklist first.

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
| 04 | [Guardrails, governance & audit](step-04-governance-guardrails.md) |
| 05 | [Networking primer (read-only view)](step-05-networking-primer.md) |

### Storage
| # | Module |
|---|---|
| 06 | [Storage accounts deep-dive](step-06-storage-accounts-deep-dive.md) |
| 07 | [Blob lifecycle management](step-07-blob-lifecycle.md) |
| 08 | [Cold tier retrieval & rehydration cost](step-08-cold-retrieval.md) |
| 09 | [Azure Files (NFS & SMB)](step-09-azure-files.md) |
| 10 | [Data protection posture](step-10-data-protection.md) |
| 11 | [Immutability & legal hold](step-11-immutability.md) |
| 12 | [Blob inventory & capacity reporting](step-12-blob-inventory.md) |

### Applications
| # | Module |
|---|---|
| 13 | [Rosetta architecture walkthrough (read-only)](step-13-rosetta-architecture.md) |
| 14 | [Application Gateway + WAF for operators](step-14-app-gateway-waf.md) |
| 15 | [Oracle on Azure (read-only ops view)](step-15-oracle-on-azure.md) |
| 16 | [WOD container operations](step-16-wod-container-ops.md) |

### Observability & Cost
| # | Module |
|---|---|
| 17 | [Azure Monitor: Logs, Metrics & Alerts](step-17-log-analytics-kql.md) |
| 18 | [Azure Monitor Workbooks](step-18-workbooks.md) |
| 19 | [Cost Management for an application owner](step-19-cost-management.md) |

### Read-only operations
| # | Module |
|---|---|
| 20 | [Backup Center read-only operations](step-20-backup-center.md) |
| 21 | [Defender for Storage (awareness)](step-21-defender-storage.md) |

### Capstones
| # | Module |
|---|---|
| 22 | [Capstone: Weekly Health Report](step-22-capstone-weekly-health.md) |
| 23 | [Capstone: Monthly Cost Report](step-23-capstone-monthly-cost.md) |
| 24 | [Capstone: Incident triage tabletop](step-24-capstone-incident-triage.md) |

---

## 🚦 How to start

1. **Set up your Azure sandbox** — follow [lab-environment-setup.md](lab-environment-setup.md). Don't skip this; every later lab assumes it.
2. Work through the **Foundations** stream (Modules 01–05) first — every later lab assumes it.
3. Follow stream order: Storage → Applications → Observability & Cost → Read-only ops → Capstones.
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
