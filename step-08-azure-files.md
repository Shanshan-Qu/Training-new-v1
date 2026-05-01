# Step 08 — Azure Files: NFS vs SMB

_The "Rosetta's hot path runs through here" lab._ 📂

> [!NOTE]
> **Duration:** 120 minutes
> **Lab cost:** < NZD $3 (Premium FileStorage is the largest single cost in the series)
> **Pairs with:** Module 8 of the training plan

---

## 📖 Session overview

Azure Files is a managed file share service. DSR uses two flavours:

- **`stanlnznfileprdrosi01`** — **Premium NFS v4.1** — Rosetta's hot path, mounted by RHEL VMs.
- **`stanlnznfileprdrosi02`** — **Standard SMB** — admin shares, less performance-critical.

You'll mount both kinds, understand the permissions models (POSIX vs Kerberos), watch performance metrics, and learn how throttling presents itself — because when Rosetta slows down, this is often where you look first.

## 🎯 What you'll learn

- The differences between **NFS v4.1** and **SMB 3.x** protocols on Azure Files
- How to **mount** each from a Linux VM
- POSIX permissions (UID/GID) for NFS vs **Kerberos** for SMB
- How to read **share-level metrics** (latency, throttling, IOPS, throughput)
- How to spot **throttling events** and what to do
- DSR-specific mount options (per § 9.12 of the design)

## 📚 Before this session — MS Learn pre-work

