# Step 00 — Lab environment setup (your DIA sandbox)

_Read this once, before you start the training. 30–45 minutes._ ✈️

Every lab in this curriculum is hands-on. To run them safely you need your **own** Azure subscription — separate from production DSR — so you can deploy, break, and tear down without risk. This page walks you through getting one and verifying it works.

> [!IMPORTANT]
> **Do NOT use the production DSR subscription (`dsr-anl-prd`) for any lab.** Use the DIA-issued sandbox below. All labs include teardown commands so DIA's training spend stays minimal.

---

## 1. Request your DIA-issued sandbox

The Preservation Team uses a **DIA-issued sandbox subscription** for all training labs. It is paid for by DIA, scoped per-trainee, and keeps the same subscription for all 24 modules so your resources persist between sessions.

**How to request it:**

Contact your team lead or raise a request with **DIA Cloud Platform** with the following details:

> **Subject:** Sandbox subscription for Preservation Team training
>
> **Purpose:** 12-week Azure training programme for the Archives Library Digital Preservation Team.
> **Scope needed:** Contributor on a dedicated sandbox subscription (not DSR PRD/UAT/DEV).
> **Duration:** 12 weeks from start date.
> **Tagging:** `purpose=dsr-training`, `owner=<your-email>`, `app_name=training`, `env=lab`.

You should receive: a subscription name (something like `sub-dia-sandbox-<initials>`), confirmation of Contributor RBAC, and the Microsoft Entra tenant to sign in to.

> [!TIP]
> If your sandbox isn't ready by the time the training starts, ask Cloud Platform whether you can temporarily share another trainee's sandbox with **Reader** access so you don't fall behind.

---

## 2. Sign in and verify

1. Open [portal.azure.com](https://portal.azure.com) in your browser.
2. Sign in with your DIA work account.
3. Top-right profile → confirm:
   - The **directory name** is the DIA tenant
   - The **subscription name** matches the sandbox Cloud Platform issued you (use Switch Directory if you see the wrong one)
4. Open **Cloud Shell** (the `>_` icon in the top bar). Choose **Bash**. Accept the storage prompt — it costs ~NZD $0.05/month and persists your scripts across sessions.
5. Run:

   ```bash
   az account show --output table
   ```

   ✅ You should see your sandbox subscription name + ID. Save the ID — you'll reference it.

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

> **Reader** role on the `dsr-anl-dev` and `dsr-anl-uat` subscriptions for the duration of the training (~12 weeks).

If access doesn't arrive in time, those labs include screenshots and KQL/CLI snippets you can run against your own training resources instead.

---

## 6. Create your "labs" resource group

Every lab will deploy into a single shared RG so cleanup is one command. Run once:

> [!IMPORTANT]
> **Your RG name MUST include your initials.** The sandbox subscription is shared across the whole training cohort — if two trainees both create `rg-labs-foundations`, deployments collide and one trainee will overwrite the other's resources. The `<your-initials>` suffix (e.g. `-sq`, `-jb`) keeps every trainee's RG, tags, and lab resources isolated. Use the **same initials** for every later lab (storage accounts, VMs, vaults, workspaces) so cleanup is one `az group delete` at the end.

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

If you see another trainee's RG with the same initials, pick a longer suffix (e.g. add a digit or a second letter from your surname) and re-run `az group create`.

> [!TIP]
> Most labs use Australia East as the default region (closest to NZ with all features available). New Zealand North also works for most resources but lacks paired-region replication options.

---

## 7. Tag your subscription (audit hygiene)

```bash
SUB=$(az account show --query id -o tsv)
az tag create --resource-id /subscriptions/$SUB \
  --tags purpose=dsr-training owner=<your-email>
```

This makes your training spend easy to spot in Cost Management later.

---

## 8. Final verification

```bash
az account list --output table
az group list --output table
```

✅ The first command lists your training subscription. The second shows your `rg-labs-foundations-<your-initials>` resource group (probably the only one — that's fine).

---

## ✅ Pre-flight checklist

Before you start Step 02, confirm:

- [ ] DIA Cloud Platform has issued you a sandbox subscription (not production)
- [ ] You can sign in to [portal.azure.com](https://portal.azure.com) and see the sandbox
- [ ] You can open Cloud Shell and `az account show` works
- [ ] You have a Microsoft Learn account (work email)
- [ ] You've requested Reader on DSR DEV/UAT (acknowledgement is enough — actual grant can come later)
- [ ] Your subscription is tagged `purpose=dsr-training`
- [ ] You've created `rg-labs-foundations-<your-initials>` **with your own initials** (not someone else's, not the unsuffixed name)

---

## 💰 Total cost expectation across all 24 labs

All training spend lands on the DIA-issued sandbox subscription — **$0 out-of-pocket** for trainees. Estimated total spend on the sandbox if every lab is torn down promptly: **~NZD $20–30** across 24 modules.

The single most expensive lab is **Step 09 (Azure Files NFS)** at ~NZD $3 if left running >2 hours. Tear down promptly.

Each lab page has its own cost note + a **Cleanup** command at the end — always run it.

---

➡️ **Next:** [Step 02 — Portal & Cloud Shell tour](step-02-portal-and-cloud-shell.md)
