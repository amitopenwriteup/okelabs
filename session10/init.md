# Lab: Using a Custom Cloud-init Initialization Script to Set Up OKE Managed Nodes

**Estimated time:** 40–50 minutes
**Level:** Intermediate

> **Note:** OCI Console labels change over time. If a menu name differs slightly, use the console search bar.

---

## Concept

**Cloud-init** runs once, on the first boot of each worker node. In OKE you can supply a custom initialization script to:

- Install packages or agents (monitoring, security tools)
- Configure OS settings (`sysctl`, ulimits, time zone)
- Write files, create users, or configure proxies and registries
- Prepare storage or mounts

## Rules You Must Know

| Rule | Why it matters |
|---|---|
| Script runs **only on new nodes** | Editing it does not change existing nodes until they are replaced or cycled |
| Must call the **OKE init script** | Otherwise the node never registers with the cluster |
| Supported formats | Shell script (`#!/bin/bash`), `#cloud-config`, or multipart MIME |
| Keep it **fast and idempotent** | Long or failing scripts delay or break node readiness |
| No secrets in plain text | Script content is visible in instance metadata |

## The Mandatory OKE Bootstrap Lines

Every shell-format script must include:

```bash
curl --fail -H "Authorization: Bearer Oracle" -L0 \
  http://169.254.169.254/opc/v2/instance/metadata/oke_init_script \
  | base64 --decode > /var/run/oke-init.sh
bash /var/run/oke-init.sh
```

Place OS preparation commands **before** this call so they finish before the kubelet starts.

## Objective

Write a script that tunes the OS, installs utilities, and leaves a marker file; apply it to a node pool; verify it ran on the node.

## Prerequisites

| Requirement | Details |
|---|---|
| Existing OKE cluster | With `kubectl` access (Cloud Shell recommended) |
| Worker subnet outbound access | NAT gateway, service gateway, or mirror, so `yum` can install packages |
| SSH key pair | To log in to the worker node |
| Bastion or network path | Required if worker nodes are in a private subnet |

---

## Step 1 – Write the Script

1. Open **Cloud Shell** from the top-right of the OCI Console.
2. Create the file:

```bash
cat > cloud-init.sh <<'EOF'
#!/bin/bash
set -x

# ---- Custom setup (runs before the node joins the cluster) ----
LOG=/var/log/oke-custom-init.log
{
  echo "Custom init started: $(date)"

  # 1. Kernel tuning
  cat >/etc/sysctl.d/99-oke-lab.conf <<SYS
vm.max_map_count=262144
net.core.somaxconn=1024
SYS
  sysctl --system

  # 2. Install helpful tools
  yum install -y htop jq nmap-ncat

  # 3. Node banner and marker file
  echo "Provisioned by OKE custom cloud-init on $(hostname)" > /etc/motd
  touch /etc/oke-lab-marker

  echo "Custom init finished: $(date)"
} >> "$LOG" 2>&1

# ---- Mandatory OKE bootstrap ----
curl --fail -H "Authorization: Bearer Oracle" -L0 \
  http://169.254.169.254/opc/v2/instance/metadata/oke_init_script \
  | base64 --decode > /var/run/oke-init.sh
bash /var/run/oke-init.sh
EOF
```

3. Check the syntax:

```bash
bash -n cloud-init.sh && echo "Syntax OK"
```

4. Download the file to your computer (Cloud Shell menu → **Download**) or copy its contents, so you can paste it in the console.

> The example uses `yum` for Oracle Linux 7/8. `dnf` also works on Oracle Linux 8 and later.

---

## Step 2 – Create a Node Pool with the Script

1. Open **Developer Services → Containers & Artifacts → Kubernetes Clusters (OKE)** and select your cluster.
2. Under **Resources**, click **Node pools → Add node pool**.
3. Configure:

| Field | Value |
|---|---|
| Name | `pool-cloudinit` |
| Node type | Managed |
| Kubernetes version | Same as the control plane |
| Worker node subnet | Your worker subnet |
| Shape | `VM.Standard.E5.Flex`, 1 OCPU / 8 GB |
| Image | Oracle Linux (OKE image) |
| Number of nodes | 1 |
| SSH public key | Paste your public key |

4. Click **Show advanced options**.
5. Under **Initialization script**, choose **Paste Cloud-Init Script** (or **Choose Cloud-Init Script file**) and provide the contents of `cloud-init.sh`.
6. Click **Add**.

---

## Step 3 – Verify the Script Ran

1. Wait for the node pool to become **Active**, then check the node:

```bash
kubectl get nodes -l name=pool-cloudinit
```

Expected: one node in `Ready` state.

2. SSH to the node (through a bastion if it is in a private subnet):

```bash
ssh -i ~/.ssh/id_rsa opc@<node-private-ip>
```

3. Run these checks on the node:

```bash
cat /etc/oke-lab-marker
cat /etc/motd
cat /var/log/oke-custom-init.log
sysctl vm.max_map_count
rpm -q htop jq
sudo tail -50 /var/log/cloud-init-output.log
```

Expected results:

| Check | Expected |
|---|---|
| Marker file | Exists (no error from `cat`/`ls`) |
| `/etc/motd` | Contains the "Provisioned by OKE custom cloud-init" text |
| `vm.max_map_count` | `262144` |
| `htop`, `jq` | Installed |
| Init log | Shows start and finish timestamps |

---

## Step 4 – Update the Script and Apply It

1. Edit `cloud-init.sh` (for example, add the line `net.ipv4.ip_local_port_range = 10240 65000` (no quotes) to the sysctl block in the script).
2. In the console, open **Node pools → pool-cloudinit → Edit**.
3. Replace the initialization script under **Show advanced options**, then click **Save changes**.
4. Confirm that the existing node is unchanged (the new setting is absent).
5. Apply the change to all nodes by using **Cycle nodes** on the node pool (set maximum addional node to `0` and maximum unavailable to `1`), or by scaling up and deleting the old node.
6. Re-run the verification checks on a new node, and confirm the new setting:

```bash
cat /etc/sysctl.d/99-oke-lab.conf
```

Expected: `net.ipv4.ip_local_port_range = 10240 65000` on the new node. The old node shows the kernel default (typically `32768 60999`).

---

## Validation Checklist

- [ ] Script contains the OKE bootstrap lines
- [ ] Node pool created with the initialization script
- [ ] Marker file, MOTD and log exist on the node
- [ ] `sysctl` value and installed packages confirmed
- [ ] Node is `Ready`
- [ ] Updated script applied to a new or cycled node

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Node never becomes `Ready` | OKE init call missing, or script failed before reaching it | Check `/var/log/cloud-init-output.log`; make optional steps non-fatal |
| Packages fail to install | No outbound route from worker subnet | Add a NAT or service gateway route, or use an internal mirror |
| Script changes not visible on old nodes | Script runs only at first boot | Cycle or replace the nodes |
| Node pool creation fails on script upload | Invalid format or encoding | Confirm the file starts with `#!/bin/bash` and passes `bash -n` |
| Cannot SSH to node | Private subnet or security rules | Use a bastion; allow port 22 from the bastion CIDR |

---

## Clean-up

1. Delete the node pool: **Cluster → Node pools → pool-cloudinit → More actions → Delete node pool**.
2. Remove local files if no longer needed:

```bash
rm -f cloud-init.sh
```

---

## Review Questions

1. What happens if the OKE bootstrap call is missing from your script?
2. Why does editing the initialization script not affect running nodes?
3. Why should custom commands usually run before the OKE bootstrap call?
4. How can you apply an updated script to every node in a pool?
5. Why should secrets not be placed in a cloud-init script?
