# Lab: Autoscaling Kubernetes Node Pools and Pods on OKE

**Duration:** ~60–75 minutes
**Platform:** Oracle Container Engine for Kubernetes (OKE), existing cluster

---

## Lab Objectives

By the end of this lab you will be able to:

1. Deploy and configure the **Cluster Autoscaler** to scale an OKE node pool.
2. Deploy the **Kubernetes Metrics Server** with `kubectl`.
3. Create a **Horizontal Pod Autoscaler (HPA)** that scales pods on CPU.
4. Watch pod scaling trigger node scaling end to end.

## Concepts at a Glance

| Component | Scales | Trigger | Needs |
|---|---|---|---|
| Cluster Autoscaler (CA) | **Nodes** in a node pool | Pods stuck `Pending` for lack of resources; underused nodes | OCI permissions, min/max node counts |
| Metrics Server | Nothing (provides data) | n/a | Kubelet access on port 10250 |
| Horizontal Pod Autoscaler (HPA) | **Pods** (replicas) | CPU/memory vs. target utilization | Metrics Server and pod resource **requests** |

**Flow in this lab:** load increases → HPA adds pods → pods can't fit → Cluster Autoscaler adds a node. When load drops, the reverse happens.

## Prerequisites

- An existing OKE cluster with at least one **node pool** (this lab assumes a pool of 1–2 nodes).
- `kubectl` configured for the cluster (`oci ce cluster create-kubeconfig ...`).
- OCI CLI configured (optional, used for verification).
- Permission to create IAM dynamic groups and policies (or ask your tenancy admin).

Verify access:

```bash
kubectl get nodes -o wide
kubectl cluster-info
```

Record these values; you will need them:

```bash
export COMPARTMENT_OCID="ocid1.compartment.oc1..xxxx"
export NODEPOOL_OCID="ocid1.nodepool.oc1.<region>.xxxx"
export REGION="<your-region-key>"        # e.g. ap-mumbai-1
```

Find the node pool OCID in the Console under **Developer Services → Kubernetes Clusters (OKE) → your cluster → Node pools**, or:

```bash
oci ce node-pool list --compartment-id $COMPARTMENT_OCID --query 'data[].{name:name,id:id}' --output table
```

---

## Part 1: Working with the Cluster Autoscaler

### Step 1.1: Check Your Kubernetes Version

The autoscaler image should match your cluster's minor version.

```bash
kubectl version
```

Note the server version (for example `v1.30.x`). Browse the Cluster Autoscaler releases and pick the OCI image tag matching your minor version.

> **Alternative:** On **Enhanced** clusters, OKE offers Cluster Autoscaler as a managed **add-on** (Cluster → Add-ons). The manual deployment below shows what the add-on does under the hood and works on any cluster type.

### Step 1.2: Create IAM Dynamic Group and Policies (Instance Principal)

The autoscaler runs in the cluster and calls OCI APIs to resize the node pool, so it needs permission.

**Dynamic group** (for example `oke-autoscaler-dg`), matching rule:

```
ALL {instance.compartment.id = '<COMPARTMENT_OCID>'}
```

**Policy** (for example `oke-autoscaler-policy`):

```
Allow dynamic-group oke-autoscaler-dg to manage cluster-node-pools in compartment <compartment-name>
Allow dynamic-group oke-autoscaler-dg to manage instance-family in compartment <compartment-name>
Allow dynamic-group oke-autoscaler-dg to use subnets in compartment <compartment-name>
Allow dynamic-group oke-autoscaler-dg to read virtual-network-family in compartment <compartment-name>
Allow dynamic-group oke-autoscaler-dg to use vnics in compartment <compartment-name>
Allow dynamic-group oke-autoscaler-dg to inspect compartments in compartment <compartment-name>
```

> If your cluster uses **Workload Identity**, use a workload-identity policy for the `cluster-autoscaler` service account in `kube-system` instead, and set the matching env var in the Deployment.

### Step 1.3: Download the Autoscaler Manifest

Start from the upstream OCI example manifest (includes ServiceAccount, RBAC, and Deployment):

