# Lab environment setup — your personal Azure sandbox

_Read this once, before Step 01. 30–45 minutes._ ✈️

Every lab in this curriculum is hands-on. To run them safely you need your **own** Azure subscription — separate from production DSR — so you can deploy, break, and tear down without risk. This page walks you through getting one and verifying it works.

> [!IMPORTANT]
> **Do NOT use the production DSR subscription (`dsr-anl-prd`) for any lab.** Use a personal training subscription. All labs include teardown commands so the cost stays under NZD $30 end-to-end.

---

## 1. Pick a sandbox option

| Option | Best for | Cost | How |
|---|---|---|---|
| **Azure free trial** | Anyone without existing Azure access | $0 (NZD ~$300 credit, 30 days) | Sign up at [azure.microsoft.com/free](https://azure.microsoft.com/free). Requires phone + credit card for verification (no charge unless you exceed the credit). |
| **DIA-issued sandbox** | If DIA Cloud Platform has provisioned one | $0 — paid by DIA | Ask your team lead or open a request with Cloud Platform: "Sandbox subscription for Preservation Team training, 12 weeks, contributor scope." |
| **Visual Studio / Microsoft Partner credits** | If you already have an MSDN/VS subscription | Included monthly credit | Sign in at [portal.azure.com](https://portal.azure.com) with your VS account. |
| **Microsoft Learn Sandbox** | Single MS Learn modules only | Free, 1-hour windows | Activated automatically inside MS Learn modules — does NOT work for these labs (no resource persistence). |

**Recommendation for the team:** request a **DIA-issued sandbox** so you keep the same subscription for all 24 modules and the cost is zero out-of-pocket.

---

## 2. Sign in and verify

1. Open [portal.azure.com](https://portal.azure.com) in your browser.
2. Sign in with the account from step 1.
3. Top-right profile → confirm:
   - The **directory name** (DIA tenant or your personal tenant)
   - The **subscription name** (use Switch Directory if you see the wrong one)
4. Open **Cloud Shell** (the `>_` icon in the top bar). Choose **Bash**. Accept the storage prompt — it costs ~NZD $0.05/month and persists your scripts across sessions.
5. Run:

   ```bash
   az account show --output table
   ```

   ✅ You should see your subscription name + ID. Save the ID — you'll reference it.

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

Steps 13–16 (Rosetta architecture, AGW + WAF, Oracle, WOD) are easier when you can *look at* the production-like resources. You don't need to change anything — just read.

Email your team lead or DIA Core Support requesting:

> **Reader** role on the `dsr-anl-dev` and `dsr-anl-uat` subscriptions for the duration of the training (~12 weeks).

If access doesn't arrive in time, those labs include screenshots and KQL/CLI snippets you can run against your own training resources instead.

---

## 6. Create your "labs" resource group

Every lab will deploy into a single shared RG so cleanup is one command. Run once:

```bash
az group create \
  --name rg-labs-foundations-<your-initials> \
  --location australiaeast \
  --tags purpose=dsr-training app_name=training env=lab owner=<your-email>
```

Replace `<your-initials>` (e.g. `sq` for Shanshan Qu) and `<your-email>`.

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

Before you start Step 01, confirm:

- [ ] You have a personal training Azure subscription (not production)
- [ ] You can sign in to [portal.azure.com](https://portal.azure.com)
- [ ] You can open Cloud Shell and `az account show` works
- [ ] You have a Microsoft Learn account (work email)
- [ ] You've requested Reader on DSR DEV/UAT (acknowledgement is enough — actual grant can come later)
- [ ] Your subscription is tagged `purpose=dsr-training`
- [ ] You've created `rg-labs-foundations-<your-initials>`

---

## 💰 Total cost expectation across all 24 labs

- **Free trial / DIA sandbox / MPN credits:** $0 out-of-pocket (well within NZD $300 trial credit).
- **Pay-as-you-go:** ~NZD $20–30 if you tear down between labs.
- The single most expensive lab is **Step 09 (Azure Files NFS)** at ~NZD $3 if left running >2 hours. Tear down promptly.

Each lab page has its own cost note + a **Cleanup** command at the end — always run it.

---

➡️ **Next:** [Step 01 — Azure foundations](step-01-azure-foundations.md)
