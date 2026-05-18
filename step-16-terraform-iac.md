# Step 16 — Terraform & IaC for DSR (read-only)

> [!WARNING]
> **STATUS: DRAFT — pending DIA discussion.** Scope and depth to be confirmed with the DSR Platform team. The goal of this module is **reading and tracing the existing Terraform code**, not authoring new modules. Update this banner once scope is signed off.

_The "find it in the code" lab._ 🧭 Builds the ability to open the DSR Terraform repository, locate the resource that backs a given Azure object, follow variables and module inputs end-to-end, and read a plan output safely. No `terraform apply` is performed by the Preservation Team.

> [!NOTE]
> **Trainee duration:** 90 minutes
> **Lab cost:** $0 — entirely local (`git clone` + `terraform init -backend=false` + `terraform validate`/`plan` against a sandbox only).
> **Prerequisites:** Steps 00–04 complete. Git installed locally, Terraform CLI ≥ 1.6 installed, Reader on the ALNZ dev subscription.
> **Pairs with:** the DSR Platform team's Terraform repo (link to be added once DIA confirms which repo / branch trainees should read).

---

## 📖 Session overview

The DSR landing zone — networking, storage accounts, Log Analytics workspaces, RBAC assignments, diagnostic settings, Backup vaults — is defined in **Terraform**, owned by the **DSR Platform team**. The Preservation Team does **not** apply Terraform in production, but you are expected to:

- Find the Terraform definition behind any Azure resource you operate (e.g. "where is `stalnznznblobprdwod01` declared?").
- Read a Terraform plan output produced by Platform and identify whether a proposed change will impact your application.
- Raise a change request against the right module/variable rather than asking for a portal change.
- Spot drift between Portal state and the declared state.

This lab is **read-only**: clone, browse, validate, plan against a sandbox. Nothing in production is mutated.

**What you'll learn**
- Terraform repo layout the DSR Platform team uses (modules, environments, state).
- How to map an Azure resource name back to its `.tf` declaration.
- How to read `terraform plan` output: adds / changes / destroys / replacements.
- How remote state and backend config work — and why you should never run `terraform apply` from your workstation.
- How to file a well-scoped change request: file path, variable, env, justification.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **IaC** | Infrastructure as Code — describing cloud resources in text files instead of clicking in the portal. |
| **Terraform** | HashiCorp's IaC tool. Reads `.tf` files, talks to Azure ARM, makes reality match the code. |
| **HCL** | HashiCorp Configuration Language — the syntax used in `.tf` files. |
| **Provider** | The plugin that knows how to talk to a cloud — `azurerm`, `azuread`, etc. |
| **Module** | A reusable bundle of `.tf` files. DSR has modules like `storage_account`, `law_workspace`. |
| **Resource** | A single Azure thing declared in code, e.g. `azurerm_storage_account.wod`. |
| **Data source** | Read-only lookup of something Terraform did **not** create (existing subscription, existing key vault, etc.). |
| **Variable** | A named input to a module — e.g. `env = "prd"`. |
| **State** | Terraform's record of what it has created. Stored remotely (Azure Storage backend) — **never** edit by hand. |
| **Backend** | Where state lives. DSR uses an Azure Storage backend with a per-environment state file. |
| **Plan** | Dry-run output: what Terraform *would* do. The artefact your team reads before approving a change. |
| **Apply** | Execute the plan. Only Platform pipeline service principals do this in DSR. |
| **Drift** | When the Portal reality no longer matches the code (someone clicked something). |

## 📚 Prepare in advance — HashiCorp & Microsoft Learn

