# Step 15 — WOD container operations

_The "web archive in containers" lab._ 🕸️ Builds working knowledge of the Web On-Demand stack: PODMAN containers running NGINX, pyweb, and OutbackCDX, with `blobfuse2` mounting `stanlnznblobprdwod01` for content access.

> [!NOTE]
> **Trainee duration:** 90 minutes
> **Instructor EDE:** 3.5 hours (1h prep + 1.5h delivery + 1h Q&A buffer)
> **Lab cost:** under NZD $0.50 — a small Linux VM with PODMAN.
> **Prerequisites:** Steps 01–05, 08 complete; basic Linux comfort.
> **Pairs with:** Module 3 of the DIA training plan (Application Operations).

---

## 📖 Session overview

WOD (Web On-Demand) is DIA's web-archive replay surface — visitors browse archived web content live. Architecturally it's a small set of containers (NGINX as TLS terminator and reverse proxy, pyweb as the WARC-replay engine, OutbackCDX as the index server) running under **PODMAN** on a Linux VM. The VM mounts the WOD blob storage account via **blobfuse2** so the containers see WARC files as a regular filesystem. Your team operates this stack — restart containers, read logs, validate the blobfuse mount, swap config. This lab gets you fluent.

**What you'll learn**
- The role of each container: **NGINX**, **pyweb**, **OutbackCDX**.
- How **PODMAN** differs from Docker (rootless, daemonless, systemd integration).
- How **`blobfuse2`** mounts a storage account container as a Linux filesystem.
- How a request flows: AGW → NGINX → pyweb → OutbackCDX → WARC (via blobfuse2) → response.
- How to start/stop/inspect containers, read their logs.
- How to test the blobfuse mount and recognise mount failures.

## 💡 Jargon buster

| Term | Plain meaning |
|---|---|
| **WOD** | Web On-Demand — the live replay surface for archived web content. |
| **WARC** | Web ARChive file format — standard for storing crawled web content. |
| **PODMAN** | A daemonless OCI container engine; near-drop-in for Docker but rootless and systemd-friendly. |
| **NGINX** | The TLS terminator and reverse proxy at the front of the WOD stack. |
| **pyweb** | The WARC replay engine that returns archived web pages. |
| **OutbackCDX** | The index server — given a URL, returns which WARC has the page. |
| **blobfuse2** | A FUSE-based filesystem driver that exposes a blob container as a Linux directory. |
| **Systemd unit** | A service definition Linux uses to start/stop daemons. PODMAN containers can run as systemd units. |
| **Rootless** | PODMAN runs as a normal user — no root needed, smaller blast radius. |

## 📚 Prepare in advance — Microsoft Learn

