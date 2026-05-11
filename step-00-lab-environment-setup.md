# Step 00 — Lab environment setup (your DIA sandbox)

_Read this once, before you start the training. 30–45 minutes._ ✈️

Every lab in this curriculum is hands-on. To run them safely the Preservation Team uses a **shared DIA-issued sandbox subscription** — separate from production DSR — where each trainee owns their own resource group. This page walks you through getting access and verifying it works.

> [!IMPORTANT]
> **Do NOT use the production DSR subscription (`dsr-anl-prd`) for any lab.** All labs run in the shared training sandbox below, and every trainee operates inside **their own resource group** (named with their initials) so deployments never collide.

---

## 1. Get access to the shared training sandbox

The Preservation Team uses **one shared DIA-issued sandbox subscription** for all training labs. It is paid for by DIA and reused across the whole training programme so resources persist between sessions. Each trainee gets **Contributor** access to that single subscription and operates inside their **own personally-named resource group** — there is no per-trainee subscription.

**How to request access:**

Contact your team lead or raise a request with **DIA Cloud Platform** with the following details:

> **Subject:** Training sandbox access — Preservation Team
>
> **Purpose:** Azure training programme for the Archives Library Digital Preservation Team.
> **Scope needed:** Contributor on the shared Preservation Team training sandbox subscription (not DSR PRD/UAT/DEV).
> **Duration:** Duration of the training programme (~9 weeks from start date).
> **Tagging convention I will use:** `purpose=dsr-training`, `owner=<your-email>`, `app_name=training`, `env=lab`.

You should receive: the **shared sandbox subscription name and ID**, confirmation of Contributor RBAC, and the Microsoft Entra tenant to sign in to. Every trainee in the cohort lands on the same subscription — your isolation comes from your own resource group (Section 6 below).

> [!TIP]
> If your Contributor grant isn't ready by the time the training starts, ask Cloud Platform for **Reader** on the sandbox first so you can at least see other trainees' RGs and follow along.

---

## 2. Sign in and verify

