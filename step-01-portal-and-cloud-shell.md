# Step 01 — Portal & Cloud Shell tour

_The "find anything in 30 seconds" lab._ 🧭 Builds Portal fluency: search, pinning, JSON view, Resource Graph queries, and Cloud Shell — the everyday tools you'll use to operate the DSR landing zone.

> [!NOTE]
> **Trainee duration:** 60 minutes
> **Lab cost:** $0 — Cloud Shell uses a small free file share, everything else is metadata.
> **Prerequisites:** [Step 00 — Lab environment setup](step-00-lab-environment-setup.md) complete; your `rg-labs-foundations-<your-initials>` resource group exists in your sandbox subscription.
> **Pairs with:** Module 1 of the DIA training plan (Foundations).

---

## 📖 Session overview

The Azure portal is large. Without the right shortcuts, finding a specific resource in DSR (which has hundreds) is painful. This session shows you how to navigate efficiently, read the JSON behind any resource, run quick inventory queries with Resource Graph, and use Cloud Shell — Azure's built-in browser terminal.

**What you'll learn**
- How to **search, pin, and favourite** resources you use often.
- How to read the **JSON view** of any resource (the source of truth behind the GUI).
- How to write a **Resource Graph query** to find anything across all subscriptions in seconds.
- How to use **Cloud Shell** — a free, browser-based Azure CLI / PowerShell terminal that requires nothing on your laptop.
- The right tool for the right job: portal vs CLI vs Resource Graph.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **Portal** | The web interface at portal.azure.com. |
| **Blade** | A pane that slides in from the right when you click something — Azure's name for a screen. |
| **Cloud Shell** | A free Linux or PowerShell terminal that runs in your browser, pre-authenticated. |
| **Azure CLI (`az`)** | The command-line tool for Azure. Works in Cloud Shell or installed locally. |
| **Resource Graph** | A query engine that searches all your resources at once using KQL (Kusto Query Language). |
| **KQL** | Kusto Query Language — Azure's read-only query language for logs, metrics, and Resource Graph. Looks a bit like SQL piped together. |
| **JSON view** | The raw definition of a resource. Useful for debugging or copying settings. |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Tour the Azure portal](https://learn.microsoft.com/training/modules/tour-azure-portal/) | The team's daily home base. |
| [Introduction to Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) | No-install access for ad-hoc commands. |
| [Introduction to Azure Resource Graph](https://learn.microsoft.com/azure/governance/resource-graph/overview) | The fastest way to find things across DSR's three subs. |
| [Get started with Azure CLI](https://learn.microsoft.com/cli/azure/get-started-with-azure-cli) | The CLI is what scripts and runbooks use under the hood. |

About **1.5 hours** of optional pre-reading.

## 🧱 Foundational primer

- **Three ways to do anything:** Portal (clicks), Azure CLI (commands), Resource Graph (queries). Different tools for different jobs.
- **Pinning saves clicks.** The portal home page can show your most-used resources as tiles.
- **JSON view is read-only by default.** You can copy values out of it but cannot edit in place — you'd use ARM templates or the CLI for that.
- **Cloud Shell is free**, but it does need a small Azure Files share to persist your home directory between sessions (~5 cents/month). It's automatically created the first time you launch it.
- **Resource Graph is read-only**, fast, and queries across **all subscriptions you have access to** at once. Perfect for "find every storage account in DSR with versioning disabled".

## ⌨️ Activity 1 — Master the search bar

1. Open <https://portal.azure.com>.
2. Click the search bar at the top.
3. Type `storage` — note three result categories: **Resources** (your actual storage accounts), **Services** (the storage account configuration screen), **Marketplace** (things you could create), and **Documentation**.
4. Click any storage account you can see in your sandbox subscription (create a small one in your `rg-labs-foundations-<your-initials>` if there isn't one yet). Note that you can search by full name, partial name, or even resource type.

> [!TIP]
> The search bar is the fastest way to navigate. Avoid clicking through "All services" or the left menu — search is always quicker.

## ⌨️ Activity 2 — Pin to the home page

1. Open your test storage account.
2. Click the pin icon at the top right of the **Overview** page.
3. Click the Azure logo (top-left) to go home. You should see the storage account as a tile.
4. Right-click the tile → **Resize** to small/medium/large.
5. Drag and drop tiles to reorder.

You can pin individual blades, individual resources, or saved Resource Graph queries. The home screen becomes a personal dashboard.

## ⌨️ Activity 3 — Read the JSON behind any resource

1. Open your test storage account.
2. Top-right of the Overview page → **JSON view**.
3. Read the JSON. Find:
   - The `id` (the ARM resource path)
   - The `properties.primaryEndpoints` (the actual URLs blob/file/queue/table are reachable on)
   - The `tags` you set on the RG in Step 00 (`purpose=dsr-training`, `app_name=training`, `env=lab`)
4. Click **Copy to clipboard** at the top.
5. This is the same JSON an ARM template, Bicep file, or Terraform plan would describe. It's the source of truth.

## ⌨️ Activity 4 — Your first Resource Graph query

1. Search → **Resource Graph Explorer**.
2. In the query box, paste:

```kql
resources
| where type == "microsoft.storage/storageaccounts"
| project name, location, sku.name, kind, resourceGroup, subscriptionId
```

3. Click **Run query**. You should see your test storage account.
4. Now try this — find everything tagged for the lab:

```kql
resources
| where tags["purpose"] == "dsr-training"
| project name, type, resourceGroup, tags
```

5. Try counting resources by type:

```kql
resources
| summarize count() by type
| order by count_ desc
```

> [!IMPORTANT]
> Resource Graph is the fastest way to inventory the DSR environment. In production, queries that would take 50 portal clicks return in under a second — across DEV, UAT, and PRD simultaneously.

## ⌨️ Activity 5 — Cloud Shell

1. Top of the portal → click the **`>_`** icon (next to the search bar).
2. First time only: pick **Bash** (or PowerShell, your choice). It will offer to create a small storage account for your home directory — accept.
3. After about 30 seconds, you have a Linux terminal with Azure CLI pre-installed and pre-authenticated.

Try these:

```bash
# Who am I?
az account show

# List resource groups
az group list -o table

# List storage accounts
az storage account list -o table

# Show your test storage account as JSON
az storage account show -g rg-labs-foundations-<your-initials> -n <your-storage-account>
```

4. Note the `-o table` flag — gives a tidy human-readable view. Without it you get JSON.
5. Try the same Resource Graph query from Activity 4 via CLI:

```bash
az graph query -q "resources | where type == 'microsoft.storage/storageaccounts' | project name, location, resourceGroup"
```

## ⌨️ Activity 6 — Use the right tool

You're going to be told "find every storage account in DSR that has Hot tier set as default." Pick the best tool:

| Approach | Time |
|---|---|
| Click into each storage account's Configuration blade in the portal | 20+ minutes |
| `az storage account list` and read the JSON | 2 minutes |
| Resource Graph query | 5 seconds |

The query:

```kql
resources
| where type == "microsoft.storage/storageaccounts"
| extend defaultAccessTier = tostring(properties.accessTier)
| where defaultAccessTier == "Hot"
| project name, defaultAccessTier, resourceGroup
```

## 🦾 Now your turn!

1. Pin a custom Resource Graph query: in Resource Graph Explorer, write a query that lists all your lab resources (filter by `tags.purpose == "dsr-training"`). Click **Save query** → make it private. Then **Pin to dashboard**.
2. From Cloud Shell, write a one-liner that outputs only the **names** of your lab resource groups (hint: use `--query` and `[].name`).
3. Open the JSON view of your resource group (not the storage account) and find the `properties.provisioningState` field. What value is there?

## ✅ Success checklist

- [ ] You can find any resource by name in under 5 seconds.
- [ ] At least one resource is pinned to your portal home page.
- [ ] You can read the JSON view of a resource and explain what `id`, `properties`, and `tags` mean.
- [ ] You've run at least three Resource Graph queries.
- [ ] Cloud Shell launches and `az account show` returns your tenant.
- [ ] You can articulate when to use Portal vs CLI vs Resource Graph.

## 📚 Self-serve refresher

- [Resource Graph sample queries](https://learn.microsoft.com/azure/governance/resource-graph/samples/starter) — copy-paste templates for common questions.
- [Azure CLI cheat sheet](https://learn.microsoft.com/cli/azure/) — the full command reference.

## 💰 Cost note

- Cloud Shell file share: ~NZD $0.05/month (5 GB free).
- Resource Graph queries: free.
- Portal navigation: free.

**Cleanup:** nothing to clean up beyond your `rg-labs-foundations-<your-initials>` resource group. The Cloud Shell file share lives in its own RG (`cloud-shell-storage-*`) — leave it; you'll reuse it across all later labs.

---

⬅️ **Previous:** [Step 00 — Lab environment setup](step-00-lab-environment-setup.md)
➡️ **Next:** [Step 02 — Identity & access for the operator](step-02-identity-and-access.md)
