# Step 05 — Networking primer (read-only view)

_The "I can read the network without breaking it" lab._ 🌐

> [!NOTE]
> **Duration:** 90 minutes
> **Lab cost:** < NZD $1 (a small VNet + Private Endpoint, torn down at end)
> **Pairs with:** Module 5 of the training plan
> **Networking is owned by DIA Cloud Networking** — this lab is read-only and diagnostic only.

---

## 📖 Session overview

The Digital Preservation Team doesn't configure DSR networking — that's the Cloud Networking team's job. But you do need to *read* the network when troubleshooting Rosetta connectivity (e.g. "the deposit VM can't reach the database"). This session teaches you the network shapes you'll encounter in DSR (VNets, subnets, NSGs, private endpoints, hub-and-spoke, ExpressRoute) and how to interpret them — without giving you the right to change them.

## 🎯 What you'll learn

- What a **VNet** is and how subnets divide it
- How **NSGs** filter traffic and how to read NSG rules
- What **Private Endpoints** and **Private DNS Zones** are and why DSR uses them
- The **Hub-and-Spoke** topology DSR uses
- How **ExpressRoute** connects DIA on-premises to Azure
- How to use **Network Watcher** for read-only diagnostics
- The escalation path when you suspect a network issue

## 📚 Before this session — MS Learn pre-work

