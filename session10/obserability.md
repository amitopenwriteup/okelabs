# Lab: Observing Kubernetes Clusters in OKE Using Add-ons

**Estimated time:** 40–50 minutes
**Level:** Beginner to Intermediate

> **Note:** OCI Console labels and add-on names change over time. Items most likely to differ are marked **(verify)**.

---

## Concept

Observing a cluster means answering four questions: *Is it healthy? How much is it using? What is running? What went wrong?* In OKE you combine several sources.

| Source | What it shows | How you reach it |
|---|---|---|
| **Metrics Server add-on** | Live CPU and memory for nodes and pods | `kubectl top` and the Horizontal Pod Autoscaler |
| **kubectl** | Status, events, logs, resource details | Cloud Shell or a jump host |
| **OCI Console** | Cluster and node pool state, work requests, compute instance metrics | Console pages |

> `kubectl` shows what is in the cluster. The OCI Console shows how the cluster and its compute infrastructure are doing. Use both.

## Objective

1. Review the add-ons on a cluster.
2. Enable the **Metrics Server** add-on and read resource usage with `kubectl top`.
3. Deploy a sample workload and observe it from `kubectl` and the OCI Console.
4. Generate load and watch the metrics change.
5. Investigate a failing pod using events and logs.

## Prerequisites

| Requirement | Details |
|---|---|
| Cluster type | **Enhanced** cluster (needed for managing add-ons from the console) |
| Node pool | A managed node pool with at least 2 nodes |
| Access | `kubectl` through Cloud Shell or a jump host that can reach the API endpoint |

---

## Step 1 – Review the Current Add-ons

1. Open **Developer Services → Containers & Artifacts → Kubernetes Clusters (OKE)** and select your cluster.
2. Under **Resources**, click **Add-ons**.
3. Review the list. Note which add-ons are enabled and their status and version.

Check what is already running in the cluster:

```bash
kubectl get pods -n kube-system
kubectl get deployment metrics-server -n kube-system
```

If `metrics-server` is not found, enable it in Step 2.

---

## Step 2 – Enable the Metrics Server Add-on

1. On the **Add-ons** page click **Manage add-ons**.
2. Select **Kubernetes Metrics Server** **(verify name)** and enable it.
3. Choose the update mode: **Automatic updates** or **Choose a version**. This lab uses **Automatic updates**.
4. Leave the default configuration and click **Save changes**.
5. Wait until the add-on status is **Active**.

### Verify

```bash
kubectl get pods -n kube-system | grep metrics-server
kubectl top nodes
kubectl top pods -A
```

Expected: `metrics-server` is `Running`, and `kubectl top nodes` lists each node with CPU and memory usage. It can take a minute or two before the first values appear. If you see `Metrics API not available`, wait and retry.

---

## Step 3 – Deploy a Workload and Observe It

Deploy a sample application with resource requests, so usage and capacity can be compared:

```yaml
# observe-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: observe-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: observe-demo
  template:
    metadata:
      labels:
        app: observe-demo
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: observe-demo
spec:
  selector:
    app: observe-demo
  ports:
  - port: 80
```

```bash
kubectl apply -f observe-demo.yaml
kubectl get pods -l app=observe-demo -o wide
```

Observe it in two places:

| Where | What to check |
|---|---|
| `kubectl` | `kubectl get deploy,pods,svc -l app=observe-demo` and `kubectl describe pod <pod-name>` |
| OCI Console | **Cluster → Node pools → (your pool) → Nodes**, then open a node's compute instance and its **Metrics** |

Read container logs and events:

```bash
kubectl logs deployment/observe-demo --tail=20
kubectl get events --sort-by=.lastTimestamp | tail -20
```

---

## Step 4 – Generate Load and Watch the Metrics

1. In Terminal 1, generate load:

```bash
kubectl run load-generator --image=busybox:1.28 --restart=Never -it --rm -- \
  /bin/sh -c "while true; do wget -q -O- http://observe-demo; done"
```

2. In Terminal 2, watch usage:

```bash
kubectl top pods -l app=observe-demo
kubectl top nodes
```

Re-run the commands every 15 to 30 seconds. Metrics Server refreshes about once a minute.

3. In the OCI Console, open the worker node's compute instance **Metrics** and look at **CPU utilization** for the same period.
4. Press `Ctrl+C` in Terminal 1 to stop the load, and watch the values fall.

Tell the difference between *requested* and *used* resources:

```bash
kubectl describe node <node-name> | sed -n '/Allocated resources/,/Events/p'
```

Allocated figures show **requests and limits** (what is reserved). `kubectl top` shows **actual usage**.

---

## Step 5 – Investigate a Problem (Optional)

Create a pod that fails, then find out why:

```bash
kubectl run broken-pod --image=nginx:does-not-exist
```

1. `kubectl get pod broken-pod` shows `ErrImagePull` or `ImagePullBackOff`.
2. `kubectl describe pod broken-pod` shows the failure under **Events**.
3. `kubectl get events --sort-by=.lastTimestamp | tail` shows the same warning.

Delete it when done:

```bash
kubectl delete pod broken-pod
```

---

## Validation Checklist

- [ ] Metrics Server add-on is **Active** and `kubectl top nodes` works
- [ ] Sample workload visible in `kubectl` and the OCI Console
- [ ] CPU usage rose under load and fell afterwards
- [ ] Requested and used resources told apart
- [ ] Failure investigated through events

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `kubectl top` says `Metrics API not available` | Metrics Server not ready yet or not enabled | Wait a minute; check `kubectl get pods -n kube-system \| grep metrics-server` and the add-on status |
| Add-on stuck in a non-Active state | Cluster is Basic, or a work request is running | Check the cluster type and **Work requests**; retry |
| `kubectl top pods` shows no data for a new pod | Metrics not collected yet | Wait about a minute and retry |
| Pods stay `Pending` | Not enough free CPU or memory on the nodes | `kubectl describe pod`; reduce requests or add nodes |

---

## Clean-up

```bash
kubectl delete -f observe-demo.yaml --ignore-not-found
kubectl delete pod load-generator broken-pod --ignore-not-found
```

Keep the Metrics Server add-on if you plan to use the Horizontal Pod Autoscaler. Otherwise disable it from **Add-ons → Manage add-ons**.

---

## Review Questions

1. What is the difference between `kubectl top` values and the resource requests shown by `kubectl describe node`?
2. Which add-on provides the data behind `kubectl top`?
3. Which questions are better answered by `kubectl`, and which by the OCI Console metrics?
4. Why can `kubectl top` show no data right after a pod starts?
