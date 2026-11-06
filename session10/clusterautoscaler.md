# Lab: Autoscaling Kubernetes Node Pools with the Cluster Autoscaler in OKE

**Estimated time:** 60–75 minutes
**Level:** Intermediate

> **Note:** OCI Console labels, add-on configuration keys and policy syntax change over time. Items most likely to differ are marked **(verify)**.

---

## Concept

The **Cluster Autoscaler (CA)** automatically changes the number of worker nodes in a managed node pool:

| Event | CA action |
|---|---|
| Pods are `Pending` because no node has enough free CPU or memory | **Scale up**: adds nodes to the pool (up to the maximum) |
| Nodes are underused and their pods can run elsewhere | **Scale down**: drains and removes nodes (down to the minimum) |

The CA reacts to **pod resource requests**, not to actual CPU usage. Pods without resource requests do not trigger scale-up.

| Autoscaler | Scales | Trigger |
|---|---|---|
| Horizontal Pod Autoscaler (HPA) | Pod replicas | CPU, memory or custom metrics |
| Cluster Autoscaler | Worker nodes | Unschedulable pods, underused nodes |

HPA adds pods; if they cannot be scheduled, the Cluster Autoscaler adds nodes.

## Objective

Enable the Cluster Autoscaler on a managed node pool, trigger a scale-up with pending pods, trigger a scale-down by removing the workload, and test it together with HPA.

## Prerequisites

| Requirement | Details |
|---|---|
| Cluster type | **Enhanced** cluster (for the managed add-on) |
| Node pool | A managed node pool, for example `pool-autoscale` (created in Step 1) |
| Not supported | Virtual nodes |
| Access | `kubectl` through Cloud Shell |
| Service limits | Room for 2–4 extra small flex-shape VMs |

---

## Step 1 – Create a Node Pool for Autoscaling

1. Open **Developer Services → Containers & Artifacts → Kubernetes Clusters (OKE)** and select your cluster.
2. Under **Resources**, click **Node pools → Add node pool**.
3. Configure:

| Field | Value |
|---|---|
| Name | `pool-autoscale` |
| Node type | Managed |
| Kubernetes version | Same as the control plane |
| Worker node subnet | Your worker subnet |
| Shape | `VM.Standard.E5.Flex` |
| OCPUs / Memory | 1 OCPU / 8 GB |
| Number of nodes | 1 |
| SSH public key | Paste your public key |

4. Click **Add** and wait for the pool to become **Active**.
5. Copy the **node pool OCID** from the node pool details page. You need it in Step 3.

```bash
kubectl get nodes -l name=pool-autoscale
```

> **Tip:** If other node pools exist, the test pods in Step 4 may land on them. Use the `nodeSelector` shown in Step 4 so pods target only `pool-autoscale`.

---

## Step 2 – Grant the Autoscaler Permissions

Create a **dynamic group** that contains the worker nodes, then a policy for that group. The add-on's own note in the console says the instances (worker nodes) must be allowed to manage node pools.

1. Open **Identity & Security → Domains → (your domain) → Dynamic groups → Create dynamic group**.
2. Name: `oke-workers-dg`. Matching rule **(verify)**, replacing the OCID with your compartment's OCID:

```
ALL {instance.compartment.id = '<compartment-ocid>'}
```

3. Open **Identity & Security → Policies**, select the cluster's compartment, click **Create Policy**, and name it `oke-cluster-autoscaler-policy`.
4. Add these statements **(verify)**, replacing `<compartment-name>`:

```
Allow dynamic-group oke-workers-dg to manage cluster-node-pools in compartment <compartment-name>
Allow dynamic-group oke-workers-dg to manage instance-family in compartment <compartment-name>
Allow dynamic-group oke-workers-dg to use subnets in compartment <compartment-name>
Allow dynamic-group oke-workers-dg to read virtual-network-family in compartment <compartment-name>
Allow dynamic-group oke-workers-dg to use vnics in compartment <compartment-name>
Allow dynamic-group oke-workers-dg to inspect compartments in compartment <compartment-name>
```

5. Click **Create**.

> If your tenancy uses workload identity instead, the add-on documentation lists the equivalent policy and the matching authentication option. Follow the link in the add-on's warning note ("Working with the Cluster Autoscaler as a Cluster Add-on").

---

## Step 3 – Enable the Cluster Autoscaler Add-on

1. Open the cluster → **Resources → Add-ons → Manage add-ons**.
2. Select **Cluster Autoscaler** and enable it.
3. Choose how the add-on is updated: **Automatic updates** (the add-on updates itself) or **Choose a version** (you pin a specific add-on version). This lab uses **Automatic updates**.
4. Read the warning box. It repeats the `nodes` format and notes that a policy must allow the worker node instances to manage node pools, **plus other policy statements related to initializing worker nodes**. Use its link, "Working with the Cluster Autoscaler as a Cluster Add-on", to confirm the complete list.
5. In the **Configuration** section, open the **Option** dropdown, choose **nodes**, and enter the value **(verify)**:

| Option | Example value | Meaning |
|---|---|---|
| `nodes` | `1:4:<node-pool-ocid>` | `min-num-of-nodes:max-num-of-nodes:OCID-of-nodepool` |

For more than one pool, separate the entries with commas:

```
1:4:<first-nodepool-ocid>,1:3:<second-nodepool-ocid>
```

6. Click **Save changes** and wait for the add-on status to become **Active**.
7. Verify in Cloud Shell:

```bash
kubectl get pods -n kube-system | grep cluster-autoscaler
kubectl logs -n kube-system deployment/cluster-autoscaler --tail=50
```