```bash
curl -L -o cluster-autoscaler.yaml \
  https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/oci/examples/oci-nodepool-cluster-autoscaler-w-principals.yaml
```

If that URL has moved, copy the file from the `cluster-autoscaler/cloudprovider/oci/examples/` folder of the `kubernetes/autoscaler` GitHub repository.

### Step 1.4: Edit the Manifest

Open `cluster-autoscaler.yaml` and update the Deployment container section:

```yaml
containers:
  - image: iad.ocir.io/oracle/oci-cluster-autoscaler:<TAG-MATCHING-YOUR-K8S-VERSION>
    name: cluster-autoscaler
    command:
      - ./cluster-autoscaler
      - --v=4
      - --stderrthreshold=info
      - --cloud-provider=oci-oke
      - --max-node-provision-time=25m
      - --nodes=1:4:<NODEPOOL_OCID>          # min:max:nodepool-ocid
      - --scale-down-delay-after-add=10m
      - --scale-down-unneeded-time=10m
      - --unremovable-node-recheck-timeout=5m
      - --balance-similar-node-groups
      - --balancing-ignore-label=displayName
      - --balancing-ignore-label=hostname
      - --balancing-ignore-label=internal_addr
      - --balancing-ignore-label=oci.oraclecloud.com/fault-domain
    imagePullPolicy: "Always"
    env:
      - name: OKE_USE_INSTANCE_PRINCIPAL
        value: "true"
      - name: OCI_SDK_APPEND_USER_AGENT
        value: "oci-oke-cluster-autoscaler"
```

Key settings:

- `--nodes=1:4:<OCID>`: the pool will never go below 1 or above 4 nodes.
- `--scale-down-unneeded-time`: how long a node must be underused before removal. Lower values make the demo faster (the default is 10m).
- Make sure the Deployment has `nodeSelector` / tolerations that let it run on a worker node (the example manifest already handles this).

> Make sure your pool's maximum is within your compartment's compute service limits.

### Step 1.5: Deploy the Cluster Autoscaler

```bash
kubectl apply -f cluster-autoscaler.yaml
kubectl -n kube-system get pods -l app=cluster-autoscaler
kubectl -n kube-system logs -f deployment/cluster-autoscaler
```

**Expected:** the pod is `Running`, and logs show the node group discovered with your min/max values, with no authorization errors. Press `Ctrl+C` to stop the log stream.

### Step 1.6: Trigger a Node Scale-Out

Create a deployment whose pods cannot all fit on the current nodes. Each pod requests a large amount of CPU.

```bash
cat <<'EOF' > ca-test.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ca-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ca-test
  template:
    metadata:
      labels:
        app: ca-test
    spec:
      containers:
        - name: pause
          image: registry.k8s.io/pause:3.9
          resources:
            requests:
              cpu: "700m"
              memory: "512Mi"
EOF
kubectl apply -f ca-test.yaml
```

Check how much free CPU a node has, then scale beyond that:

```bash
kubectl describe nodes | grep -A6 "Allocated resources"
kubectl scale deployment ca-test --replicas=8
```

Watch pods and nodes:

```bash
kubectl get pods -l app=ca-test -w          # some pods show Pending
kubectl get nodes -w                         # a new node appears (takes several minutes)
```

Confirm the autoscaler's decision:

```bash
kubectl describe pod <pending-pod-name> | grep -A5 Events
kubectl -n kube-system logs deployment/cluster-autoscaler | grep -i "scale-up"
kubectl -n kube-system get configmap cluster-autoscaler-status -o yaml
```

**Expected:** pending pods trigger a scale-up (events such as `TriggeredScaleUp`), a new worker node joins (`Ready`), and all pods become `Running`.

### Step 1.7: Trigger a Node Scale-In

```bash
kubectl scale deployment ca-test --replicas=1
kubectl get nodes -w
```

After the `--scale-down-unneeded-time` period (plus the post-scale-up delay), the extra node is cordoned, drained and removed. Check:

```bash
kubectl -n kube-system logs deployment/cluster-autoscaler | grep -i "scale-down"
```

### Step 1.8: Clean Up Part 1 Test Workload