| Module | Time | Why this matters |
|---|---|---|
| [Describe Azure networking](https://learn.microsoft.com/training/modules/describe-azure-networking-services/) | 30 min | VNets, subnets, public/private endpoints |
| [Configure Azure virtual networks](https://learn.microsoft.com/training/modules/configure-virtual-networks/) | 45 min | Hands-on with NSGs, peering |
| [Introduction to Azure Private Endpoints](https://learn.microsoft.com/training/modules/introduction-azure-private-link/) | 30 min | Why DSR uses Private Endpoints everywhere |

## 🔤 Acronyms used

- **DNS** = Domain Name System
- **ER** = ExpressRoute. A private fibre link from DIA to Azure.
- **NSG** = Network Security Group. Stateful firewall rules attached to a subnet or NIC.
- **PE** = Private Endpoint. A network interface in your VNet that gives a service a private IP.
- **VNet** = Virtual Network

## ⏱️ EDE accounting

- Trainee self-paced: 90 min
- Instructor-led delivery: 1.5h
- Prep work: 1.75h
- Q&A: 30 min
- **Total EDE per trainee: ~5h**

## 💰 Cost note

- Small VNet: free
- Private Endpoint: ~NZD $0.30/day
- Test storage account: ~NZD $0.03/day
- Tear down within the lab — total < NZD $1.

---

## 🧱 Foundational primer

### Hub-and-Spoke (DSR uses this)

```
┌─────────────────────┐
│  Hub VNet (shared)  │ ← ExpressRoute gateway, Azure Firewall, shared DNS
└──────────┬──────────┘
           │ peering
   ┌───────┴───────┐
   │               │
┌──┴───────┐  ┌────┴─────┐
│ Spoke A  │  │ Spoke B  │  ← ANL workloads (one spoke per env)
│ (DEV)    │  │ (UAT)    │
└──────────┘  └──────────┘
```

DSR has spokes per env: `vnet-anl-dev`, `vnet-anl-uat`, `vnet-anl-prd`. Each is peered to the central hub.

### Subnets (DSR has many — read the design doc table)

| Subnet | What lives there |
|---|---|
| `snet-app-prd` | Rosetta application VMs |
| `snet-ora-prd` | Oracle DB VMs (`rosidbs01/02`) |
| `snet-pe-prd` | Private Endpoints (storage accounts, Key Vault) |
| `snet-agw-prd` | Application Gateway |
| `snet-bastion` | Azure Bastion (jump host) |

### NSG rules (read-only — don't modify)

DSR's `nsg-snet-ora-prd` allows TCP 1521 from `snet-app-prd` only — that's how Rosetta app VMs reach Oracle. If you see this, it's intentional.

### Private Endpoints

Instead of a storage account being on the public internet, a Private Endpoint puts it inside the VNet at a private IP. The DNS zone `privatelink.blob.core.windows.net` rewrites the public hostname to the private IP, so applications don't need to change.

---

## ⌨️ Activity 1 — Inspect the DSR VNet (read-only)

If you have Reader on DSR DEV:

1. Portal → search **Virtual networks** → `vnet-anl-dev`.
2. Note: Address space, Subnets list, Peerings (peers to the hub).
3. Click any subnet → see NSG attached, route table, service endpoints, delegations.

## ⌨️ Activity 2 — Read NSG rules

1. Portal → search **Network security groups** → `nsg-snet-ora-prd` (or DEV equivalent).
2. **Inbound security rules** — see allow/deny rules.
3. Look for the rule allowing TCP 1521 from `snet-app-prd`.
4. Click any rule → see priority, source, destination, action.

## ⌨️ Activity 3 — Trace a network path with Network Watcher

Network Watcher is free and read-only diagnostics.

1. Portal → search **Network Watcher**.
2. Left blade → **Connection troubleshoot**.
3. Source: pick any DSR VM. Destination: pick a storage account FQDN like `stanlnznblobprdrosi01.blob.core.windows.net`.
4. Run the test. ✅ You'll see hop-by-hop reachability.

> [!TIP]
> If a Rosetta VM can't reach storage, this is the first place you look. Output tells you exactly which hop blocks the traffic.

## ⌨️ Activity 4 — Inspect a Private Endpoint

1. Portal → search **Private endpoints**.
2. Pick one in DSR DEV (e.g. `pe-stanlnznfileprdrosi01`).
3. Note: Private IP, Network interface, DNS configuration, Connection state.
4. Click **DNS configuration** → see the FQDN being mapped to a private IP.

## ⌨️ Activity 5 — Hands-on: deploy a tiny VNet + PE in your own sub

```bash
# Resource group
az group create -n rg-training-net-<your-initials> -l australiaeast \
  --tags purpose=dsr-training

# VNet with two subnets
az network vnet create \
  -g rg-training-net-<your-initials> \
  -n vnet-training \
  --address-prefix 10.50.0.0/16 \
  --subnet-name snet-app --subnet-prefix 10.50.1.0/24

az network vnet subnet create \
  -g rg-training-net-<your-initials> \
  --vnet-name vnet-training \
  -n snet-pe \
  --address-prefix 10.50.2.0/24 \
  --disable-private-endpoint-network-policies true

# Storage account to put a PE on
SA=sttraining<your-initials>$RANDOM
az storage account create \
  -n $SA -g rg-training-net-<your-initials> \
  -l australiaeast --sku Standard_LRS --kind StorageV2

# Create Private Endpoint
az network private-endpoint create \
  -g rg-training-net-<your-initials> \
  -n pe-$SA \
  --vnet-name vnet-training --subnet snet-pe \
  --private-connection-resource-id $(az storage account show -n $SA -g rg-training-net-<your-initials> --query id -o tsv) \
  --group-id blob \
  --connection-name myconn
```

Inspect the PE in the portal — you'll see it has a private IP from `10.50.2.0/24`.

## ⌨️ Activity 6 — View effective routes (advanced)

1. Portal → any VM → **Networking** → **Effective routes**.
2. See the actual routing table the OS uses.
3. Identify the route to `0.0.0.0/0` — in DSR, this points to the Azure Firewall in the hub.

## ⌨️ Activity 7 — Tear down

```bash
az group delete -n rg-training-net-<your-initials> --yes --no-wait
```

---

## 🦾 Now your turn!

Use Network Watcher → **NSG diagnostics** to test whether a hypothetical packet from `snet-app-prd` to `snet-ora-prd` on TCP 1521 would be allowed. Document the rule that allows it.

If you don't have access to DSR DEV, simulate it on your training VNet by creating two subnets and an NSG with one rule.

---

## ✅ Success checklist

- [ ] You can read the structure of a VNet (subnets, peerings)
- [ ] You can find the NSG rule allowing Rosetta app → Oracle
- [ ] You've used Network Watcher to trace a connection
- [ ] You've inspected a Private Endpoint
- [ ] You've deployed your own VNet + PE
- [ ] You understand: networking changes go via DIA Cloud Networking, not you

---

## 🚨 Escalation path

If you suspect a network issue causing Rosetta failure:

1. Capture the Network Watcher Connection Troubleshoot output (screenshot).
2. Note the NSG rule that's blocking (if any).
3. Open a ticket with **DIA Cloud Networking**, attaching the diagnostics.
4. Do **not** modify NSG rules yourself.

---

## 💰 Cost note

< NZD $1 if torn down promptly.

---

➡️ **Next:** [Step 06 — Storage accounts deep-dive](step-06-storage-accounts.md)
