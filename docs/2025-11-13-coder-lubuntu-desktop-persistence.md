---
title: "Persistent Lubuntu Desktop Workspaces for Coder"
date: 2025-11-13
categories:
  - infrastructure
  - coder
  - ai
  - observability
tags:
  - coder
  - proxmox
  - truenas
  - langfuse
  - litellm
  - kasmvnc
  - terraform
excerpt: "Deep dive into the dual-disk Coder template: Proxmox + TrueNAS persistence, KasmVNC desktops, Langfuse-observed AI tooling, Wazuh telemetry, and OpenBao-managed secrets." 
---

The refreshed Lubuntu desktop workspace is the backbone of Ozteklab’s AI portfolio demos. It takes the one-off scripts and experiments we used last quarter and locks them into a reproducible desktop that survives reboots, keeps `/home` on TrueNAS, traces every AI call via Langfuse, and reports into Wazuh the second a session starts. Below is the full recipe, with Terraform, cloud-init, Ansible, OpenBao, and TrueNAS all working together so a “workspace” behaves like a hardened laptop.

![Coder workspace dashboard](https://s3.local.ozteklab.com/uploads/coder-screenshots/01-coder-workspaces-dashboard.png)

## Why rebuild the Coder desktop?

1. **Ephemeral disks were dangerous.** The first-gen template used Proxmox linked clones only. Any stop/start cycle caused Proxmox to reprovision the disk, nuking `/home` in the process.
2. **NFS couldn’t keep up.** Mounting `/home` over NFS looked good on paper, but VS Code Remote, Langfuse tracing, and KasmVNC all stalled whenever the NAS was busy.
3. **Secrets shouldn’t depend on manual variables.** Vault’s BSL changes pushed me to OpenBao. Every secret is fetched at build time; template pushes contain zero credentials.

So the new architecture couples Terraform, cloud-init, and Ansible to deliver a reproducible desktop that behaves like a real workstation.

### Workspace boot timeline

1. **Template clone (Proxmox).** Terraform clones template `5013`, attaches the OS disk plus the dedicated TrueNAS ZVOL, and renders cloud-init data on the fly.
2. **cloud-init (first boot).** The bootstrap script formats `/dev/sdb`, seeds `/home`, installs QEMU Guest Agent/OpenSSH, and drops the Langfuse session proxy.
3. **Ansible (post-boot).** When the Coder agent phones home, Ansible roles install Langfuse, LiteLLM, Wazuh, Kasm helpers, OpenMemory CLI, and Docker tooling.
4. **User login (KasmVNC).** The Lubuntu desktop is exposed through the browser, VS Code already trusts the Coder agent, and `/home` lives on the persistent disk with nightly snapshots.

Each step is deterministic, so anyone pointing the Terraform module at their own Proxmox + OpenBao endpoints can reproduce the exact same desktop.

## Storage + lifecycle (Proxmox + TrueNAS)

Key pieces from `main.tf`:

```hcl
resource "proxmox_virtual_environment_vm" "workspace" {
  clone { vm_id = 5013 }

  disk {                   # scsi0 – OS disk
    interface    = "scsi0"
    datastore_id = var.proxmox_storage
    size         = data.coder_parameter.disk_size.value
  }

  disk {                   # scsi1 – persistent data disk
    interface    = "scsi1"
    datastore_id = "TANK-K8S"
    size         = data.coder_parameter.data_volume_size.value
    file_format  = "raw"
    iothread     = true
    cache        = "writeback"
  }

  lifecycle {
    ignore_changes = [started, clone, disk, initialization]
  }
}
```

- **`scsi1` lives on TrueNAS (pool `TANK-K8S`).** That disk is the user’s `/home`. TrueNAS snapshots the ZVOL nightly, so even if someone trashes their desktop we can roll back just the data disk without touching the VM OS.
- **`ignore_changes` stops Terraform from destroying the VM.** Workspaces can stop/start safely; Terraform only toggles power state.
- **`null_resource vm_cleanup`** handles orphaned VM IDs before provisioning so retries stay clean.

## Persistent home mount logic (cloud-init)

`cloud-init.yaml.tpl` emits a bootstrap script that waits for `/dev/sdb` (scsi1) and only formats it on the first boot:

```bash
if [ -b /dev/sdb ]; then
  if ! sudo blkid /dev/sdb | grep -q TYPE; then
    sudo mkfs.ext4 -F /dev/sdb
  fi

  sudo mount /dev/sdb /mnt/persistent-home
  if [ -z "$(ls -A /mnt/persistent-home)" ]; then
    sudo mv /home/$WORKSPACE_USER /home/$WORKSPACE_USER.orig
    sudo cp -a /home/$WORKSPACE_USER.orig/. /mnt/persistent-home/
  fi

  sudo umount /mnt/persistent-home
  sudo mount /dev/sdb /home/$WORKSPACE_USER
  echo "/dev/sdb /home/$WORKSPACE_USER ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
fi
```

This means:
- First boot seeds the persistent disk with the Lubuntu skeleton.
- Subsequent reboots remount `/home` immediately, so git repos stay put.
- Because `/dev/sdb` is where TrueNAS snapshots live, we only backup what matters; the OS disk can always be recloned from template 5013.
- Nightly `zfs snapshot` + weekly `zfs send` jobs on TrueNAS keep historical restore points without touching the VM template.

## Secrets + configuration via **OpenBao**

The template still uses the official Vault provider, but the backend is **OpenBao** (fully open-source fork). Every credential is fetched dynamically at workspace build time:

```hcl
data "vault_kv_secret_v2" "proxmox" {
  mount = "secret"
  name  = "proxmox"
}

provider "proxmox" {
  endpoint  = data.vault_kv_secret_v2.proxmox.data["api_url"]
  api_token = "${local.proxmox_token_id_clean}=${data.vault_kv_secret_v2.proxmox.data["token_secret"]}"
}
```

- **No template push variables.** The only placeholder is `vault_token` when the template is registered. On build, Terraform reads Proxmox, LiteLLM, Anthropic, Forgejo, and Wazuh secrets straight from OpenBao.
- **SSO-friendly.** OpenBao integrates with the same identity provider that guards Coder, so rotating tokens is centralized.
- **Multi-module reuse.** The same provider block feeds the KasmVNC module, the OpenMemory MCP module, and the Codex CLI environment setup.

### How secrets travel end-to-end

1. **Template registration.** The only manual input is the short-lived OpenBao token. Terraform, cloud-init, and Ansible artifacts stay credential-free.
2. **Workspace build.** Terraform authenticates to OpenBao using that token and retrieves Proxmox, LiteLLM, Anthropic, Forgejo, Authentik, and Wazuh credentials over TLS 1.3. Nothing is echoed to stdout; it is all stored in in-memory locals.
3. **cloud-init consumption.** User-data renders one-time files (e.g., `/etc/codex.d/*.env`) with `0600` permissions, then wipes any temporary copy. Secrets only live on disk if the target service needs them at runtime.
4. **Ansible follow-up.** Roles use the `hashivault` lookup plugin so they request short-lived tokens during execution. When Ansible exits, there is no `.vault_pass` or `ansible.cfg` residue.

Rotating a LiteLLM key or swapping the Wazuh enrollment password is now just an OpenBao change. Existing desktops pick up the new value on their next rebuild, and no repo history ever contains an API key.

## Cloud-init + Ansible automation

After Terraform clones the VM and writes user-data, two layers finish the machine:

1. **cloud-init script** (rendered from `cloud-init.yaml.tpl`) handles:
   - Fixing SSH key permissions from the K8s secret mount.
   - Mounting `/dev/sdb` as described above.
   - Installing QEMU Guest Agent and OpenSSH before the Coder agent connects.
   - Dropping `langfuse-session-proxy.py` that binds LiteLLM traffic to Langfuse sessions.
   - Writing `langfuse-session-proxy.service` so tracing restarts automatically after reboots.

2. **Ansible playbook** (`ansible/playbook.yml`) runs inside the workspace using roles:
   - `langfuse` – installs `langfuse` + `litellm`, and a `/usr/local/bin/litellm-with-session` helper.
   - `coder-agent` – configures the Coder agent, VS Code server, Kasm helper scripts, and the OpenMemory CLI.
   - `wazuh-agent` – enrolls the VM with the Wazuh manager so SecOps gets process/log telemetry immediately. Future iteration: add AI-aware rules to flag suspicious docker runs or scripts triggered by Codex.
   - `docker` + `user-environment` – ensure developers land in a ready-to-use Lubuntu desktop with Docker, CLI tools, and prompts.

![Kasm desktop via browser](https://s3.local.ozteklab.com/uploads/coder-screenshots/04-kasm-linux-desktop.png)

## AI + desktop UX

1. **KasmVNC** publishes the desktop through Coder’s app drawer (port 6901). Users don’t need a local VNC client, and file transfer/clipboard sharing are native to the browser UI.
2. **VS Code (desktop + web)** comes preinstalled. The desktop version launches inside Kasm, and the web version is exposed through Coder’s port forwarding panel (see screenshot #5).
3. **Langfuse everywhere** – `langfuse-session-proxy.py` listens on port 7777. Every Codex CLI interaction, Droid AI call, or arbitrary `curl` to LiteLLM inherits `LANGFUSE_SESSION_ID`, `LANGFUSE_USER_ID`, etc. Traces show up in `https://langfuse.ozteklab.com` with workspace/user tagging.
4. **OpenMemory MCP server** – the workspace already runs the multi-user OpenMemory service (Authentik OIDC + MCP). Users click the “OpenMemory” app in Coder, get redirected through Authentik, and land inside the memory UI without touching tokens. Under the hood, Terraform pulls the Authentik client/secret from OpenBao and the MCP server authenticates via OAuth automatically.

<img
  src="https://s3.local.ozteklab.com/uploads/coder-screenshots/05-coder-port-forwarding.png"
  alt="Port forwarding panel"
  style={{maxWidth: '450px', width: '100%', height: 'auto', display: 'block', margin: '0 auto'}}
/>

### Living inside the workspace

- **Browser-native workflow.** Kasm profiles launch Firefox, VS Code, and Langfuse dashboards with preloaded tabs. Clipboard sync + file transfer mean you can move artifacts between the desktop and your laptop without SCP.
- **Git + pnpm ready.** The Ansible `user-environment` role installs Node 20, pnpm, Python 3.11, uv, Docker, Terraform, and the Codex CLI. A `post-login.sh` message of the day spells out what ports are forwarded and which services already run.
- **AI tooling on day zero.** LiteLLM routes to Claude, GPT, and internal Ollama models; Langfuse proxies every call; and the OpenMemory CLI already knows which workspace/user issued each note. Testing the ozteklab.com chat stack locally produces the same traces as production.
- **Security overlays.** Authentik enforces MFA at the Coder portal, Wazuh records process starts, and systemd audit rules capture attempts to load kernel modules or mount external USB devices. It feels like a laptop, but every keystroke is observable.

## Security & observability in one turn

- **Langfuse traces** track prompt, completion, latency, and cost across Codex CLI + Droid.
- **Wazuh agent** gives SecOps visibility into processes, logs, and file integrity events per workspace.
- **TrueNAS snapshots** protect `/home` separately from the OS disk, so restoring data doesn’t entail recloning the VM.
- **OpenBao** as the secret source means SSO + RBAC apply uniformly, and the Terraform code doesn’t expose API keys.
- **Authentik SSO** keeps Coder, OpenMemory, and Langfuse under the same MFA gate, so every surface inherits the same trust boundary.

### Day-2 operations checklist

1. **Snapshot hygiene.** TrueNAS takes hourly `zfs snapshot` cuts during work hours, nightly long-term snaps, and a weekly `zfs send` to object storage. Only the `/home` disk travels; the OS template is disposable.
2. **Workspace webhooks.** Coder emits `workspace.stopped` and `workspace.deleted` events to a FastAPI callback. That service tags Langfuse sessions as complete, clears dangling port forwards, and requests a lightweight Wazuh scan so SOC sees the final state.
3. **Patch windows.** A GitHub Actions job rebuilds the base Lubuntu template every Friday with `apt-get dist-upgrade` and pushes the new VM ID into Terraform. Existing desktops can reboot to pick up kernel fixes without manual snapshots.
4. **Drift detection.** Nightly `terraform plan -refresh-only` runs compare the desired CPU/RAM against what Proxmox reports. If someone tampers with a VM, the diff lands in Mattermost before it snowballs.
5. **Audit trails.** All Terraform plans, Ansible logs, and Langfuse traces are archived to S3 so future incident reviews can replay exactly what happened during a workspace build.

## Where to go next

1. Add AI-aware Wazuh rules that inspect code execution triggered by Codex tasks.
2. Automate incremental backups by calling the TrueNAS API whenever a workspace stops.
3. Publish an Outline doc that mirrors this post (diagrams + screenshots) so team members can self-serve.
4. Wire the upcoming markdown editing UI into these workspaces so changes to ozteklab.com can be staged, previewed, sanitized, and vectorized from the same desktop.

Need the Terraform or Ansible files? They live under `infra/coder` in the repo; just mirror the OpenBao mount paths and you can reproduce the same flow.
