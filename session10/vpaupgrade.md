# Lab: Vertical Pod Autoscaling, Cluster Upgrades, DNS and Observability on OKE

**Duration:** ~90–120 minutes
**Platform:** Oracle Container Engine for Kubernetes (OKE), existing cluster

---

## Lab Objectives

1. Deploy the **Vertical Pod Autoscaler (VPA)** and use its recommendations to right-size pods.
2. Upgrade the **control plane** of an OKE cluster to a newer Kubernetes version.
3. Upgrade the **worker nodes** (node pools) to the new version.
4. Inspect and customize **DNS (CoreDNS)** for the cluster.
5. **Observe** the cluster using `kubectl`, events, logs, metrics and the OCI Console.

## Prerequisites

- An existing OKE cluster with at least one node pool (2+ nodes recommended, so upgrades don't cause downtime).
- `kubectl` and OCI CLI configured for the cluster.
- **Metrics Server** installed (see the autoscaling lab). Verify: `kubectl top nodes`.
- `git` and `openssl` on your workstation (VPA install script).
- Permission to manage clusters and node pools in the compartment.

```bash
export CLUSTER_OCID="ocid1.cluster.oc1.<region>.xxxx"
export NODEPOOL_OCID="ocid1.nodepool.oc1.<region>.xxxx"
export COMPARTMENT_OCID="ocid1.compartment.oc1..xxxx"

kubectl get nodes -o wide
kubectl version
```

---

## Part 1: Kubernetes Vertical Pod Autoscaler (VPA)

### Overview

Where the HPA changes the **number** of pods, the VPA changes the **CPU and memory requests/limits** of each pod. It has three components:

| Component | Role |
|---|---|
| **Recommender** | Watches usage history and computes recommended requests |
| **Updater** | Evicts pods whose requests differ significantly from the recommendation |
| **Admission Controller** | Mutates new pods to apply the recommended requests |

**Update modes** (`updatePolicy.updateMode`):

| Mode | Behavior |
|---|---|
| `Off` | Only produces recommendations; nothing is changed (safest, good for learning) |
| `Initial` | Applies requests only when a pod is created |
| `Recreate` / `Auto` | Evicts and recreates running pods to apply new values |

> **Important:** Do not run VPA and HPA on the **same resource metric** (for example CPU) for the same workload. They will fight each other.

### Step 1.1: Verify Metrics Server

```bash
kubectl top nodes
```

If this fails, install the Metrics Server first.

### Step 1.2: Install VPA

```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh
```

Verify:

```bash
kubectl get pods -n kube-system | grep vpa
kubectl get crd | grep verticalpodautoscaler
```

**Expected:** `vpa-recommender`, `vpa-updater` and `vpa-admission-controller` pods are `Running`, and the VPA CRDs exist.

### Step 1.3: Deploy a Sample Workload

The pods request much less CPU than they will use, which is a good case for VPA.

```bash
cat <<'EOF' > hamster.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hamster
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hamster
  template:
    metadata:
      labels:
        app: hamster
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
      containers:
        - name: hamster
          image: registry.k8s.io/ubuntu-slim:0.14
          resources:
            requests:
              cpu: 100m
              memory: 50Mi
          command: ["/bin/sh"]
          args:
            - "-c"
            - "while true; do timeout 0.5s yes >/dev/null; sleep 0.5s; done"
EOF
kubectl apply -f hamster.yaml
kubectl get pods -l app=hamster
```

> If the image tag is unavailable, any small image that loops a CPU-burning command works for this demo.

### Step 1.4: Create a VPA in Recommendation Mode

```bash
cat <<'EOF' > hamster-vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: hamster-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hamster
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
      - containerName: "*"
        minAllowed:
          cpu: 50m
          memory: 50Mi
        maxAllowed:
          cpu: 1
          memory: 500Mi
EOF
kubectl apply -f hamster-vpa.yaml
```

Wait 2–5 minutes, then read the recommendations:

```bash
kubectl describe vpa hamster-vpa
kubectl get vpa hamster-vpa -o jsonpath='{.status.recommendation.containerRecommendations}' ; echo
```

**Expected:** a `Recommendation` block with `Lower Bound`, `Target`, `Upper Bound` and `Uncapped Target` values. The **Target** CPU should be well above the 100m requested.

### Step 1.5: Let VPA Apply the Recommendations

Switch to `Auto` mode:

```bash
kubectl patch vpa hamster-vpa --type merge -p '{"spec":{"updatePolicy":{"updateMode":"Auto"}}}'
kubectl get pods -l app=hamster -w
```

The updater evicts the pods (one at a time, subject to availability); replacements are created with updated requests. Confirm:

```bash
kubectl get pod -l app=hamster -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.containers[0].resources.requests}{"\n"}{end}'
kubectl describe pod <hamster-pod> | grep -i "vpa"      # annotation: vpaUpdates
kubectl get events --sort-by=.lastTimestamp | grep -i evict
```

> VPA needs at least **2 replicas** (by default) to evict pods without downtime. For single-replica workloads, use `Initial` or `Off`.

### Step 1.6: Clean Up Part 1

```bash
kubectl delete -f hamster-vpa.yaml -f hamster.yaml
# Optional: remove VPA entirely
cd autoscaler/vertical-pod-autoscaler && ./hack/vpa-down.sh
```

---

## Part 2: Upgrading Clusters to Newer Kubernetes Versions

### Overview

Upgrade order on OKE is always:

1. **Control plane** (the cluster itself)
2. **Worker nodes** (each node pool)

Rules to remember:

- Upgrade **one minor version at a time** (for example 1.30 → 1.31, not 1.30 → 1.32).
- Worker nodes may lag the control plane only within the version skew OKE supports. Never run workers *newer* than the control plane.
- Control plane upgrade does **not** restart your pods. The API is briefly unavailable for changes, but running workloads continue.
- Upgrades can't be rolled back, so prepare carefully.

### Step 2.1: Pre-Upgrade Checks

```bash
# Current versions
kubectl version
kubectl get nodes -o wide

# Unhealthy workloads
kubectl get pods -A | grep -v -E "Running|Completed"
kubectl get nodes

# PodDisruptionBudgets that could block drains
kubectl get pdb -A

# Deprecated API usage (look for requests to deprecated/removed APIs)
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis
```

Checklist before you proceed:

- [ ] Read the Kubernetes release notes and the OKE release notes for the target version.
- [ ] Workloads don't use APIs removed in the target version (scan manifests with a tool such as `pluto` or `kubent`).
- [ ] Applications run **2+ replicas** and have PDBs where appropriate.
- [ ] Add-ons and controllers (ingress controller, CSI, autoscaler, metrics server) support the target version.
- [ ] Manifests and Helm values are in Git, so you can recreate resources if needed.
- [ ] **Pause the Cluster Autoscaler** during node upgrades:

```bash
kubectl -n kube-system scale deployment cluster-autoscaler --replicas=0
```

### Step 2.2: List Available Upgrade Versions

Console: **Developer Services → Kubernetes Clusters (OKE) → your cluster**. If a newer version is available, an **Upgrade available** link appears on the cluster details.

CLI:

```bash
oci ce cluster get --cluster-id $CLUSTER_OCID \
  --query 'data."kubernetes-version"'

oci ce cluster-options get --cluster-option-id all \
  --query 'data."kubernetes-versions"'
```

Pick the next minor version, for example:

```bash
export TARGET_VERSION="v1.xx.x"      # replace with a version from the list above
```

### Step 2.3: Upgrade the Control Plane

**Console:** Cluster details → **Upgrade available** → choose the version → **Upgrade**.

**CLI:**

```bash
oci ce cluster update --cluster-id $CLUSTER_OCID --kubernetes-version $TARGET_VERSION
```

Track progress (takes several minutes):

```bash
oci ce work-request list --compartment-id $COMPARTMENT_OCID --cluster-id $CLUSTER_OCID \
  --query 'data[0].{op:"operation-type",status:status,pct:"percent-complete"}'
oci ce cluster get --cluster-id $CLUSTER_OCID --query 'data."lifecycle-state"'
```

The cluster state shows `UPDATING`, then `ACTIVE`.

Verify:

```bash
kubectl version                  # Server Version should match the target
kubectl get nodes                # nodes still show the OLD version (expected)
kubectl get pods -A | grep -v -E "Running|Completed"
```

### Step 2.4: Post-Control-Plane Validation

```bash
kubectl get componentstatuses 2>/dev/null
kubectl -n kube-system get pods
kubectl get --raw='/readyz?verbose' | tail -n 5
```

Confirm applications still respond before touching worker nodes.

---

## Part 3: Upgrading the Kubernetes Version on Worker Nodes

### Overview

Updating the node pool's Kubernetes version changes the version for **new** nodes only. Existing nodes keep running the old version until they are replaced. There are two ways to replace them:

| Method | How | Best for |
|---|---|---|
| **Node cycling** (managed) | OKE replaces nodes in a rolling fashion with a configured surge/unavailable | Most clusters |
| **Manual replacement** | Add new nodes, cordon and drain old nodes, delete old nodes | Maximum control |

### Step 3.1: Set the New Version on the Node Pool

**Console:** Node pool details → **Edit** → set **Kubernetes version** to the target (and optionally a newer node image) → save.

**CLI:**

```bash
oci ce node-pool update --node-pool-id $NODEPOOL_OCID --kubernetes-version $TARGET_VERSION
```

> Existing nodes are unaffected so far.

### Step 3.2A: Method 1 — Node Cycling

Console: Node pool → **Cycle nodes** (choose maximum surge and maximum unavailable, for example surge 1, unavailable 0).

CLI:

```bash
oci ce node-pool update --node-pool-id $NODEPOOL_OCID \
  --kubernetes-version $TARGET_VERSION \
  --node-pool-cycling-details '{"isNodeCyclingEnabled": true, "maximumSurge": "1", "maximumUnavailable": "0"}'
```

OKE creates a new node at the new version, cordons and drains an old node, then terminates it, repeating for each node. Watch:

```bash
kubectl get nodes -o wide -w
kubectl get pods -A -o wide | grep -v Running
```

### Step 3.2B: Method 2 — Manual Replacement

1. **Add capacity** (new nodes use the new version):

```bash
oci ce node-pool update --node-pool-id $NODEPOOL_OCID --size <current-size + 1>
kubectl get nodes -w          # wait until the new node shows the new version and Ready
```

2. **Cordon and drain** an old node:

```bash
kubectl get nodes
kubectl cordon <old-node-name>
kubectl drain <old-node-name> --ignore-daemonsets --delete-emptydir-data
```

If the drain stalls, check PDBs (`kubectl get pdb -A`) and `kubectl get pods -A -o wide | grep <old-node-name>`.

3. **Delete the old node** and decrement the pool size:

```bash
oci ce node-pool delete-node --node-pool-id $NODEPOOL_OCID \
  --node-id <node-ocid> --is-decrement-size true
```

Find the node OCID with `oci ce node-pool get --node-pool-id $NODEPOOL_OCID --query 'data.nodes[].{name:name,id:id,version:"kubernetes-version"}'`.

4. Repeat for each remaining old node, one at a time.

### Step 3.3: Verify

```bash
kubectl get nodes -o wide
# All nodes show the target version in the VERSION column

oci ce node-pool get --node-pool-id $NODEPOOL_OCID \
  --query 'data.nodes[].{name:name,version:"kubernetes-version",state:"lifecycle-state"}' --output table

kubectl get pods -A | grep -v -E "Running|Completed"
```

Resume the autoscaler:

```bash
kubectl -n kube-system scale deployment cluster-autoscaler --replicas=1
```

Repeat Part 3 for every other node pool in the cluster.

---

## Part 4: Configuring DNS Servers for Kubernetes Clusters

### Overview

Pods resolve names through **CoreDNS**, which runs in `kube-system` and is exposed by the `kube-dns` Service.

```
Pod → kube-dns Service (CoreDNS) →  cluster.local names answered directly
                                 →  everything else forwarded to upstream resolvers
                                    (by default, the node's resolv.conf / VCN resolver)
```

On OKE the VCN resolver handles VCN hostnames and public names. To resolve private zones (for example an on-premises domain), you add a **stub domain** in CoreDNS or configure a private resolver in the VCN.

### Step 4.1: Inspect the Current DNS Setup

```bash
kubectl -n kube-system get deployment coredns
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system get svc kube-dns
kubectl -n kube-system get configmap coredns -o yaml
```

Look at the `Corefile`; the key pieces are `kubernetes cluster.local ...` and `forward . /etc/resolv.conf`.

### Step 4.2: Test Resolution from a Pod

```bash
kubectl run dnstest --image=registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3 \
  --restart=Never --command -- sleep 3600

kubectl exec dnstest -- cat /etc/resolv.conf
kubectl exec dnstest -- nslookup kubernetes.default
kubectl exec dnstest -- nslookup kube-dns.kube-system.svc.cluster.local
kubectl exec dnstest -- nslookup oracle.com
```

**Expected:** the `nameserver` in `resolv.conf` is the kube-dns ClusterIP, and all three lookups succeed.

### Step 4.3: Add a Custom DNS Server (Stub Domain)

> **Back up first**, and note that OKE manages CoreDNS, so confirm your change survives cluster/add-on upgrades. If you use the CoreDNS cluster add-on, prefer its configuration options where available.

```bash
kubectl -n kube-system get configmap coredns -o yaml > coredns-backup.yaml
kubectl -n kube-system edit configmap coredns
```

Add a server block **for your private domain** to the Corefile (replace the domain and IP with your DNS server):

```
corp.example.com:53 {
    errors
    cache 30
    forward . 10.0.0.53
}
```

Optionally replace the default upstream for all other names:

```
forward . 10.0.0.53 169.254.169.254
```

CoreDNS usually reloads automatically (the `reload` plugin). To force it:

```bash
kubectl -n kube-system rollout restart deployment coredns
kubectl -n kube-system rollout status deployment coredns
```

Test:

```bash
kubectl exec dnstest -- nslookup host.corp.example.com
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=30
```

> The DNS server must be reachable from the **worker node subnet** (route tables, security lists/NSGs allowing UDP/TCP 53).

### Step 4.4: Per-Pod DNS Settings (Optional)

For a single workload, override DNS without touching CoreDNS:

```yaml
spec:
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 10.0.0.53
    searches:
      - corp.example.com
    options:
      - name: ndots
        value: "2"
```

### Step 4.5: Scale and Tune CoreDNS (Optional)

```bash
kubectl -n kube-system scale deployment coredns --replicas=3
kubectl -n kube-system top pods -l k8s-app=kube-dns
```

For large clusters, consider the DNS autoscaler and NodeLocal DNSCache to reduce latency and load.

### Step 4.6: Roll Back If Needed

```bash
kubectl apply -f coredns-backup.yaml
kubectl -n kube-system rollout restart deployment coredns
```

### Step 4.7: Clean Up

```bash
kubectl delete pod dnstest
```

---

## Part 5: Observing Kubernetes Clusters

### Step 5.1: Cluster and Node Health

```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl describe node <node-name>          # Conditions, Capacity, Allocatable, Allocated resources
kubectl top nodes
kubectl get --raw='/readyz?verbose' | tail -n 10
```

Things to read in `describe node`: `Conditions` (MemoryPressure, DiskPressure, PIDPressure, Ready), taints, and "Allocated resources".

### Step 5.2: Workload Health

```bash
kubectl get pods -A -o wide
kubectl get pods -A --field-selector=status.phase!=Running
kubectl get deploy,sts,ds -A
kubectl top pods -A --sort-by=cpu | head
kubectl top pods -A --sort-by=memory | head
```

### Step 5.3: Events

Events explain most scheduling and startup problems.

```bash
kubectl get events -A --sort-by=.lastTimestamp | tail -n 30
kubectl get events -A --field-selector type=Warning
kubectl get events -w
```

### Step 5.4: Logs

```bash
kubectl logs <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --previous          # crashed container
kubectl logs -l app=<label> -n <namespace> --tail=50 -f
kubectl -n kube-system logs deployment/coredns
```

### Step 5.5: Troubleshooting a Failing Pod

```bash
kubectl run broken --image=nginx:doesnotexist
kubectl get pod broken                     # ErrImagePull / ImagePullBackOff
kubectl describe pod broken                # read the Events section
kubectl delete pod broken
```

Typical status → first check:

| Status | First check |
|---|---|
| `Pending` | `describe pod` events: insufficient resources, taints, PVC unbound |
| `ImagePullBackOff` | Image name/tag, registry credentials, network access |
| `CrashLoopBackOff` | `logs --previous`, probes, config/secrets |
| `OOMKilled` | Raise the memory limit (see the VPA recommendation) |

### Step 5.6: OCI-Level Observability

- **Console → cluster details:** cluster state, work requests (create, upgrade, scale operations) and node pool status.
- **Work requests (CLI):**

```bash
oci ce work-request list --compartment-id $COMPARTMENT_OCID --cluster-id $CLUSTER_OCID --output table
```

- **OCI Monitoring:** use the Metrics Explorer to chart metrics for the cluster's compute instances (CPU, memory, network) and, where available, OKE control-plane metrics. Create **alarms** for sustained node CPU or memory pressure.
- **OCI Logging / Audit:** use Audit to see who changed the cluster or node pools.

### Step 5.7: Collect a Diagnostic Snapshot (Optional)

```bash
kubectl cluster-info dump --output-directory=./cluster-dump --all-namespaces
ls ./cluster-dump
```

---

## Verification Checklist

- [ ] VPA components are running and `hamster-vpa` shows recommendations.
- [ ] VPA in `Auto` mode recreated pods with updated requests.
- [ ] Control plane is at the target Kubernetes version.
- [ ] All worker nodes in the node pool show the target version.
- [ ] Cluster Autoscaler is running again after the upgrade.
- [ ] Pods resolve internal and external names; the custom domain resolves after the stub-domain change.
- [ ] You can find node conditions, events and logs for any pod, and you reviewed OCI work requests.

## Troubleshooting Quick Reference

| Problem | Likely Cause | Action |
|---|---|---|
| VPA shows no recommendation | Too little history or Metrics Server issue | Wait a few more minutes; check `kubectl top pods` and the recommender logs |
| VPA never evicts pods | Single replica, or mode is `Off` | Use 2+ replicas and `Auto` mode |
| Control plane upgrade rejected | Skipping a minor version | Upgrade one minor version at a time |
| Drain hangs | PDB, local storage, or pods not managed by a controller | Check `kubectl get pdb -A`; use `--delete-emptydir-data`; handle bare pods manually |
| Pods Pending after drain | Not enough capacity | Add nodes before draining |
| Nodes still show the old version | Only the pool setting changed | Cycle or replace the nodes |
| DNS custom zone fails | Corefile syntax, or port 53 blocked | Check CoreDNS logs and security lists/NSGs |
| CoreDNS config reverted | Managed by OKE or add-on | Reapply after upgrades, or use add-on configuration |

## Cleanup

```bash
kubectl delete -f hamster-vpa.yaml -f hamster.yaml --ignore-not-found
kubectl delete pod dnstest broken --ignore-not-found
rm -rf ./cluster-dump
```

Restore any pool sizes, DNS changes, or autoscaler replicas you altered for the lab.

## Summary

| Topic | Key Takeaway |
|---|---|
| VPA | Right-sizes requests per pod; start with `Off`, avoid mixing with HPA on the same metric |
| Cluster upgrade | One minor version at a time, control plane first, no rollback |
| Worker node upgrade | Update the node pool version, then cycle or replace nodes |
| DNS | CoreDNS handles cluster names and forwards the rest; add stub domains for private zones |
| Observability | Use nodes, pods, events, logs and OCI work requests/monitoring together |
