# Lab: Upgrading an OKE Cluster and Its Worker Nodes to a Newer Kubernetes Version

**Estimated time:** 60–90 minutes
**Level:** Intermediate

> **Note:** OCI Console labels and the Kubernetes versions on offer change over time. Items most likely to differ are marked **(verify)**.

---

## Concept

An OKE upgrade happens in **two separate parts**:

| Part | What is upgraded | Effect on workloads |
|---|---|---|
| **Control plane** (the cluster) | Kubernetes API server, scheduler, controllers | Running pods keep running. The API may be briefly unavailable during the update. |
| **Worker nodes** (node pools) | kubelet, kube-proxy and the node image on each node | Pods are drained and rescheduled as nodes are replaced. |

## Rules You Must Know

| Rule | Why it matters |
|---|---|
| Upgrade the **control plane first** | A node pool cannot run a newer version than the control plane |
| Upgrade **one minor version at a time** (for example 1.29 to 1.30) | Skipping minor versions is not supported |
| Worker nodes may lag the control plane only within the supported skew | A pool that falls too far behind must be upgraded promptly |
| Editing a node pool's version affects **new nodes only** | Existing nodes keep the old version until they are replaced or cycled |
| Upgrades cannot be rolled back | Plan, back up, and test first |

## Objective

1. Prepare and check a cluster for an upgrade.
2. Upgrade the control plane to the next Kubernetes version.
3. Upgrade the worker nodes with node cycling.
4. (Optional) Upgrade worker nodes with the blue/green method.
5. Verify that the cluster and workloads are healthy.

## Prerequisites

| Requirement | Details |
|---|---|
| Existing OKE cluster | At least one minor version behind the newest available version |
| Node pool | A managed node pool with at least 2 nodes |
| Access | `kubectl` through Cloud Shell |
| Sample workload | Deployed in Step 1 so you can watch availability during the upgrade |
| Service limits | Room for 1 to 2 extra worker nodes (cycling creates surge nodes) |

---

## Step 1 – Prepare for the Upgrade

### 1.1 Record the current state

1. In the console, open **Developer Services → Containers & Artifacts → Kubernetes Clusters (OKE)** and select your cluster.
2. Note the **Kubernetes version** of the control plane. If a newer version is available, an **Upgrade available** link or button appears.
3. Open **Resources → Node pools** and note the version of each pool.
4. In Cloud Shell, confirm versions as the cluster reports them:

```bash
kubectl version
kubectl get nodes -o wide
```

Record the `VERSION` column. All nodes should show the old version.

### 1.2 Deploy a sample workload to watch

```bash
kubectl create deployment upgrade-demo --image=nginx --replicas=4
kubectl get pods -o wide
```

Create a PodDisruptionBudget so drains keep at least 2 replicas available:

```yaml
# pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: upgrade-demo-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: upgrade-demo
```

```bash
kubectl apply -f pdb.yaml
kubectl get pdb
```

### 1.3 Pre-upgrade checklist

| Check | How |
|---|---|
| Read the release notes for the target version | Kubernetes release notes and OKE release notes (look for removed APIs) |
| Look for deprecated or removed APIs in your manifests | Review your YAML against the notes; `kubectl get <resource> -o yaml` shows `apiVersion` |
| Back up important data and manifests | Export manifests; snapshot persistent volumes where needed |
| Check add-on and autoscaler versions | **Resources → Add-ons**; the Cluster Autoscaler version should match the cluster's Kubernetes minor version **(verify)** |
| Check that nodes are healthy | `kubectl get nodes` shows all nodes `Ready` |
| Check PodDisruptionBudgets | `kubectl get pdb -A`; a budget that allows zero disruptions will block drains |
| Confirm spare capacity | Pods from a drained node need room on other nodes |

> **Warning:** Do not start the upgrade while nodes are `NotReady` or while another work request on the cluster is running.

---

## Step 2 – Upgrade the Control Plane

1. On the cluster details page, click **Upgrade available** (or **More actions → Upgrade**, depending on the console version **(verify)**).
2. In the **Upgrade cluster** dialog, choose the **target Kubernetes version**. Only the next minor version (and patch versions) are offered.
3. Read the message about worker nodes, then click **Upgrade**.
4. The cluster state changes to **Updating**. Open **Work requests** (cluster page, left menu) to follow progress. This typically takes several minutes.
5. While waiting, confirm that your workload stays up:

```bash
kubectl get pods -l app=upgrade-demo
```

Pods keep running. If `kubectl` briefly times out, retry after a few seconds.

### Verify the control plane

When the cluster state returns to **Active**:

```bash
kubectl version
kubectl get nodes
```

Expected results:

| Item | Expected |
|---|---|
| Server version in `kubectl version` | The new version |
| `kubectl get nodes` VERSION column | **Still the old version** (workers are not upgraded yet) |
| Console cluster page | New version shown |

> **Important:** `kubectl` itself should stay within one minor version of the server. Update your client if needed.

---

## Step 3 – Upgrade Worker Nodes with Node Cycling

Node cycling replaces each node in a pool with a new node at the new version, one batch at a time, without you draining nodes by hand.

### 3.1 Set the new version on the node pool