| Module | Time | Why this matters |
|---|---|---|
| [Introduction to Azure Files](https://learn.microsoft.com/training/modules/intro-to-azure-files/) | 30 min | NFS vs SMB, premium vs standard |
| [Configure Azure Files](https://learn.microsoft.com/training/modules/configure-azure-files-file-sync/) | 45 min | Hands-on overview |

## 🔤 Acronyms used

- **GID / UID** = Group / User Identifier (POSIX numeric IDs)
- **IOPS** = Input/Output Operations Per Second
- **NFS** = Network File System
- **SMB** = Server Message Block (Windows-native file protocol; Linux supports it)
- **SSSD** = System Security Services Daemon (Linux integration with AD)

## ⏱️ EDE accounting

- Trainee self-paced: 2h
- Instructor-led delivery: 2h
- Prep work: 1.25h
- Q&A: 30 min
- **Total EDE per trainee: ~6h**

## 💰 Cost note

- Premium FileStorage account (100 GiB minimum): ~NZD $0.50/day if left running
- Small RHEL B2s VM: ~NZD $0.20/day
- Tear down within 1–2 hours: total < NZD $3.

---

## 🧱 Foundational primer

### NFS v4.1 vs SMB 3.x on Azure Files

| | NFS v4.1 | SMB 3.x |
|---|---|---|
| Account kind | Premium FileStorage | StorageV2 (Standard or Premium) |
| Authentication | POSIX UID/GID + IP firewall | NTLM, Kerberos (AD), or shared key |
| Mounting from | Linux | Linux + Windows |
| Encryption in transit | NFS over TLS *(preview)* / IPSec | SMB 3.x channel encryption (always) |
| Used in DSR | Premium NFS, Rosetta hot path (`...rosi01`) | Standard SMB, admin shares (`...rosi02`) |

### DSR-specific mount options (per § 9.12)

For NFS:

```bash
mount -t nfs -o vers=4,minorversion=1,sec=sys,nconnect=4 \
  stanlnznfileprdrosi01.file.core.windows.net:/stanlnznfileprdrosi01/permanent /mnt/permanent
```

Notes:
- `vers=4,minorversion=1` — required for Premium NFS
- `sec=sys` — POSIX UID/GID auth (no Kerberos)
- `nconnect=4` — multiplexes I/O across 4 TCP connections (boosts throughput up to 4x)

### Throttling rules (Premium FileStorage)

Premium provisions IOPS and throughput per share, by share size:

| Share size | Baseline IOPS | Burst IOPS | Throughput MiB/s |
|---|---|---|---|
| 100 GiB | 400 | 4,000 | 110 |
| 1 TiB | 4,000 | 10,000 | 200 |
| 10 TiB | 10,000 | 30,000 | 1,000 |

Hit the limit → throttled → operations queue or fail. Visible as `ServerBusy` in metrics.

---

## ⌨️ Activity 1 — Inspect DSR Files accounts (read-only)

1. Portal → `stanlnznfileprdrosi01` → **Overview**. Confirm: kind = **FileStorage**, performance = **Premium**, redundancy = **ZRS**.
2. **File shares** → see `permanent`, etc.
3. Click into `permanent` → **Settings** → note: protocol = NFS, root squash setting, snapshot count.
4. Repeat for `stanlnznfileprdrosi02` (Standard SMB).

## ⌨️ Activity 2 — Deploy a Premium NFS share in your training sub

```bash
RG=rg-training-files-<your-initials>
az group create -n $RG -l australiaeast --tags purpose=dsr-training

# Premium FileStorage account, NFS-capable
SA=sttrainnfs<your-initials>$RANDOM
az storage account create -n $SA -g $RG -l australiaeast \
  --sku Premium_ZRS --kind FileStorage \
  --enable-large-file-share \
  --default-action Deny  # Firewall blocks public

# Allow your IP for mounting (replace with your IP)
MY_IP=$(curl -s https://api.ipify.org)
az storage account network-rule add -n $SA -g $RG --ip-address $MY_IP

# Disable secure transfer (NFS doesn't support TLS in production GA yet)
az storage account update -n $SA -g $RG --https-only false

# Create the NFS share (100 GiB minimum)
az storage share-rm create \
  --storage-account $SA \
  --name preservation \
  --quota 100 \
  --enabled-protocols NFS \
  --root-squash NoRootSquash
```

## ⌨️ Activity 3 — Spin up a small RHEL VM to mount from

```bash
az vm create -g $RG -n vm-trainfile-<your-initials> \
  --image RedHat:RHEL:9-lvm-gen2:latest \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard \
  --tags purpose=dsr-training

# Get the public IP
VM_IP=$(az vm show -g $RG -n vm-trainfile-<your-initials> -d --query publicIps -o tsv)
echo "VM at $VM_IP"
```

## ⌨️ Activity 4 — Mount the NFS share

```bash
ssh azureuser@$VM_IP

# On the VM:
sudo dnf install -y nfs-utils
sudo mkdir -p /mnt/preservation
sudo mount -t nfs -o vers=4,minorversion=1,sec=sys,nconnect=4 \
  <SA>.file.core.windows.net:/<SA>/preservation /mnt/preservation

mount | grep preservation
df -h /mnt/preservation

# Test write
sudo touch /mnt/preservation/hello.txt
ls -la /mnt/preservation/
```

✅ You see the share mounted, ~100 GiB, and a file written.

## ⌨️ Activity 5 — Performance metrics

In the portal, on the storage account → **Insights** (Storage Insights):

- Latency (success/failure)
- Transactions
- Egress / Ingress
- Availability
- Used capacity

Generate some load from the VM:

```bash
# Sequential write test
sudo dd if=/dev/zero of=/mnt/preservation/test.bin bs=1M count=500 conv=fsync
```

Watch the metrics tick up.

## ⌨️ Activity 6 — Simulate throttling

```bash
# Multiple parallel writes to push IOPS
for i in {1..10}; do
  sudo dd if=/dev/zero of=/mnt/preservation/test-$i.bin bs=4k count=10000 oflag=direct &
done
wait
```

In Storage Insights → look for **Transactions / Server-side latency**. Spike = pressure. Repeated `ServerBusy` responses = actual throttling.

## ⌨️ Activity 7 — Inspect a Standard SMB share

DSR's `stanlnznfileprdrosi02` uses SMB. SMB on Linux uses Kerberos for AD auth.

For training, mount with shared key:

```bash
SA_SMB=sttrainsmb<your-initials>$RANDOM
az storage account create -n $SA_SMB -g $RG -l australiaeast \
  --sku Standard_ZRS --kind StorageV2

az storage share create -n adminshare --account-name $SA_SMB

KEY=$(az storage account keys list -n $SA_SMB -g $RG --query '[0].value' -o tsv)

# On your VM:
sudo dnf install -y cifs-utils
sudo mkdir -p /mnt/admin
sudo mount -t cifs //$SA_SMB.file.core.windows.net/adminshare /mnt/admin \
  -o vers=3.1.1,username=$SA_SMB,password=$KEY,uid=1000,gid=1000,serverino,nosharesock,actimeo=30

mount | grep admin
sudo touch /mnt/admin/hello.txt
```

In production, DSR uses Kerberos (via `SSSD`) instead of shared key — but the mount semantics are the same.

## ⌨️ Activity 8 — Tear down

```bash
exit  # leave the VM
az group delete -n $RG --yes --no-wait
```

---

## 🦾 Now your turn!

Write a KQL query for Log Analytics that returns:

- Total `ServerBusy` (throttled) responses on `stanlnznfileprdrosi01` over the last 7 days
- Grouped by hour
- Rendered as a column chart

Hint: the table is `StorageFileLogs`, the field is `StatusCode`.

---

## ✅ Success checklist

- [ ] You've inspected both DSR Files accounts in production
- [ ] You've deployed and mounted a Premium NFS share with DSR's mount options
- [ ] You've deployed and mounted a Standard SMB share
- [ ] You've watched performance metrics under load
- [ ] You know how to spot throttling
- [ ] You torn down everything

---

## 💰 Cost note

< NZD $3 if torn down within 1–2 hours.

---

➡️ **Next:** [Step 09 — Data protection posture](step-09-data-protection.md)