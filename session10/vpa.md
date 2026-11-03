# Lab: Vertical Pod Autoscaler (VPA) on OKE

**Duration:** ~30-40 minutes
**Platform:** Oracle Container Engine for Kubernetes (OKE), existing cluster

---

## Lab Objectives

By the end of this lab you will be able to:

1. Deploy the **Vertical Pod Autoscaler** components with `kubectl`.
2. Create a VPA object in recommendation-only mode (`Off`) and read its recommendations.
3. Switch to automatic mode and watch VPA resize pod CPU and memory requests.
4. Constrain VPA with `minAllowed`, `maxAllowed` and `controlledResources`.

## Concepts at a Glance

| Component | Scales | Trigger | Needs |
|---|---|---|---|
| HPA | Pod **count** (replicas) | CPU/memory vs. target | Metrics Server, pod requests |
| VPA | Pod **size** (requests/limits) | Observed usage history | Metrics Server, VPA components |
| Cluster Autoscaler | **Nodes** | Pending pods / idle nodes | OCI permissions |

VPA has three components, all installed in `kube-system`:

| Component | Role |
|---|---|
| **Recommender** | Watches usage (from Metrics Server) and computes recommended requests |
| **Updater** | Evicts pods whose requests differ too much from the recommendation |
| **Admission Controller** | Mutating webhook that sets the new requests on pods as they are (re)created |

### Update modes

| `updateMode` | Behavior |
|---|---|
| `Off` | Only computes recommendations; changes nothing. Safe for production analysis. |
| `Initial` | Applies recommendations only when a pod is created; never evicts running pods. |
| `Recreate` | Evicts pods to apply new requests. |
| `Auto` | Currently behaves like `Recreate` (may use in-place resize on newer Kubernetes and VPA versions). |

> **Important:** Do not run VPA and HPA on the **same metric** (CPU or memory) for the same workload. They will fight each other. VPA on CPU/memory with HPA on a custom or external metric is fine.

## Prerequisites

- An existing OKE cluster with at least 2 worker nodes' worth of spare capacity (a few hundred millicores is enough).
- `kubectl` configured for the cluster.
- `git` and `openssl` on the machine where you run the install script.
- **Metrics Server** running:

```bash
kubectl top nodes
```

If this returns `error: Metrics API not available`, install it first:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl rollout status deployment/metrics-server -n kube-system
```

(If it fails with certificate or scrape errors on OKE, add `--kubelet-insecure-tls` and `--kubelet-preferred-address-types=InternalIP` to the Metrics Server args, and ensure TCP 10250 is open from the control plane to workers.)

---

## Part 1: Install the VPA

### Step 1.1: Clone the Autoscaler Repository

```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
```

### Step 1.2: Run the Install Script

```bash
./hack/vpa-up.sh
```

The script creates the CRDs, RBAC, the three VPA deployments, and the TLS certificates for the admission webhook.

> **Short-name enforcement:** If your nodes enforce short-name mode (you saw `short name mode is enforcing` earlier) and a VPA pod shows `ImageInspectError`, check the image names in the deployments and use fully qualified names (for example `registry.k8s.io/autoscaling/vpa-recommender:<tag>`).

### Step 1.3: Verify

```bash
kubectl get pods -n kube-system | grep vpa
kubectl get crd | grep verticalpodautoscaler
```

**Expected:** three pods `Running`: `vpa-recommender`, `vpa-updater`, `vpa-admission-controller`, plus the CRDs `verticalpodautoscalers.autoscaling.k8s.io` and `verticalpodautoscalercheckpoints.autoscaling.k8s.io`.

Troubleshooting:

```bash
kubectl -n kube-system logs deployment/vpa-recommender
kubectl -n kube-system logs deployment/vpa-admission-controller
```

---

## Part 2: Deploy a Sample Workload

The workload burns CPU in a loop with deliberately **too-small** requests, so VPA has something to correct. We use 2 replicas because the VPA updater only evicts pods when at least 2 replicas exist (default `--min-replicas=2`).

### Step 2.1: Create the Deployment

```bash
cat <<'EOF' > vpa-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vpa-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vpa-demo
  template:
    metadata:
      labels:
        app: vpa-demo
    spec:
      containers:
        - name: burner
          image: docker.io/library/busybox:1.36
          command: ["/bin/sh", "-c", "while true; do :; done"]
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
EOF
kubectl apply -f vpa-demo.yaml
kubectl get pods -l app=vpa-demo -o wide
```

### Step 2.2: Check Actual Usage vs. Requests

```bash
kubectl top pods -l app=vpa-demo
kubectl get pods -l app=vpa-demo -o custom-columns=NAME:.metadata.name,CPU_REQ:.spec.containers[0].resources.requests.cpu,MEM_REQ:.spec.containers[0].resources.requests.memory
```

**Expected:** each pod uses close to 1000m CPU (a full core, if the node has one free) while requesting only 50m. That gap is what VPA will fix.

---

## Part 3: Recommendation-Only Mode (`Off`)

### Step 3.1: Create the VPA

```bash
cat <<'EOF' > vpa-demo-off.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: vpa-demo
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vpa-demo
  updatePolicy:
    updateMode: "Off"
EOF
kubectl apply -f vpa-demo-off.yaml
```

> The quotes around `"Off"` are required. Unquoted `Off` is parsed by YAML as a boolean.

### Step 3.2: Read the Recommendations

Wait 2-5 minutes for the Recommender to collect data, then:

```bash
kubectl get vpa vpa-demo
kubectl describe vpa vpa-demo
```

Look at the `Recommendation` section:

```
Container Recommendations:
  Container Name:  burner
  Lower Bound:     Cpu: ...   Memory: ...
  Target:          Cpu: ...   Memory: ...
  Uncapped Target: Cpu: ...   Memory: ...
  Upper Bound:     Cpu: ...   Memory: ...
