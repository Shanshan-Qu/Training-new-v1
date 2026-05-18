# Step 04 — Networking primer (read-only view)

_The "I'm not configuring this, I just need to read it" lab._ 🌐 Builds enough networking literacy to troubleshoot Rosetta connectivity issues without ever needing Owner on a VNet — VNet, subnet, NSG, Private Endpoint, Private DNS, Hub-Spoke, ExpressRoute.

> [!NOTE]
> **Trainee duration:** 90 minutes
> **Lab cost:** under NZD $1 — small VNet, NSG, and a single private endpoint. All free or near-free.
> **Prerequisites:** Steps 00–02 complete.
> **Pairs with:** Module 1 of the DIA training plan (Foundations) — addresses the "Guardrails & Governance" requirement by showing where network controls live.

---

## 📖 Session overview

In DSR, networking is owned by DIA Cloud Networking — your team won't configure VNets, NSGs, or Private Endpoints. But when Rosetta can't reach Oracle, or when a storage account refuses a connection, you need to be able to *read* the network and tell whether the issue is with the network or with the application. This lab gives you that literacy.

**What you'll learn**
- The shape of an Azure network: **VNet → Subnet → NIC**.
- How **NSGs** filter traffic and how to read NSG rules.
- What a **Private Endpoint** is and how it's used in DSR for storage access.
- How **Private DNS Zones** make storage URLs resolve to private IPs.
- The DSR **Hub-Spoke** topology with **ExpressRoute** to DIA on-premises.
- How to troubleshoot connectivity using **Network Watcher** (read-only).

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **VNet** | Virtual Network. The Azure equivalent of a private network segment. |
| **Subnet** | A subdivision of a VNet. Each subnet has its own IP range. |
| **NIC** | Network Interface Card. Every VM has one or more NICs that attach it to a subnet. |
| **NSG** | Network Security Group. A stateful firewall attached to a subnet or NIC. |
| **Private Endpoint (PE)** | A private IP in your VNet that talks to an Azure PaaS service (e.g. a storage account) — keeps traffic off the public internet. |
| **Private DNS Zone** | DNS records that resolve PaaS service hostnames (e.g. `stalnznznblobprdrosi01.blob.core.windows.net`) to the private IP of a Private Endpoint. |
| **Hub-Spoke** | A topology where shared services (firewall, ExpressRoute, DNS) live in a Hub VNet, and apps live in Spoke VNets that peer to the Hub. |
| **ExpressRoute** | A dedicated, private circuit between DIA's on-premises network and Azure (no internet involved). |
| **Azure Firewall** | A managed network firewall for the Hub VNet. DSR uses it for outbound DNAT/SNAT. |
| **Network Watcher** | A diagnostics service for Azure networking — connection troubleshooting, NSG flow logs, etc. |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ALNZ |
|---|---|
| [Introduction to Azure Virtual Networks](https://learn.microsoft.com/training/modules/introduction-to-azure-virtual-networks/) | Foundational — every other module assumes this. |
| [Secure your virtual network with Azure Firewall and NSGs](https://learn.microsoft.com/training/modules/introduction-azure-firewall/) | DSR uses both. |
| [Introduction to Azure Private Link](https://learn.microsoft.com/training/modules/introduction-azure-private-link/) | The mechanism behind every storage and Key Vault private endpoint in DSR. |
| [Hub-and-spoke network topology](https://learn.microsoft.com/azure/architecture/networking/architecture/hub-spoke) | DSR's reference architecture. |

About **2 hours** of optional pre-reading.

## 🧱 Foundational primer

- **VNets are isolated by default.** Two VNets in the same subscription cannot talk to each other unless you peer them.
- **NSG rules are processed in priority order** (lowest first), and the first match wins. Default rules at the bottom allow VNet-internal and deny inbound from internet.
- **Private Endpoints are the security pattern** for PaaS in DSR. The storage account has its public endpoint blocked; only Private Endpoints from approved subnets can reach it.
- **DNS is the most common cause of "it doesn't work".** Private DNS zones must be linked to the right VNet for hostnames to resolve to private IPs. Without it, name resolution still goes to the public IP — which is firewalled — and the request fails.
- **Hub-Spoke separates concerns.** The Hub holds shared firewall, gateway, DNS. Spokes hold workloads. ALNZ is a spoke.
- **You cannot change the network in DSR.** Period. But you can read every rule, route, and connection — and that's how you triage.
- **Read-only ≠ powerless.** A Network Watcher Connection Troubleshooter test costs nothing and can prove a problem is/isn't network-related.

## ⌨️ Activity 1 — Create a tiny lab VNet

1. Search → **Virtual networks → + Create**.
2. Subscription: the shared training sandbox. RG: `rg-labs-foundations-<your-initials>`. Name: `vnet-labs-net`. Region: Australia East.
3. **IP Addresses** tab. Address space: `10.99.0.0/16`. Replace the default subnet with two subnets:
   - `snet-app` — `10.99.1.0/24`
   - `snet-pe`  — `10.99.2.0/24` (this is where the private endpoint will go)
4. **Review + create → Create**. Wait ~30 sec.
5. Tag the VNet with `purpose=dsr-training`, `owner=<your email>`.

## ⌨️ Activity 2 — Read an NSG

1. Search → **Network security groups → + Create**.
2. Same RG, region. Name: `nsg-snet-app-labs`. Create.
3. Open it → **Inbound security rules**. You'll see four default rules:

| Priority | Name | Source | Destination | Action |
|---|---|---|---|---|
| 65000 | AllowVnetInBound | VirtualNetwork | VirtualNetwork | Allow |
| 65001 | AllowAzureLoadBalancerInBound | AzureLoadBalancer | Any | Allow |
| 65500 | DenyAllInBound | Any | Any | **Deny** |

4. Add a rule: **+ Add → Source = Any, Source port = *, Destination = Any, Destination port = 443, Protocol = TCP, Action = Allow, Priority = 100**. Name: `Allow-HTTPS-In`. Save.
5. Now re-read the rules in priority order. Notice that your rule (priority 100) wins over the default DenyAll (priority 65500) for HTTPS.

> [!IMPORTANT]
> NSG rules are stateful. Once an inbound flow is allowed, the response is automatically allowed back out — you don't need a matching outbound rule.

6. Associate the NSG: open it → **Subnets → + Associate** → choose `vnet-labs-net` and `snet-app`.

## ⌨️ Activity 3 — Read a real DSR-style NSG rule

In production, `nsg-snet-ora-prd` has this inbound rule allowing the Rosetta app subnet to reach the Oracle subnet on port 1521:

```
Priority: 100
Name: Allow-App-to-Oracle-1521
Source: snet-app-prd (10.x.y.z/24)
Source port: *
Destination: snet-ora-prd (10.x.y.z/24)
Destination port: 1521
Protocol: TCP
Action: Allow
```

When Rosetta can't connect to Oracle, your first read of the network is exactly this rule — confirm it's there, allowing 1521 from the right source. If it's not, escalate to DIA Cloud Networking.

## ⌨️ Activity 4 — Create a Private Endpoint to your storage account

1. Open a test storage account in your `rg-labs-foundations-<your-initials>` (create a small one if you don't have one yet) → **Networking → Private endpoint connections → + Private endpoint**.
2. Subscription / RG / region: sandbox / labs RG / Australia East. Name: `pe-storage-labs`.
3. Resource: storage account, target sub-resource: **blob**.
4. Virtual network: `vnet-labs-net`. Subnet: `snet-pe`.
5. **Integrate with Private DNS zone**: Yes (this auto-creates a private DNS zone `privatelink.blob.core.windows.net`).
6. Review + create → Create. Wait ~2 min.

7. Once created, open the Private Endpoint resource → **Overview**. Note the private IP (something like `10.99.2.4`).
8. Open the Private DNS Zone (find it under Resource groups → look for `privatelink.blob.core.windows.net`) → see the auto-created A record pointing your storage account name to that private IP.

This is how DSR's Rosetta VMs reach `stalnznznblobprdrosi01` — through a private endpoint, with private DNS resolving the public hostname to a private IP.

## ⌨️ Activity 5 — Lock down the storage account public endpoint

Now that we have a private endpoint, we can disable public access.

1. Storage account → **Networking → Public network access** → **Disabled** → Save.
2. Try to access the storage account from your browser via its public URL (`https://<your-account>.blob.core.windows.net`). You should get an error.
3. Now, if you had a VM in `snet-app` and DNS resolved through the linked Private DNS Zone, it could still access the storage. (We won't deploy a VM here for cost reasons; the read-only takeaway is the pattern.)

## ⌨️ Activity 6 — Network Watcher: Connection Troubleshoot (read-only)

1. Search → **Network Watcher → Connection troubleshoot**.
2. Source: pick any VM (you may not have one in the sandbox yet — if not, run this against the real DSR DEV/UAT estate where you have Reader on a VM).
3. Destination: `stalnznznblobprdrosi01.blob.core.windows.net` (or your test storage account's URL), port 443.
4. **Check** — Network Watcher will show the path, latency, and whether any NSG / firewall rule blocks it.

In DSR you'll have read access to Network Watcher in your subscriptions. Use it to **prove** a problem is or isn't a network problem before calling Cloud Networking.

> [!TIP]
> Network Watcher's **NSG diagnostic** is also read-only and tells you exactly which NSG rule allowed/denied a specific flow. Brilliant for triage.

## 🌐 The DSR network in one picture

```
              ┌─────────────────────────────────┐
              │  DIA on-premises (Wellington)   │
              └────────────────┬────────────────┘
                               │ ExpressRoute (private circuit)
                               ▼
         ┌─────────────────────────────────────────────┐
         │  HUB VNet  (NZ North)                       │
         │  · Azure Firewall                            │
         │  · ExpressRoute Gateway                      │
         │  · Private DNS resolver                      │
         └────────────┬───────────────────┬─────────────┘
                      │ VNet peering      │ VNet peering
                      ▼                   ▼
        ┌────────────────────┐   ┌────────────────────┐
        │ ALNZ Spoke (PRD)    │   │ ALNZ Spoke (UAT/DEV)│
        │  snet-app-prd      │   │  snet-app-uat etc. │
        │  snet-ora-prd      │   │                    │
        │  snet-pe-prd       │   │                    │
        │   ↓ Private Endpts │   │                    │
        │  Storage / KV /    │   │                    │
        │  Other PaaS        │   │                    │
        └────────────────────┘   └────────────────────┘
```

## 🦾 Now your turn!

1. Look at the **Effective security rules** on `snet-app` (your subnet → Effective security rules). What are the priority numbers in order?
2. Find the Private DNS Zone created in Activity 4. What other VNets is it linked to? (Should be just `vnet-labs-net`.)
3. In Resource Graph, write a query that lists every Private Endpoint in your subscription:
    ```kql
    resources
    | where type == "microsoft.network/privateendpoints"
    | project name, resourceGroup, location, properties.privateLinkServiceConnections
    ```
4. From the storage account JSON view (Step 01), find the `properties.networkAcls` block. What does it say now that public access is disabled?

## ✅ Success checklist

- [ ] You can sketch the four-layer model: VNet → Subnet → NIC → Resource.
- [ ] You can read an NSG rule and predict whether traffic will be allowed.
- [ ] You understand the Private Endpoint + Private DNS pattern.
- [ ] You can describe the DSR Hub-Spoke + ExpressRoute topology.
- [ ] You know that you read but never write the network in DSR — and which team owns it.
- [ ] You've run at least one Network Watcher diagnostic.

## 📚 Self-serve refresher

- [Network Watcher overview](https://learn.microsoft.com/azure/network-watcher/network-watcher-overview) — your daily read-only diagnostic kit.
- [Private Endpoint troubleshooting](https://learn.microsoft.com/azure/private-link/troubleshoot-private-endpoint-connectivity) — when name resolution looks wrong.

## 💰 Cost note

- VNet: free.
- NSG: free.
- Private Endpoint: ~NZD $0.01/hour ≈ $7/month if left running. **Delete after lab.**
- Private DNS Zone: ~NZD $0.50/month + $0.40 per million queries.
- Network Watcher: free for ad-hoc diagnostics.

**Cleanup:** delete the Private Endpoint + Private DNS Zone first (they cost the most), then the rest can stay tagged in your RG until the end of the series. Or just delete the entire `rg-labs-foundations-*` if you're done.

```bash
# Cloud Shell — delete the PE quickly
az network private-endpoint delete -g rg-labs-foundations-<your-initials> -n pe-storage-labs
```

---

⬅️ **Previous:** [Step 03 — Guardrails, governance & audit](step-03-governance-guardrails.md)
➡️ **Next:** [Step 05 — Storage accounts & tier management](step-05-storage-accounts-and-tiers.md) (Phase 2 begins)
