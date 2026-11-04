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
| Cluster type | **Enhanced** cluster (for the managed add-on and workload identity) |
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

The Cluster Autoscaler must be allowed to resize node pools. This lab uses **workload identity**.

1. Open **Identity & Security → Policies** and select the cluster's compartment.
2. Click **Create Policy**, name it `oke-cluster-autoscaler-policy`.
3. Add these statements **(verify)**, replacing `<compartment-name>` and `<cluster-ocid>`:

```
Allow any-user to manage cluster-node-pools in compartment <compartment-name>
  where all {request.principal.type = 'workload',
             request.principal.namespace = 'kube-system',
             request.principal.service_account = 'cluster-autoscaler',
             request.principal.cluster_id = '<cluster-ocid>'}

Allow any-user to manage instance-family in compartment <compartment-name>
  where all {request.principal.type = 'workload',
             request.principal.namespace = 'kube-system',
             request.principal.service_account = 'cluster-autoscaler',
             request.principal.cluster_id = '<cluster-ocid>'}

Allow any-user to use subnets in compartment <compartment-name>
  where all {request.principal.type = 'workload',
             request.principal.namespace = 'kube-system',
             request.principal.service_account = 'cluster-autoscaler',
             request.principal.cluster_id = '<cluster-ocid>'}

Allow any-user to read virtual-network-family in compartment <compartment-name>
  where all {request.principal.type = 'workload',
             request.principal.namespace = 'kube-system',
             request.principal.service_account = 'cluster-autoscaler',
             request.principal.cluster_id = '<cluster-ocid>'}

Allow any-user to use vnics in compartment <compartment-name>
  where all {request.principal.type = 'workload',
             request.principal.namespace = 'kube-system',
             request.principal.service_account = 'cluster-autoscaler',
             request.principal.cluster_id = '<cluster-ocid>'}

Allow any-user to inspect compartments in compartment <compartment-name>
  where all {request.principal.type = 'workload',
             request.principal.namespace = 'kube-system',
             request.principal.service_account = 'cluster-autoscaler',
             request.principal.cluster_id = '<cluster-ocid>'}
```

4. Click **Create**.

---

## Step 3 – Enable the Cluster Autoscaler Add-on

1. Open the cluster → **Resources → Add-ons → Manage add-ons**.
2. Select **Cluster Autoscaler** and enable it.
3. Add the configuration key/value pairs **(verify key names)**:

| Key | Example value | Meaning |
|---|---|---|
| `nodePools` | `1:4:<node-pool-ocid>` | `min:max:node-pool-ocid`; one entry per pool |
| `authType` | `workload` | Use workload identity |

4. Click **Save changes** and wait for the add-on status to become **Active**.
5. Verify in Cloud Shell:

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
      nodeSelector:
        name: pool-autoscale
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
| No scale-up, pods `Pending` | Autoscaler unauthorized | Check logs for 401/403; review policies and `authType` |
| Pods `Pending` but no scale-up | Pod request larger than any node; `nodeSelector` or taint mismatch; pool not in `nodePools` | Reduce requests, fix selectors, confirm the pool OCID and min/max |
| Pods have no requests | CA only sees requests | Add `resources.requests` to the pods |
| Scale-up fails with capacity error | No host capacity in the AD | Use a capacity reservation or add another AD to the pool |
| Nodes do not scale down | PodDisruptionBudgets, local storage, `safe-to-evict=false`, system pods, or the delay has not elapsed | Check logs for "not removed" reasons and wait longer |
| Pool size exceeds `max` | Pool also resized manually | Let the autoscaler own the pool size |
| Autoscaler pod crash-looping | Wrong add-on config | Review the `nodePools` value format and the pod logs |

---

## Clean-up

```bash
kubectl delete deployment scale-test php-apache --ignore-not-found
kubectl delete service php-apache --ignore-not-found
kubectl delete hpa php-apache --ignore-not-found
kubectl delete pod load-generator --ignore-not-found
```

In the console:
1. Disable the **Cluster Autoscaler** add-on, or remove `pool-autoscale` from `nodePools`.
2. Delete the `pool-autoscale` node pool.
3. Delete the `oke-cluster-autoscaler-policy` policy.

---

## Review Questions

1. What condition causes the Cluster Autoscaler to add nodes?
2. Why do pod resource requests matter for autoscaling?
3. What is the difference between HPA and the Cluster Autoscaler?
4. What does the `nodePools` value `1:4:<ocid>` mean?
5. Why should you avoid manually resizing a pool the autoscaler manages?
6. How can you stop the autoscaler from removing a particular node or pod?