Expected: the autoscaler pod is `Running` and the logs show it discovering the node pool, with no authorization errors.

> **Important:** Once a pool is managed by the autoscaler, do not resize it manually. The pool size must stay within the `min`–`max` range.

---

## Step 4 – Trigger a Scale-Up

1. Record the current node count:

```bash
kubectl get nodes -l name=pool-autoscale
```

2. Create `scale-test.yaml`. The requests are large enough that six replicas will not fit on one small node:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: scale-test
spec:
  replicas: 6
  selector:
    matchLabels:
      app: scale-test
  template:
    metadata:
      labels:
        app: scale-test
    spec:
      containers:
      - name: pause
        image: registry.k8s.io/pause:3.9
        resources:
          requests:
            cpu: 400m
            memory: 1Gi
```

3. Apply it and observe:

```bash
kubectl apply -f scale-test.yaml
kubectl get pods -l app=scale-test -o wide
kubectl get nodes -l name=pool-autoscale --watch
```

4. In another Cloud Shell tab, follow the autoscaler logs:

```bash
kubectl -n kube-system logs deployment/cluster-autoscaler -f | grep -i "scale"
```

Expected sequence:

| Time | What you see |
|---|---|
| Immediately | Some `scale-test` pods are `Pending` |
| Within about 1–2 minutes | Autoscaler logs a scale-up decision for `pool-autoscale` |
| A few minutes later | New nodes appear and become `Ready` |
| After nodes join | Pending pods move to `Running` |

5. In the console, open **Node pools → pool-autoscale → Work requests** to see the resize request.

To confirm why pods are pending:

```bash
kubectl describe pod <pending-pod-name>
```

Look for `Insufficient cpu` or `Insufficient memory` under **Events**.

---

## Step 5 – Trigger a Scale-Down

1. Remove the workload:

```bash
kubectl delete deployment scale-test
```

2. Watch the nodes:

```bash
kubectl get nodes -l name=pool-autoscale --watch
```

3. Follow the logs for scale-down messages:

```bash
kubectl -n kube-system logs deployment/cluster-autoscaler -f | grep -i "unneeded\|scale down\|removing"
```

Expected: after the underutilization delay (commonly around 10 minutes by default), idle nodes are drained and removed until the pool returns to its minimum size of 1.

---

## Step 6 – Combine with the Horizontal Pod Autoscaler

1. Confirm metrics-server is running:

```bash
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
```

If it is missing, enable it from the cluster's **Add-ons** page or install it from the Kubernetes metrics-server release manifest.

2. Create `php-apache.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      run: php-apache
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      nodeSelector:
        name: pool-autoscale
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 500m
            memory: 256Mi
          limits:
            cpu: 1
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
spec:
  selector:
    run: php-apache
  ports:
  - port: 80
```

3. Apply it and create an HPA:

```bash
kubectl apply -f php-apache.yaml
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=12
kubectl get hpa
```

4. Generate load (Terminal 1):

```bash
kubectl run load-generator --image=busybox:1.28 --restart=Never -it --rm -- \
  /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
```

5. Watch the chain react (Terminal 2):

```bash
kubectl get hpa php-apache --watch
kubectl get pods -l run=php-apache
kubectl get nodes -l name=pool-autoscale
```

Expected chain: CPU rises → HPA increases replicas → some pods go `Pending` → Cluster Autoscaler adds nodes → pods schedule.

6. Stop the load with `Ctrl+C`. After the stabilization windows, replicas drop first, then idle nodes are removed.

---

## Step 7 – Protect a Node or Pod from Scale-Down (Optional)

```bash
# Prevent a specific node from being removed
kubectl annotate node <node-name> cluster-autoscaler.kubernetes.io/scale-down-disabled=true

# Prevent a pod from being evicted during scale-down
kubectl annotate pod <pod-name> cluster-autoscaler.kubernetes.io/safe-to-evict=false
```

Remove the node annotation afterwards:

```bash
kubectl annotate node <node-name> cluster-autoscaler.kubernetes.io/scale-down-disabled-
```

---

## Validation Checklist

- [ ] Node pool `pool-autoscale` created on `VM.Standard.E5.Flex`
- [ ] IAM policy for the autoscaler created
- [ ] Cluster Autoscaler add-on is **Active** and its pod is `Running`
- [ ] Node count increased when pods were `Pending`
- [ ] Node count decreased after the workload was deleted
- [ ] HPA and Cluster Autoscaler worked together

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| No scale-up, pods `Pending` | Autoscaler unauthorized | Check logs for 401/403; review the dynamic group and policies |
| Pods `Pending` but no scale-up | Pod request larger than any node; `nodeSelector` or taint mismatch; pool not listed in the `nodes` value | Reduce requests, fix selectors, confirm the pool OCID and min/max |
| Pods have no requests | CA only sees requests | Add `resources.requests` to the pods |
| Scale-up fails with capacity error | No host capacity in the AD | Use a capacity reservation or add another AD to the pool |
| Nodes do not scale down | PodDisruptionBudgets, local storage, `safe-to-evict=false`, system pods, or the delay has not elapsed | Check logs for "not removed" reasons and wait longer |
| Pool size exceeds `max` | Pool also resized manually | Let the autoscaler own the pool size |
| Autoscaler pod crash-looping | Wrong add-on config | Review the `nodes` value format and the pod logs |

---

## Clean-up

```bash
kubectl delete deployment scale-test php-apache --ignore-not-found
kubectl delete service php-apache --ignore-not-found
kubectl delete hpa php-apache --ignore-not-found
kubectl delete pod load-generator --ignore-not-found
```