1. Open [portal.azure.com](https://portal.azure.com) in your browser.
2. Sign in with your DIA work account.
3. Top-right profile → confirm:
   - The **directory name** is the DIA tenant
   - The **subscription name** matches the shared training sandbox Cloud Platform pointed you at (use Switch Directory if you see the wrong one)
4. Open **Cloud Shell** (the `>_` icon in the top bar). Choose **Bash**. Accept the storage prompt — it costs ~NZD $0.05/month and persists your scripts across sessions.
5. Run:

   ```bash
   az account show --output table
   ```

   ✅ You should see the shared sandbox subscription name + ID. Save the ID — you'll reference it.

---

## 3. (Optional) Install Azure CLI on your laptop

You can do every lab inside Cloud Shell, no install needed. Install locally only if you prefer your own terminal.

- **Windows:** download from [aka.ms/installazurecliwindows](https://aka.ms/installazurecliwindows)
- **macOS:** `brew install azure-cli`
- **RHEL / Linux:** `sudo dnf install azure-cli` (see [official install docs](https://learn.microsoft.com/cli/azure/install-azure-cli))

Sign in: `az login`.

---

## 4. Create a Microsoft Learn account

Each lab links to MS Learn pre-work modules. To get progress tracking and certifications later:

1. Go to [learn.microsoft.com](https://learn.microsoft.com).
2. **Sign in** top-right with your **work email** (not the training Azure account).
3. Pick a display name and confirm.

✅ MS Learn pre-work links from each lab will now track your progress.

---

## 5. (Optional but recommended) Request Reader on DSR DEV / UAT

Some later labs (e.g. AGW + WAF, WOD) are easier when you can *look at* the production-like resources. You don't need to change anything — just read.

Email your team lead or DIA Core Support requesting:

> **Reader** role on the `dsr-anl-dev` and `dsr-anl-uat` subscriptions for the duration of the training (~9 weeks).

If access doesn't arrive in time, those labs include screenshots and KQL/CLI snippets you can run against your own training resources instead.

---

## 6. Create your personal "labs" resource group

Every lab will deploy into a single resource group **owned by you** so cleanup is one command at the end. Because the sandbox subscription is shared by the whole cohort, **your RG name must include your initials** — that's what keeps your work isolated from other trainees'.

> [!IMPORTANT]
> **Your RG name MUST include your initials.** The sandbox subscription is shared across the whole training cohort — if two trainees both create `rg-labs-foundations`, deployments collide and one trainee will overwrite the other's resources. The `<your-initials>` suffix (e.g. `-sq`, `-jb`) keeps every trainee's RG, tags, and lab resources isolated. Use the **same initials** for every later lab (storage accounts, VMs, vaults, workspaces) so cleanup is one `az group delete` at the end.

Run once:

```bash
az group create \
  --name rg-labs-foundations-<your-initials> \
  --location australiaeast \
  --tags purpose=dsr-training app_name=training env=lab owner=<your-email>
```

Replace `<your-initials>` (e.g. `sq` for Shanshan Qu) and `<your-email>`.

Verify your RG name is unique in the subscription before continuing:

```bash
az group list --query "[?starts_with(name,'rg-labs-foundations')].name" -o tsv
```

You'll see other trainees' RGs in the output — that's expected. If you see one with **your** initials already, pick a longer suffix (e.g. add a digit or a second letter from your surname) and re-run `az group create`.

> [!TIP]
> Most labs use Australia East as the default region (closest to NZ with all features available). New Zealand North also works for most resources but lacks paired-region replication options.

---

## 7. Tag your resource group (audit hygiene)

You already passed `--tags` during creation in Section 6, but if you need to add or update them later:

```bash
az tag update --resource-id $(az group show -n rg-labs-foundations-<your-initials> --query id -o tsv) \
  --operation merge \
  --tags purpose=dsr-training owner=<your-email> app_name=training env=lab
```

> [!NOTE]
> **Do not tag the subscription** — it's shared with the rest of the cohort and your tags would overwrite theirs. Apply training tags at the **resource group** level so each trainee's work is independently traceable in Cost Management.

---

## 8. Final verification

```bash
az account list --output table
az group list --query "[?name=='rg-labs-foundations-<your-initials>']" --output table
```

✅ The first command shows the shared training sandbox subscription. The second shows **your** `rg-labs-foundations-<your-initials>` resource group with its tags.

---

## ✅ Pre-flight checklist

Before you start Step 01, confirm:

- [ ] DIA Cloud Platform has granted you Contributor on the **shared** training sandbox subscription (not production)
- [ ] You can sign in to [portal.azure.com](https://portal.azure.com) and see the sandbox
- [ ] You can open Cloud Shell and `az account show` works
- [ ] You have a Microsoft Learn account (work email)
- [ ] You've requested Reader on DSR DEV/UAT (acknowledgement is enough — actual grant can come later)
- [ ] You've created `rg-labs-foundations-<your-initials>` **with your own initials** — not someone else's, not the unsuffixed name
- [ ] Your RG is tagged `purpose=dsr-training` + `owner=<your-email>`

---

## 💰 Total cost expectation across all 17 modules

All training spend lands on the shared DIA-issued sandbox subscription — **$0 out-of-pocket** for trainees. Estimated total spend on the sandbox **per trainee** if every lab is torn down promptly: **~NZD $15–25** across the 17 modules (Steps 00–16).

The single most expensive lab is **Step 09 (Application Gateway + WAF)** at ~NZD $5 if left running >2 hours. Tear down promptly.

Each lab page has its own cost note + a **Cleanup** command at the end — always run it. Because the sandbox is shared, **leaving resources running affects every trainee's combined bill** — cleanup is a courtesy, not just hygiene.

---

➡️ **Next:** [Step 01 — Portal & Cloud Shell tour](step-01-portal-and-cloud-shell.md)