1. Open the cluster, then **Resources → Node pools**, and select the pool.
2. Click **Edit**.
3. Set **Kubernetes version** to the new control plane version.
4. Under **Image**, select an OKE node image that matches the new version **(verify)**.
5. Click **Save changes**.

> The existing nodes are **not** changed yet. Only new nodes will use the new version and image.

### 3.2 Cycle the nodes

1. On the node pool page, click **Cycle nodes** (or **More actions → Cycle nodes**).
2. Set the cycling options:

| Setting | Lab value | Meaning |
|---|---|---|
| Maximum surge | `1` | Extra nodes OKE may create at the same time |
| Maximum unavailable | `0` | Nodes that may be out of service at the same time |

3. Confirm. For each batch, OKE creates a new node at the new version, cordons and drains the old node, then terminates it.

### 3.3 Watch the rollout

```bash
kubectl get nodes --watch
```

In a second Cloud Shell tab, keep an eye on the workload:

```bash
kubectl get pods -l app=upgrade-demo -o wide --watch
kubectl get pdb
```

Expected sequence:

| Stage | What you see |
|---|---|
| New node appears | A node with the new version joins and becomes `Ready` |
| Old node cordoned | Old node shows `SchedulingDisabled` |
| Pods rescheduled | Pods move to other nodes; at least 2 stay running because of the PDB |
| Old node removed | The old node disappears from the list |

Follow progress in the console under **Node pool → Work requests**.

### 3.4 Verify the worker nodes

```bash
kubectl get nodes -o wide
```

All nodes should now show the **new version** in the `VERSION` column and the new OS image in `OS-IMAGE`.

---

## Step 4 – (Optional) Upgrade Worker Nodes with Blue/Green

In this method you build a **new pool** at the new version, move workloads, then remove the old pool. It makes rollback easier because the old pool stays available until you are satisfied.

1. **Add node pool**: name `pool-new`, shape `VM.Standard.E5.Flex` (1 OCPU / 8 GB), 2 nodes, **Kubernetes version = the new version**, matching image, same subnet and SSH key.
2. Wait until its nodes are `Ready`:

```bash
kubectl get nodes
```

3. Cordon each node in the old pool so nothing new lands there:

```bash
kubectl cordon <old-node-name>
```

4. Drain each old node, one at a time:

```bash
kubectl drain <old-node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60
```

5. Confirm the pods are now running on `pool-new` nodes:

```bash
kubectl get pods -l app=upgrade-demo -o wide
```

6. When everything is healthy, delete the old pool: **Node pools → (old pool) → More actions → Delete node pool**.

| Method | Strengths | Trade-offs |
|---|---|---|
| Node cycling | Simple; fully managed; keeps the same pool | In-place; rollback means another cycle |
| Blue/green | Easy rollback; full control of timing | More manual steps; needs capacity for two pools |

---

## Step 5 – Post-Upgrade Verification

```bash
kubectl version
kubectl get nodes -o wide
kubectl get pods -A
kubectl get pdb
kubectl get events -A --sort-by=.lastTimestamp | tail -20
```

Also check in the console:

1. **Resources → Add-ons**: all add-ons are **Active**; update them if the console offers a newer compatible version.
2. **Node pools**: version matches the control plane for every pool.
3. If you use the Cluster Autoscaler, confirm its version matches the new Kubernetes minor version, then check its logs:

```bash
kubectl -n kube-system logs deployment/cluster-autoscaler --tail=30
```

---

## Validation Checklist

- [ ] Version and node state recorded before the upgrade
- [ ] Sample workload and PodDisruptionBudget created
- [ ] Control plane upgraded and cluster is **Active**
- [ ] Node pool version and image updated
- [ ] Nodes cycled; every node shows the new version
- [ ] Workload stayed available during the upgrade
- [ ] Add-ons and autoscaler checked after the upgrade

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| No upgrade option on the cluster | Already on the newest version, or another work request is running | Check supported versions and wait for work requests to finish |
| Cannot select the new version for the node pool | Control plane not upgraded yet | Upgrade the control plane first |
| Node cycling stalls | PodDisruptionBudget blocks eviction, or no spare capacity | Check `kubectl get pdb -A`; relax the budget or add nodes |
| Cycling fails with a capacity error | No host capacity for surge nodes in the AD | Use a capacity reservation, another AD, or lower the surge |
| Nodes stay on the old version | Version edited but nodes not cycled or replaced | Run **Cycle nodes** or use blue/green |
| Pods fail after the upgrade | Removed or deprecated API versions | Update manifests to supported API versions |
| `kubectl` version warning | Client too far from server version | Install a client within one minor version |
| Drain hangs | Unmanaged pods or local storage | Use the drain flags shown above; check the pod's owner |

---

## Clean-up

```bash
kubectl delete deployment upgrade-demo --ignore-not-found
kubectl delete pdb upgrade-demo-pdb --ignore-not-found
```

If you created `pool-new` only for the lab, delete it in the console. Remove any extra node pools to avoid charges.

---

## Review Questions

1. Why must the control plane be upgraded before the node pools?
2. How many minor versions can you upgrade at a time?
3. Why does editing the node pool's Kubernetes version not change existing nodes?
4. What do maximum surge and maximum unavailable control during node cycling?
5. How does a PodDisruptionBudget protect your application during an upgrade?
6. When would you choose blue/green over node cycling?
