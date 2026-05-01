# Step 02 — Azure Portal foundations

_The "I can find anything in the portal" lab._ 🧭

> [!NOTE]
> **Duration:** 60 minutes
> **Lab cost:** $0 (read-only navigation)
> **Pairs with:** Module 2 of the training plan

---

## 📖 Session overview

This session is dedicated to **navigating the Azure portal fluently**. The portal is your daily window into the DSR environment — being slow in the portal slows down every other operation. This lab teaches search, filtering, JSON view, Resource Graph queries, Cloud Shell, and the keyboard shortcuts that make portal work feel less like wading through a swamp.

It's a light hands-on session — no resources are deployed, so it's free to repeat as many times as you like.

## 🎯 What you'll learn

- How to navigate the portal efficiently — search, pinning, dashboards
- How to read JSON view on any resource (every resource has one)
- How to write **Resource Graph** queries to find resources across all subscriptions
- How to use **Cloud Shell** for quick CLI without leaving the browser
- How to set up portal favourites and keyboard shortcuts

## 📚 Before this session — MS Learn pre-work

| Module | Time | Why this matters |
|---|---|---|
| [Configure Azure resources with the Azure portal](https://learn.microsoft.com/training/modules/configure-azure-resources-portal/) | 30 min | The standard portal-foundations module |
| [Query the Azure Resource Graph](https://learn.microsoft.com/training/modules/intro-to-governance/) (Resource Graph section) | 20 min | The query tool that scales across subscriptions |

## 🔤 Acronyms used

- **KQL** = Kusto Query Language — used in Resource Graph and Log Analytics
- **MFA** = Multi-Factor Authentication

## ⏱️ EDE accounting

- Trainee self-paced: 60 min
- Instructor-led delivery: 1h
- Prep work: 50 min
- Q&A: 20 min
- **Total EDE per trainee: ~3h**

## 💰 Cost note

$0. No resources deployed.

---

## 🧱 Foundational primer

The Azure portal has 5 areas you'll use daily:

1. **Top search bar** — type any resource name, RG, service, or keyword. Use this 80% of the time.
2. **Left navigation** — pinned services. Customise it.
3. **Resource blade** — the right-hand panel that opens when you click a resource. Has tabs (Overview, Activity log, Tags, IAM, etc.) along its left edge.
4. **Cloud Shell** — `>_` icon top-right. Browser-based Bash/PowerShell.
5. **Dashboards** — pinned tiles on a custom home page. Useful for ops views.

---

## ⌨️ Activity 1 — Master the search bar

1. Sign in to [portal.azure.com](https://portal.azure.com).
2. Click the search bar (or press `/`).
3. Type the name of any resource, e.g. `rg-training` — see resources, services, marketplace items, and documentation matches.
4. Use **filter chips** at the top of the results (Resources / Services / Marketplace / Documentation).

> [!TIP]
> Type `_` followed by anything to search Activity Log entries. Type a UUID to jump to that exact resource.

## ⌨️ Activity 2 — Pin and customise the left nav

1. Click the **hamburger menu** (top left).
2. Click any service to add to the left nav, e.g. **Storage accounts**, **Resource groups**, **Subscriptions**, **All resources**, **Cost Management**.
3. Drag to reorder.

✅ Your most-used services are now one click away.

## ⌨️ Activity 3 — Resource blade tour

1. Search for any storage account (or use the one you created in Module 1, then re-create if you torn it down).
2. Click into it.
3. Walk through these tabs in the left edge of the blade:
   - **Overview** — at-a-glance state
   - **Activity log** — who did what and when (audit trail)
   - **Access control (IAM)** — role assignments
   - **Tags** — DIA's tagging
   - **Diagnose and solve problems** — guided troubleshooting
   - **Properties** / **Locks** — protection settings
   - **Export template** — get the ARM/Bicep code for this resource
4. Click **JSON View** (top right of Overview). This is the source of truth.

## ⌨️ Activity 4 — Resource Graph: find resources across subscriptions

Resource Graph is a free service that lets you query every resource in every subscription you have access to, in seconds.

1. Search **Resource Graph Explorer** in the top bar.
2. Run this query:

   ```kusto
   Resources
   | where type == "microsoft.storage/storageaccounts"
   | project name, resourceGroup, location, sku.name, tags
   | order by name asc
   ```

3. Click **Run query**.

✅ You see every storage account you have access to. In production, this is how you'd inventory the 4 DSR storage accounts in one query.

Try this one — find resources missing the `app_name` tag:

```kusto
Resources
| where tags['app_name'] == ""
| project name, type, resourceGroup
```

## ⌨️ Activity 5 — Cloud Shell

1. Click the `>_` icon (top right).
2. Choose **Bash**.
3. Run:

   ```bash
   az resource list \
     --query "[?type=='Microsoft.Storage/storageAccounts']" \
     --output table
   ```

4. Try the same with PowerShell — top dropdown → switch to PowerShell.

> [!TIP]
> Cloud Shell has 5GB of free home directory storage. Drop scripts in `~/clouddrive/` and they persist between sessions.

## ⌨️ Activity 6 — Pin a tile to a dashboard

1. Search **All resources** in the top bar.
2. Right-click any tile or chart → **Pin to dashboard** → choose **My dashboard**.
3. Top bar → **Dashboard** → see your pinned items.

You'll use this in Module 24 (Healthy Report) to build the Rosetta Operations dashboard.

## ⌨️ Activity 7 — Keyboard shortcuts

| Shortcut | What it does |
|---|---|
| `G+/` | Open search |
| `G+N` | Notifications |
| `G+H` | Home |
| `G+D` | Dashboard |
| `G+B` | Cloud Shell |
| `Esc` | Close current blade |

Practice opening a resource and pressing `Esc` to close.

---

## 🦾 Now your turn!

Write a Resource Graph query that returns:

- All storage accounts in any subscription you have access to
- Showing name, region, redundancy SKU, and the `env` tag
- Sorted by environment, then name

Save it as a **Saved query** in Resource Graph Explorer so you can re-run it weekly.

Hint:

```kusto
Resources
| where type == "microsoft.storage/storageaccounts"
| project name, location, sku=sku.name, env=tags['env']
| order by env asc, name asc
```

---

## ✅ Success checklist

- [ ] You can find any resource within 5 seconds via search
- [ ] You've customised your left nav with your most-used services
- [ ] You've used JSON View on at least one resource
- [ ] You've run at least 2 Resource Graph queries
- [ ] You've used Cloud Shell to list resources
- [ ] You've pinned at least one tile to a dashboard

---

## 💰 Cost note

$0. Browsing the portal is free.

---

➡️ **Next:** [Step 03 — Identity & access for the operator](step-03-identity-access.md)