| Module | Why it matters for ALNZ |
|---|---|
| [Build infrastructure on Azure with Terraform](https://learn.microsoft.com/training/paths/terraform-fundamentals/) | The official intro path — modules 1–3 are enough for read-only work. |
| [Terraform `azurerm` provider docs](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs) | The reference you'll use most often when reading a resource block. |
| [Terraform CLI commands](https://developer.hashicorp.com/terraform/cli/commands) | Focus on `init`, `validate`, `plan`, `show`, `state list`, `state show`. |

About **2 hours** of optional pre-reading. Skip the apply / state-manipulation sections.

## 🧱 Foundational primer

- **The Preservation Team reads Terraform; it does not apply it.** All `apply` happens from Platform's CI/CD pipeline using a managed identity / service principal — never from a laptop.
- **State is sacred.** Do not run `terraform import`, `terraform state rm`, or `terraform state mv` outside a Platform-led session. State corruption is the worst-case outcome.
- **Modules are the unit of reuse.** A single `module "wod_storage"` call in `envs/prd/main.tf` may instantiate a storage account, a private endpoint, a private DNS record, diagnostic settings, and an RBAC assignment in one block.
- **Naming convention is in code.** The reason a storage account is called `stalnznznblobprdwod01` is a `locals.tf` somewhere that concatenates `app + region + workload + env + purpose + instance`. Find that file once and the rest of the repo becomes legible.
- **`terraform plan` is safe**, but only if you point it at a sandbox subscription with a backend disabled (`-backend=false`) or a personal state file. Never plan against the production backend from your workstation.
- **Drift is a signal, not a verdict.** If `plan` says "1 to change" against unchanged code, something was clicked. Open a ticket with Platform — do not "fix" it by clicking again.

## ⌨️ Activity 1 — Clone and inventory the repo

> Replace `<dsr-tf-repo-url>` with the repo URL DIA Platform team provides.

```bash
git clone <dsr-tf-repo-url> dsr-iac
cd dsr-iac
# Get the lay of the land
find . -maxdepth 3 -type d | sort
ls envs/      # expected: dev/ uat/ prd/
ls modules/   # expected: storage_account/ law_workspace/ private_endpoint/ ...
```

Open the repo in VS Code. Note the convention:
- `envs/<env>/` — per-environment composition (which modules, which variables).
- `modules/<name>/` — reusable resource bundles.
- `locals.tf` / `naming.tf` — the naming convention (where `stalnznzn*` is generated).

## ⌨️ Activity 2 — Trace a real resource end-to-end

Pick one resource you operate. Worked example: `stalnznznblobprdwod01`.

1. **Search for the name fragment** the convention will produce:
   ```bash
   # The literal name is constructed; search for the workload key
   rg -n "wod" envs/prd/ modules/storage_account/
   ```
2. **Find the module call** in `envs/prd/main.tf` — likely something like:
   ```hcl
   module "wod_storage" {
     source   = "../../modules/storage_account"
     workload = "wod"
     env      = "prd"
     # ...
   }
   ```
3. **Open the module** `modules/storage_account/main.tf`. Identify:
   - The `azurerm_storage_account` resource block.
   - Where `name` is built — usually `local.name` from `locals.tf`.
   - The `network_rules` block (public access disabled, private endpoint subnet).
   - The diagnostic-settings child resource (where the LAW workspace is wired in).
4. **Map back to the Portal.** Open the Portal blade for `stalnznznblobprdwod01` and confirm what you read in code matches: SKU, redundancy, public access, soft delete, diagnostic settings target.

Repeat the exercise for: a Log Analytics workspace, a Backup vault, an RBAC assignment.

## ⌨️ Activity 3 — Safe local plan against a sandbox

> Uses your personal sandbox RG from Step 00. Never against `dsr-alnz-prd`.

```bash
cd envs/sandbox     # or whichever env DIA designates for trainee plans
# Init without touching the real remote state
terraform init -backend=false
terraform validate

# Plan against your sandbox subscription — auth via az login first
az login
az account set --subscription "<your sandbox subscription id>"
terraform plan -var "env=sandbox" -var "owner=<your-initials>" -out tfplan
terraform show tfplan | less
```

Read the output. Categorise each line: `+ create`, `~ change in-place`, `-/+ replace`, `- destroy`. Make sure you understand which Azure operations each implies (a `replace` on a storage account = data loss).

## ⌨️ Activity 4 — Read a real Platform-produced plan

Ask DIA Platform for a recent `terraform plan` artefact (often attached to a pull request). For each change:

1. Identify the **resource type** and the **module** it lives in.
2. Identify the **input variable** that drove the change.
3. Decide: does this affect a resource the Preservation Team owns or depends on? (network rules on a storage account you mount? a tag on the WOD blob account? a diagnostic-settings rewire that would break a Workbook?)
4. Comment on the PR with your read — even a "no impact to DP" comment is valuable signal.

## 🧪 Stretch goals

- Open `modules/storage_account/locals.tf` and document — in your own words, in a markdown file — the full naming convention. Cross-check three real resource names against it.
- Use `terraform state list` against the **sandbox** state (read-only) to enumerate every resource Terraform owns there. Compare with `az resource list -g <sandbox-rg>` and explain any deltas.
- Find one resource in the Portal (in `dsr-alnz-dev`, where you have Reader) that does **not** appear in Terraform state. Is it expected manual provisioning (e.g. classic admin), or is it drift? File a Platform ticket if drift.

## ✅ You're done when…

- [ ] You can clone the DSR Terraform repo and explain its top-level layout in two sentences.
- [ ] Given any storage account / LAW / Backup vault name in DSR, you can find the `.tf` file that declares it within five minutes.
- [ ] You can read a `terraform plan` and classify each change as create / in-place change / replace / destroy.
- [ ] You know which operations you must never run (`apply`, `state rm`, `state mv`, `import`) on the DSR repo from your workstation.
- [ ] You can write a change request to Platform that specifies file path, variable, environment, and justification.

## 🧹 Teardown

```bash
# Nothing was deployed — just clean local artefacts
rm -rf .terraform tfplan terraform.tfstate*
```

If you ran `terraform plan` against your sandbox subscription, no resources were created (plan is read-only). Nothing to delete in Azure.

---

**Next:** This is the final module of the curriculum. Return to the [README](README.md) for the full map, or revisit the capstones in Phase 5 once a DSR non-prod environment is available.
