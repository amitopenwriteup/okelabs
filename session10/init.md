# Using Custom Cloud-init Scripts to Set Up Managed Nodes (OKE)

## Demo: Creating a Custom Cloud-init Initialization Script

This guide assumes you already have an **OKE cluster running** with managed nodes (node pool). Cloud-init scripts run **once, at first boot** of a node, so changes only affect **new nodes** (new node pool, scale-up, or replaced/cycled nodes). Existing nodes are not modified.

---

## 1. How it works

- OKE nodes are bootstrapped by a default cloud-init script that joins the node to the cluster (`oke-init.sh`).
- A custom script **replaces** the default one, so it **must still call the OKE init script**, otherwise the node will not join the cluster.
- Supported formats: shell script (`#!/bin/bash`), `#cloud-config`, or MIME multipart.

## 2. Create the script

Save as `custom-cloud-init.sh`:

```bash
#!/bin/bash

# ---- 1. Required: OKE bootstrap (do not remove) ----
curl --fail -H "Authorization: Bearer Oracle" -L0 \
  http://169.254.169.254/opc/v2/instance/metadata/oke_init_script \
  | base64 --decode > /var/run/oke-init.sh
bash /var/run/oke-init.sh

# ---- 2. Your customizations ----
# Install extra packages
yum install -y git jq htop

# Set a custom MOTD / marker file to verify the script ran
echo "Customized by cloud-init on $(date)" > /etc/motd
echo "cloud-init custom script ran" > /var/tmp/cloud-init-demo.txt

# Example: tune kernel parameter
echo "vm.max_map_count=262144" >> /etc/sysctl.conf
sysctl -p
```

> On Ubuntu images use `apt-get update && apt-get install -y git jq htop` instead of `yum`.

## 3. Apply it to the node pool (Console)

**Option A: New node pool (recommended for an existing cluster)**

1. Console → **Developer Services → Kubernetes Clusters (OKE)** → select your cluster.
2. **Node pools → Add node pool**.
3. Fill in name, shape, image, node count, subnets.
4. Expand **Show advanced options → Initialization script**.
5. Choose **Paste cloud-init script** (or upload `custom-cloud-init.sh`).
6. Click **Add**.

**Option B: Existing node pool**

1. Cluster → **Node pools** → select the pool → **Edit**.
2. **Show advanced options → Initialization script** → paste/upload the script → **Save changes**.
3. Only **new nodes** use it. Scale the pool up, or cycle nodes (**Cycle nodes** / replace nodes) so they boot with the new script.

## 4. Verify

```bash
# Node joined the cluster?
kubectl get nodes -o wide

# SSH to the node (via bastion / private key) and check:
cat /var/tmp/cloud-init-demo.txt
sudo cat /var/log/cloud-init-output.log
sudo cloud-init status
```

## 5. Troubleshooting

| Symptom | Likely cause |
|---|---|
| Node stays `NotReady` / never joins | Script missing the `oke-init.sh` call |
| Changes not on existing nodes | Cloud-init runs only at first boot; cycle/replace nodes |
| Package install fails | No outbound access (NAT gateway / service gateway), wrong package manager |
| Script didn't run | Bad shebang, CRLF line endings, or invalid base64 |

Logs: `/var/log/cloud-init.log` and `/var/log/cloud-init-output.log`.

## 6. Best practices

- Keep the script **idempotent** and short; test on a small pool first.
- Always keep the OKE init call **before** heavy customizations (or at the top).
- Put large installs in custom images instead of cloud-init for faster node boot.
- Don't put secrets in the script (metadata is readable); use OCI Vault.