```bash
kubectl delete -f ca-test.yaml
```

Leave the Cluster Autoscaler running for Part 3.

---

## Part 2: Kubernetes Metrics Server

### Overview

The **Metrics Server** is a cluster-wide aggregator of resource usage (CPU and memory). It scrapes each kubelet and exposes the data through the Kubernetes **Metrics API** (`metrics.k8s.io`). It powers:

- `kubectl top nodes` and `kubectl top pods`
- The Horizontal Pod Autoscaler (and the Vertical Pod Autoscaler)

It is meant for autoscaling and quick checks, **not** as a long-term monitoring store (use Prometheus or OCI Monitoring for that).

```
kubelet (each node) ──► Metrics Server ──► Metrics API ──► kubectl top / HPA
```

### Step 2.1: Check Whether It Is Already Installed

```bash
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
```

If `kubectl top nodes` returns data, skip to Part 3. If you get `error: Metrics API not available`, continue.

### Step 2.2: Deploy the Metrics Server with kubectl

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Verify:

```bash
kubectl get deployment metrics-server -n kube-system
kubectl get pods -n kube-system -l k8s-app=metrics-server
kubectl get apiservice v1beta1.metrics.k8s.io
```

**Expected:** the deployment is `1/1 READY` and the APIService shows `AVAILABLE: True`. This can take a minute or two.

### Step 2.3: Verify Metrics Are Flowing

```bash
kubectl top nodes
kubectl top pods -A
```

You should see CPU (cores / millicores) and memory for each node and pod.

### Step 2.4: Troubleshooting

If the pod is running but not `Ready`, or `kubectl top` fails, check logs:

```bash
kubectl -n kube-system logs deployment/metrics-server
```

Common OKE fixes:

| Symptom | Fix |
|---|---|
| `x509: cannot validate certificate for <ip>` | Add `--kubelet-insecure-tls` to the container args (acceptable for a lab; for production prefer properly signed kubelet serving certs). |
| `unable to fully scrape metrics ... no route / timeout` | Ensure the **worker node security list / NSG** allows the control plane (Kubernetes API endpoint) to reach workers on **TCP 10250**. |
| Hostname resolution errors | Add `--kubelet-preferred-address-types=InternalIP`. |

Apply an arg patch if needed:

```bash
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"},
  {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-preferred-address-types=InternalIP"}
]'
kubectl rollout status deployment/metrics-server -n kube-system
```

---

## Part 3: Horizontal Pod Autoscaler (HPA)

### Overview

The HPA adjusts the replica count of a Deployment (or StatefulSet) to keep a metric near a target. For CPU:

```
desiredReplicas = ceil( currentReplicas × currentUtilization / targetUtilization )
```

Utilization is measured **relative to the pod's CPU request**, so every container **must define `resources.requests`**, otherwise HPA shows `<unknown>`.

### Step 3.1: Deploy a Sample Application

```bash
cat <<'EOF' > php-apache.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
        - name: php-apache
          image: registry.k8s.io/hpa-example
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 500m
              memory: 128Mi
            limits:
              cpu: 1000m
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
spec:
  selector:
    app: php-apache
  ports:
    - port: 80
      targetPort: 80
EOF
kubectl apply -f php-apache.yaml
kubectl get pods -l app=php-apache
```

### Step 3.2: Create the HPA

Imperative method:

```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

Or declaratively (recommended; delete the imperative one first if you used it):

```bash
kubectl delete hpa php-apache --ignore-not-found
cat <<'EOF' > php-apache-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60
EOF
kubectl apply -f php-apache-hpa.yaml
kubectl get hpa php-apache
```

**Expected:** `TARGETS` shows something like `0%/50%` (it may show `<unknown>` for the first minute).

### Step 3.3: Generate Load

Open a **second terminal** and watch the HPA:

```bash
kubectl get hpa php-apache -w
```

In the first terminal, start a load generator:

```bash
kubectl run load-generator --image=busybox:1.36 --restart=Never -it --rm -- \
  /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
