# Step 00 — Environment readiness (pre-flight)

_Get your training environment set up before any hands-on session._ ✈️

> [!NOTE]
> **Duration:** 45 minutes
> **Lab cost:** $0 (only sign-ups and account verification)
> **Pairs with:** Module 0 of the training plan

---

## 📖 Session overview

This pre-flight session gets your personal Azure training environment ready so you can start the hands-on labs without delay. It walks you through getting an Azure subscription you can experiment in safely, installing the tools you'll use across the series, and confirming you have the right access to read the production DSR landing zone.

You don't need any Azure experience to complete this session — every step has a screenshot or a copy-paste command.

## 🎯 What you'll learn

- How to obtain a personal training Azure subscription (free trial or DIA-issued sandbox)
- How to sign in to the Azure portal and Cloud Shell
- How to install Azure CLI on your laptop (optional — Cloud Shell works in the browser)
- How to sign up for a Microsoft Learn account so pre-work links work
- How to confirm Reader access on the DSR DEV / UAT landing zone for read-only walkthroughs
- How to verify everything works using a one-line CLI command

## 📚 Before this session — MS Learn pre-work

None — this **is** the pre-work for everything else.

## 🔤 Acronyms used

- **CLI** = Command-Line Interface (you type commands instead of clicking)
- **CSP** = Cloud Solution Provider (the licensing model DIA uses for Azure)
- **NZD** = New Zealand Dollar
- **RBAC** = Role-Based Access Control (who can do what in Azure)

## ⏱️ EDE accounting

- Trainee self-paced: 45 min
- Instructor-led delivery: 30 min (group walk-through)
- Prep work: 0
- Q&A: 15 min
- **Total EDE per trainee: ~1.5h**

## 💰 Cost note

- $0 for setup itself.
- A free trial Azure subscription includes ~NZD $300 credit valid for 30 days — more than enough for this entire training series if you tear down between labs.

---

## ⌨️ Activity 1 — Get a training Azure subscription

You have three options. Pick one:

| Option | Best for | How |
|---|---|---|
| **A. Azure free trial** | Most trainees | Sign up at [azure.microsoft.com/free](https://azure.microsoft.com/free). Requires a phone number and credit card for verification (no charge unless you exceed the credit). Includes NZD ~$300 free credit for 30 days. |
| **B. DIA-issued sandbox** | If DIA has provisioned one for you | Ask your team lead. Use the credentials they provide. |
| **C. Visual Studio / MPN credits** | If you have a Visual Studio subscription | Already includes monthly Azure credit — sign in at [portal.azure.com](https://portal.azure.com) with your VS account. |

> [!IMPORTANT]
> Do **not** use the production DSR subscription for hands-on labs. Use a dedicated training subscription so cleanup is clean and there's no risk to production.

## ⌨️ Activity 2 — Sign in to the Azure portal

1. Go to [portal.azure.com](https://portal.azure.com).
2. Sign in with the account from Activity 1.
3. In the top right, click your profile picture and confirm:
   - The directory name (top of the menu).
   - The subscription name (Switch Directory if needed).

✅ You should see "Welcome to Microsoft Azure" and your subscription listed.

## ⌨️ Activity 3 — Open Cloud Shell

Cloud Shell is a free Linux shell in your browser — no install needed.

1. In the top bar, click the `>_` icon.
2. Choose **Bash** when prompted.
3. If asked to create storage for Cloud Shell, accept the defaults (it's free for the storage required).
4. Run:

   ```bash
   az account show --output table
   ```

✅ You should see your subscription name and ID.

## ⌨️ Activity 4 — (Optional) Install Azure CLI on your laptop

Skip this if you're happy using Cloud Shell.

- **macOS:** `brew install azure-cli`
- **Windows:** download from [aka.ms/installazurecliwindows](https://aka.ms/installazurecliwindows)
- **Linux (RHEL):** `sudo dnf install azure-cli`

After install, sign in:

```bash
az login
```

✅ A browser opens; sign in with your training account; the terminal shows your subscription.

## ⌨️ Activity 5 — Create a Microsoft Learn account

1. Go to [learn.microsoft.com](https://learn.microsoft.com).
2. Click **Sign in** in the top right.
3. Use your **work email** (not the training Azure account — these are separate).
4. Pick a display name and confirm.

✅ Pre-work links from later modules will now track your progress.

## ⌨️ Activity 6 — Confirm Reader access on DSR DEV / UAT

For modules 12–15 you'll need to *look at* the production-like Rosetta and WOD resources. You don't need to change anything — just read.

1. Email your team lead or DIA Core Support requesting:
   > **Reader** role on the `dsr-anl-dev` and `dsr-anl-uat` subscriptions for the duration of training.
2. Once granted, in [portal.azure.com](https://portal.azure.com) → top bar → switch to the DIA tenant → confirm you can see the DSR resource groups.

> [!TIP]
> If you don't have access yet by Module 12, you can still do those modules — you'll just use screenshots from training material instead of the live environment.

## ⌨️ Activity 7 — Final verification

In Cloud Shell, run:

```bash
az account list --output table
az group list --output table
```

✅ The first command lists your training subscription. The second shows resource groups (probably empty in your training sub — that's fine).

---

## 🦾 Now your turn!

Tag your training subscription so it's easy to identify later:

```bash
az tag create --resource-id /subscriptions/<your-sub-id> \
  --tags purpose=dsr-training owner=<your-email>
```

Replace `<your-sub-id>` with your training subscription ID (from `az account show`).

---

## ✅ Success checklist

- [ ] You have a training Azure subscription (not production)
- [ ] You can sign in to [portal.azure.com](https://portal.azure.com)
- [ ] You can open Cloud Shell and `az account show` works
- [ ] You have a Microsoft Learn account
- [ ] You've requested Reader on DSR DEV / UAT (acknowledgement is enough — actual grant can come later)
- [ ] Your subscription is tagged `purpose=dsr-training`

---

## 💰 Cost note

Setup itself is free. Your free trial credit is untouched — you'll start using it from Module 1 onwards.

---

➡️ **Next:** [Step 01 — Azure foundations](step-01-azure-foundations.md)
