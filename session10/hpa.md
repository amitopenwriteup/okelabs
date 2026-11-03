# Lab: Horizontal Pod Autoscaler (HPA) on OKE

**Duration:** ~30 minutes
**Platform:** Oracle Container Engine for Kubernetes (OKE), existing cluster

---

## Lab Objectives

By the end of this lab you will be able to:

1. Verify (or deploy) the Kubernetes **Metrics Server**, which HPA depends on.
2. Deploy a sample application with CPU **requests** defined.
3. Create a **Horizontal Pod Autoscaler** that scales pods on CPU utilization.
4. Generate load, watch pods scale out, then watch them scale back in.

## Concepts at a Glance

| Component | Scales | Trigger | Needs |
|---|---|---|---|
| Metrics Server | Nothing (provides data) | n/a | Kubelet access on TCP 10250 |
| Horizontal Pod Autoscaler (HPA) | **Pods** (replicas) | CPU/memory vs. target utilization | Metrics Server and pod resource **requests** |

The HPA adjusts the replica count of a Deployment (or StatefulSet) to keep a metric near a target. For CPU:

```
desiredReplicas = ceil( currentReplicas × currentUtilization / targetUtilization )
```

Utilization is measured **relative to the pod's CPU request**, so every container **must define `resources.requests`**. Otherwise the HPA shows `<unknown>`.

## Prerequisites

- An existing OKE cluster with at least one worker node.
- `kubectl` configured for the cluster (`oci ce cluster create-kubeconfig ...`).

Verify access:

```bash
kubectl get nodes -o wide
kubectl cluster-info
```

---

## Part 1: Metrics Server

The Metrics Server scrapes each kubelet and exposes CPU and memory usage through the Metrics API (`metrics.k8s.io`). It powers `kubectl top` and the HPA.

```
kubelet (each node) ──► Metrics Server ──► Metrics API ──► kubectl top / HPA
```

### Step 1.1: Check Whether It Is Already Installed

```bash
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
```

If `kubectl top nodes` returns data, skip to Part 2. If you get `error: Metrics API not available`, continue.

### Step 1.2: Deploy the Metrics Server

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

### Step 1.3: Verify Metrics Are Flowing

```bash
kubectl top nodes
kubectl top pods -A
```

### Step 1.4: Troubleshooting

```bash
kubectl -n kube-system logs deployment/metrics-server
```

| Symptom | Fix |
|---|---|
| `x509: cannot validate certificate for <ip>` | Add `--kubelet-insecure-tls` to the container args (acceptable for a lab; use properly signed kubelet serving certs in production). |
| `unable to fully scrape metrics ... no route / timeout` | Make sure the worker node security list / NSG allows the Kubernetes API endpoint to reach workers on **TCP 10250**. |
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

## Part 2: Horizontal Pod Autoscaler

### Step 2.1: Deploy a Sample Application

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

> **Note:** If your nodes enforce short-name mode (see the `short name mode is enforcing` error earlier), use a fully qualified image name such as `registry.k8s.io/hpa-example` (already qualified here) and `docker.io/library/busybox:1.36` for the load generator.

### Step 2.2: Create the HPA

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

### Step 2.3: Generate Load

Open a **second terminal** and watch the HPA:

```bash
kubectl get hpa php-apache -w
```

In the first terminal, start a load generator:

```bash
kubectl run load-generator --image=docker.io/library/busybox:1.36 --restart=Never -it --rm -- \
  /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
```

For more load, run the same command in additional terminals with different pod names (`load-generator-2`, and so on).

### Step 2.4: Observe Pod Scaling

In another terminal:

```bash
kubectl get pods -l app=php-apache -w
kubectl top pods -l app=php-apache
kubectl describe hpa php-apache
```

**Expected:** CPU utilization climbs above 50%, the HPA increases replicas (for example 1 → 3 → 5 …), and events show `SuccessfulRescale`.

### Step 2.5: Stop the Load and Watch Scale-Down

Press `Ctrl+C` in the load-generator terminal (the pod is removed because of `--rm`). Then:

```bash
kubectl get hpa php-apache -w
kubectl get pods -l app=php-apache -w
```

**Expected:** after the stabilization window, the HPA reduces replicas back to `minReplicas`.

> Scale-down is intentionally slow. The default stabilization window is 300 seconds; this lab sets it to 60 seconds to speed things up.

---

## Optional: Combine with the Cluster Autoscaler

If the Cluster Autoscaler is running, each replica requests 500m CPU, so a large replica count will exceed free capacity on your node(s):

1. Load rises, so the HPA raises replicas.
2. Replicas can't be scheduled, so pods go `Pending`.
3. The Cluster Autoscaler scales the node pool up and a new node joins.
4. Pending pods are scheduled and become `Running`.

```bash
kubectl get pods -l app=php-apache
kubectl get nodes -w
kubectl -n kube-system logs deployment/cluster-autoscaler | grep -i "scale-up"
```

---

## Verification Checklist

- [ ] `kubectl top nodes` and `kubectl top pods` return data.
- [ ] HPA shows a numeric `TARGETS` value (not `<unknown>`).
- [ ] HPA increased replicas under load.
- [ ] HPA decreased replicas back to the minimum after the load stopped.

## Troubleshooting Quick Reference

| Problem | Likely Cause | Action |
|---|---|---|
| HPA `<unknown>` target | Metrics Server unavailable, or missing CPU requests | Run `kubectl top pods`; ensure `resources.requests.cpu` is set. |
| HPA doesn't scale up | Load too low or utilization is under target | Add more load generators or lower the target. |
| HPA stuck at max replicas, pods `Pending` | Not enough node capacity | Add nodes, or use the Cluster Autoscaler. |
| `Metrics API not available` | Metrics Server not ready, or port 10250 blocked | See Step 1.4. |
| Load generator pod fails to pull image | Short-name enforcement on nodes | Use `docker.io/library/busybox:1.36`. |

## Cleanup

```bash
kubectl delete hpa php-apache
kubectl delete -f php-apache.yaml
kubectl delete pod load-generator --ignore-not-found

# Optional: remove the Metrics Server
kubectl delete -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

## Summary

| Layer | Tool | What you did |
|---|---|---|
| Data | Metrics Server | Provided CPU/memory metrics via the Metrics API |
| Pods | HPA | Scaled replicas up and down based on CPU utilization |