```

For more load, run the same command in additional terminals with different pod names (`load-generator-2`, and so on).

### Step 3.4: Observe Pod Scaling

In another terminal:

```bash
kubectl get pods -l app=php-apache -w
kubectl top pods -l app=php-apache
kubectl describe hpa php-apache
```

**Expected:** CPU utilization climbs above 50%, the HPA increases replicas (for example 1 → 3 → 5 …), and events show `SuccessfulRescale`.

### Step 3.5: Observe the Two Autoscalers Working Together

Because each replica requests 500m CPU, a larger replica count will exceed free capacity on your node(s):

```bash
kubectl get pods -l app=php-apache        # some pods Pending
kubectl get nodes -w                      # Cluster Autoscaler adds a node
kubectl -n kube-system logs deployment/cluster-autoscaler | grep -i "scale-up"
```

**Expected sequence:**

1. Load rises → HPA raises replicas.
2. Replicas can't be scheduled → pods go `Pending`.
3. Cluster Autoscaler scales the node pool up → new node joins.
4. Pending pods are scheduled and become `Running`.

### Step 3.6: Stop the Load and Watch Scale-Down

Press `Ctrl+C` in the load-generator terminal (the pod is removed because of `--rm`). Then:

```bash
kubectl get hpa php-apache -w
kubectl get pods -l app=php-apache -w
kubectl get nodes -w
```

**Expected:**

- After the stabilization window, the HPA reduces replicas back to `minReplicas`.
- After the unneeded-time period, the Cluster Autoscaler removes the extra node(s).

> Scale-down is intentionally slow. Be patient: HPA stabilization plus CA delay can take 10–15 minutes.

---

## Verification Checklist

- [ ] Cluster Autoscaler pod is `Running` in `kube-system`.
- [ ] A scale-up added a node when pods were `Pending`.
- [ ] A scale-down removed the extra node after load dropped.
- [ ] `kubectl top nodes` and `kubectl top pods` return data.
- [ ] HPA shows a numeric `TARGETS` value (not `<unknown>`).
- [ ] HPA increased and then decreased replicas under load.

## Troubleshooting Quick Reference

| Problem | Likely Cause | Action |
|---|---|---|
| CA logs show `NotAuthorizedOrNotFound` | Missing IAM policy or wrong dynamic group | Recheck Step 1.2; confirm the node pool OCID. |
| CA doesn't scale up | Pod doesn't fit even on a new node, or max reached | Check pod requests vs. node shape; raise max in `--nodes`. |
| CA doesn't scale down | Pods with local storage, PodDisruptionBudgets, or non-controller pods | Check logs for the "not removable" reason. |
| New node slow to appear | Instance provisioning time | Allow 3–7 minutes; raise `--max-node-provision-time` if needed. |
| HPA `<unknown>` target | Metrics Server unavailable, or missing CPU requests | `kubectl top pods`; ensure `resources.requests.cpu` is set. |
| HPA doesn't scale up | Load too low or utilization is under target | Add more load generators or lower the target. |
| `Metrics API not available` | Metrics Server not ready, or port 10250 blocked | See Step 2.4. |

## Cleanup

```bash
kubectl delete hpa php-apache
kubectl delete -f php-apache.yaml
kubectl delete pod load-generator --ignore-not-found
kubectl delete -f ca-test.yaml --ignore-not-found

# Optional: remove autoscaler and metrics server
kubectl delete -f cluster-autoscaler.yaml
kubectl delete -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

If you want the node pool to return to its original size afterwards, set the pool size back in the OKE Console (Node pool → Scale) or:

```bash
oci ce node-pool update --node-pool-id $NODEPOOL_OCID --size 1
```

Also remove the dynamic group and policy from Step 1.2 if they were created only for this lab.

## Summary

| Layer | Tool | What you did |
|---|---|---|
| Nodes | Cluster Autoscaler | Resized the OKE node pool based on pending pods and idle nodes |
| Data | Metrics Server | Provided CPU/memory metrics via the Metrics API |
| Pods | HPA | Scaled replicas from CPU utilization |

Together, HPA reacts to application load by adding pods, and the Cluster Autoscaler makes sure there is enough node capacity to run them.