| Module | Why it matters for ANL |
|---|---|
| [Introduction to containers](https://learn.microsoft.com/training/modules/intro-to-containers/) | If you've never touched containers — start here. |
| [blobfuse2 documentation](https://learn.microsoft.com/azure/storage/blobs/blobfuse2-what-is) | The mount used by WOD. |
| [Run containers with PODMAN](https://podman.io/docs) | PODMAN's official docs. |

About **2 hours** of optional pre-reading. If new to containers, double that.

## 🧱 Foundational primer

- **WOD doesn't write anything** to the WARC blob storage — it's read-only. WARCs are produced by the crawl pipeline (separate process, separate team).
- **PODMAN is rootless** — every container runs under a non-privileged user. No `root@host` exposure.
- **`blobfuse2` is read-only by config** — mount options enforce read-only when DSR provisions it.
- **NGINX terminates TLS** so containers downstream see plain HTTP. The TLS cert is in Key Vault and rotated via the cron-installed `certbot` or similar.
- **pyweb consults OutbackCDX before serving** — that's the index lookup. If OutbackCDX is down, pyweb returns 404 even for content that exists.
- **All container logs go to systemd-journald** and are forwarded to Log Analytics via the Azure Monitor Agent.
- **Restart order matters**: blobfuse2 must be up before pyweb; OutbackCDX before pyweb; NGINX last.

## ⌨️ Activity 1 — Provision a tiny lab VM

1. Portal → **Virtual machines → + Create**.
2. RG: `rg-labs-foundations-<your-initials>`. Region: Australia East. VM name: `vm-wod-lab`.
3. Image: Ubuntu Server 22.04 LTS. Size: B1s.
4. Auth: SSH public key (paste your key).
5. Inbound ports: SSH only.
6. Create. Wait ~3 min. SSH in.

## ⌨️ Activity 2 — Install PODMAN and blobfuse2

```bash
# On the VM
sudo apt update
sudo apt install -y podman blobfuse2 fuse3
podman --version    # confirm >= 4.0
blobfuse2 --version # confirm >= 2.0
```

## ⌨️ Activity 3 — Mount your test storage account via blobfuse2

Use the storage account from Step 05.

```bash
# Create config
cat > ~/blobfuse2.yaml <<'EOF'
allow-other: false
file-cache:
  path: /tmp/bf2-cache
  timeout-sec: 120
  max-size-mb: 256
azstorage:
  type: block
  account-name: <YOUR STORAGE ACCOUNT NAME>
  container: lab
  mode: msi              # use Managed Identity
EOF

# Enable a system-assigned MI on the VM (Portal → VM → Identity)
# Grant the VM's MI "Storage Blob Data Reader" on your storage account

# Mount
mkdir ~/wod
blobfuse2 mount ~/wod --config-file=~/blobfuse2.yaml --read-only
ls ~/wod
```

You should see your test blob from Step 05's container `lab`. Treat the mount as read-only.

> [!IMPORTANT]
> Use **Managed Identity** auth (not account key). DSR uses MI exclusively — the VM has its MI granted Storage Blob Data Reader on the WOD storage account.

## ⌨️ Activity 4 — Run a container under PODMAN

```bash
# Pull and run a tiny NGINX container, mapping our blob mount as a static site
podman run -d --name nginx-lab -p 8080:80 \
  -v $HOME/wod:/usr/share/nginx/html:ro,Z \
  docker.io/library/nginx:latest

# Inspect
podman ps
podman logs nginx-lab

# Test
curl http://localhost:8080/
```

You're serving content from blobfuse2 via an NGINX container. This is the same pattern WOD uses, scaled down to one box.

## ⌨️ Activity 5 — Stop / start / inspect

```bash
podman stop nginx-lab
podman ps -a
podman start nginx-lab
podman exec -it nginx-lab nginx -t   # test the config inside the container
podman logs --since=10m nginx-lab    # last 10 minutes
```

In production WOD, the corresponding commands restart the live containers — only do them under change control.

## ⌨️ Activity 6 — systemd unit pattern (production setup)

Production WOD has containers wrapped as systemd services so they restart on reboot.

```bash
# Generate a systemd unit
podman generate systemd --new --name nginx-lab > ~/.config/systemd/user/nginx-lab.service
systemctl --user daemon-reload
systemctl --user enable --now nginx-lab.service
systemctl --user status nginx-lab.service
```

Now NGINX restarts automatically on boot. WOD's prod stack uses this pattern for all three containers.

## ⌨️ Activity 7 — Read container logs in Log Analytics (production view)

DSR forwards journald logs to Log Analytics. Sample query (production):

```kql
Syslog
| where Computer startswith "vm-wod"
| where Facility == "daemon"
| where SyslogMessage has "nginx" or SyslogMessage has "pyweb" or SyslogMessage has "outbackcdx"
| where TimeGenerated > ago(1h)
| project TimeGenerated, Computer, SeverityLevel, SyslogMessage
| order by TimeGenerated desc
```

For HTTP access patterns:

```kql
Syslog
| where Computer startswith "vm-wod"
| where SyslogMessage has "nginx" and SyslogMessage has " 200 "
| summarize requests = count() by bin(TimeGenerated, 5m)
| render timechart
```

## ⌨️ Activity 8 — Validate the blobfuse mount

```bash
# On the VM
mount | grep blobfuse2
df -h ~/wod              # shows storage account capacity
ls ~/wod | head          # quick listing
stat ~/wod/<a blob name> # latency check; should be sub-second after first call
```

If `df` shows nothing or `ls` hangs, the mount has dropped. The fix is `umount ~/wod && blobfuse2 mount ...` again. In production this is a documented runbook step.

## 🦾 Now your turn!

1. Run a second container (e.g. `httpd:alpine`) in parallel and confirm both can serve from the same blobfuse mount.
2. Set the file cache size to 1 GB and re-mount. Test response time on the first vs second request.
3. From the VM, write a one-liner that prints "blobfuse mounted" or "BLOBFUSE MISSING" suitable for cron.
4. Find the Azure Monitor Agent on the VM (`systemctl status mdsd` or check `/etc/azure/...`) and confirm it's running. Where would you prove forwarded logs are reaching Log Analytics?

## ✅ Success checklist

- [ ] You can name the three containers in WOD and what each does.
- [ ] You've mounted a storage account container via blobfuse2 with Managed Identity.
- [ ] You've started, stopped, inspected a PODMAN container.
- [ ] You can read container logs both locally (`podman logs`) and in Log Analytics.
- [ ] You can validate (and re-mount if needed) the blobfuse2 mount.
- [ ] You've **deleted** the lab VM after the lab.

## 📚 Self-serve refresher

- [PODMAN cheat sheet](https://docs.podman.io/en/stable/Tutorials.html) — every command you'll use day-to-day.
- [blobfuse2 troubleshooting](https://learn.microsoft.com/azure/storage/blobs/blobfuse2-troubleshoot) — when the mount goes weird.

## 💰 Cost note

- B1s VM: ~NZD $0.013/hour (~$10/month). For 2-hour lab: $0.03.
- Public IP: ~$0.005/hour.

**Cleanup:**

```bash
az vm delete -g rg-labs-foundations-<your-initials> -n vm-wod-lab --yes
az network public-ip delete -g rg-labs-foundations-<your-initials> -n vm-wod-lab-ip
az network nic delete -g rg-labs-foundations-<your-initials> -n vm-wod-labVMNic
```

(Or just `az group delete -n rg-labs-foundations-<your-initials> --yes` once you're done with all of Phase 3 — but you'll want this RG for Phase 4.)

---

⬅️ **Previous:** [Step 14 — Oracle on Azure (read-only ops)](step-14-oracle-on-azure.md)
➡️ **Next:** [Step 16 — Log Analytics & KQL for storage and VMs](step-16-log-analytics-kql.md) (Phase 4 begins)