```

| Field | Meaning |
|---|---|
| **Target** | The value VPA would set as the request |
| **Lower Bound** | Minimum reasonable request; below this the pod is likely starved |
| **Upper Bound** | Above this, resources are probably wasted |
| **Uncapped Target** | Target ignoring your `minAllowed`/`maxAllowed` |

**Expected:** the CPU target is far above 50m (close to the observed usage). Nothing has been changed on the pods yet:

```bash
kubectl get pods -l app=vpa-demo -o custom-columns=NAME:.metadata.name,CPU_REQ:.spec.containers[0].resources.requests.cpu
```

---

## Part 4: Automatic Mode

### Step 4.1: Switch to `Auto` with Guardrails

Cap the allowed range so the lab does not exhaust your nodes:

```bash
cat <<'EOF' > vpa-demo-auto.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: vpa-demo
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vpa-demo
  updatePolicy:
    updateMode: "Auto"
    minReplicas: 2
  resourcePolicy:
    containerPolicies:
      - containerName: "*"
        minAllowed:
          cpu: 100m
          memory: 64Mi
        maxAllowed:
          cpu: 500m
          memory: 256Mi
        controlledResources: ["cpu", "memory"]
EOF
kubectl apply -f vpa-demo-auto.yaml
```

### Step 4.2: Watch Pods Get Recreated

```bash
kubectl get pods -l app=vpa-demo -w
```

**Expected:** within a few minutes the Updater evicts the pods one at a time and the Deployment creates replacements. VPA evicts gradually, not all at once.

### Step 4.3: Verify the New Requests

```bash
kubectl get pods -l app=vpa-demo -o custom-columns=NAME:.metadata.name,CPU_REQ:.spec.containers[0].resources.requests.cpu,MEM_REQ:.spec.containers[0].resources.requests.memory
kubectl describe pod -l app=vpa-demo | grep -i -A2 "vpaUpdates\|Annotations"
kubectl get events --field-selector reason=EvictedByVPA
```

**Expected:**

- New pods request CPU **capped at 500m** (your `maxAllowed`) instead of 50m.
- The pods carry an annotation like `vpaUpdates: Pod resources updated by vpa-demo: container 0: cpu request, memory request`.
- Events show `EvictedByVPA`.

Note that the CPU request equals `maxAllowed`, because the uncapped target (~1 core) exceeds it. Check `Uncapped Target` in `kubectl describe vpa vpa-demo` to see the difference.

---

## Part 5: Other Update Modes (Optional)

### Initial mode

Applies recommendations only to **newly created** pods and never evicts running ones:

```bash
kubectl patch vpa vpa-demo --type merge -p '{"spec":{"updatePolicy":{"updateMode":"Initial"}}}'
kubectl delete pod -l app=vpa-demo --wait=false     # replacements get VPA-set requests at creation
kubectl get pods -l app=vpa-demo -w
```

### Control only CPU or only memory

```bash
kubectl patch vpa vpa-demo --type merge -p '{"spec":{"resourcePolicy":{"containerPolicies":[{"containerName":"*","controlledResources":["memory"]}]}}}'
```

### Exclude a container (for sidecars)

```yaml
containerPolicies:
  - containerName: istio-proxy
    mode: "Off"
```

---

## Verification Checklist

- [ ] `vpa-recommender`, `vpa-updater` and `vpa-admission-controller` are `Running`.
- [ ] `kubectl describe vpa vpa-demo` shows a Target recommendation in `Off` mode.
- [ ] In `Auto` mode, pods were evicted and recreated (`EvictedByVPA` events).
- [ ] New pods have larger CPU requests, bounded by `maxAllowed`.

## Troubleshooting Quick Reference

| Problem | Likely Cause | Action |
|---|---|---|
| VPA has no `Recommendation` | Recommender hasn't gathered data yet, or Metrics Server is down | Wait 5 minutes; check `kubectl top pods` and recommender logs. |
| Pods never evicted in `Auto` | Fewer than 2 replicas, or change is below the update threshold | Use 2+ replicas or set `minReplicas: 1` in `updatePolicy`. |
| Pods evicted but requests unchanged | Admission controller not working | Check `kubectl get mutatingwebhookconfiguration vpa-webhook-config` and its logs. |
| New pods stay `Pending` | Recommended requests don't fit on any node | Lower `maxAllowed`, or add node capacity (Cluster Autoscaler). |
| Eviction blocked | A PodDisruptionBudget forbids it | Review PDBs with `kubectl get pdb`. |
| `ImageInspectError` on VPA pods | Short-name enforcement on nodes | Use fully qualified image names in the VPA deployments. |
| YAML error on `updateMode: Off` | Unquoted `Off` parsed as boolean | Write `updateMode: "Off"`. |

## Cleanup

```bash
kubectl delete vpa vpa-demo
kubectl delete -f vpa-demo.yaml

# Remove the VPA components
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-down.sh
```

## Summary

| Layer | Tool | What you did |
|---|---|---|
| Pod size | VPA | Read recommendations, then let VPA resize CPU and memory requests |
| Guardrails | `resourcePolicy` | Bounded VPA with `minAllowed` / `maxAllowed` |
| Modes | `Off` / `Initial` / `Auto` | Compared advisory, creation-time and automatic updates |
