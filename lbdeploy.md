# Demo: Defining Kubernetes Services of Type LoadBalancer

Expose an application running on OKE (`ap-hyderabad-1`) to the internet or a private network by defining a Kubernetes Service of type `LoadBalancer`. OKE provisions an OCI Load Balancer (or Network Load Balancer) for you and keeps it in sync with the Service.

> **Cost note:** each `LoadBalancer` Service creates a billable OCI load balancer. Delete the Services at the end of the demo (Lab 9).

## What you will do

| Lab | Step | Done when |
| --- | --- | --- |
| 1 | Check prerequisites | `kubectl get nodes` shows a `Ready` node |
| 2 | Deploy a sample application | Pods are `Running` |
| 3 | Create a basic LoadBalancer Service | Service is created |
| 4 | Get the external IP and test | `curl` returns the nginx page |
| 5 | Use a flexible shape | Annotations set the bandwidth |
| 6 | Create a private (internal) load balancer | Service gets a private IP |
| 7 | Use a Network Load Balancer | NLB is provisioned |
| 8 | Verify in OCI | The load balancer appears in the Console |
| 9 | Update and clean up | Services deleted, load balancers removed |

---

## Lab 1: Prerequisites

- An OKE cluster with at least one `Ready` worker node
- `kubectl` connected to the cluster (from Cloud Shell, or a VM with the OCI CLI)
- A subnet in the cluster's VCN that can host load balancers (usually the cluster's service or load balancer subnet)
- For a public load balancer, that subnet must be a public subnet. For an internal one, use a private subnet.

Connect and check:

```bash
export REGION=ap-hyderabad-1
export CLUSTER_OCID='<YOUR_CLUSTER_OCID>'

oci ce cluster create-kubeconfig \
  --cluster-id "$CLUSTER_OCID" \
  --region "$REGION" \
  --file "$HOME/.kube/config" \
  --token-version 2.0.0

kubectl get nodes
kubectl get namespaces
```

If load balancer creation fails later, check the OKE security list or NSG rules, load balancer service limits for your tenancy, and the Kubernetes Engine policy documentation (see Troubleshooting).

---

## Lab 2: Deploy a sample application

Create `nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  labels:
    app: nginx-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get deployment nginx-demo
kubectl get pods -l app=nginx-demo -o wide
```

Both pods should be `Running`.

---

## Lab 3: Create a basic LoadBalancer Service

Create `nginx-lb.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx-demo
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```

```bash
kubectl apply -f nginx-lb.yaml
```

What happens: OKE sees the Service and provisions an OCI load balancer. The Service's `selector` must match the pod labels (`app: nginx-demo`).

By default, the load balancer is public. With Kubernetes 1.37 and later, a `LoadBalancer` Service is provisioned as a **flexible** load balancer with minimum and maximum bandwidth of 100 Mbps, unless you set shape annotations. Existing load balancers are not converted or resized.

---

## Lab 4: Get the external IP and test

```bash
kubectl get service nginx-lb -o wide -w
```

`EXTERNAL-IP` shows `<pending>` while the load balancer is created. Wait until it shows a public IP address, then press `Ctrl+C`.

Test it:

```bash
EXTERNAL_IP=$(kubectl get service nginx-lb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "$EXTERNAL_IP"
curl -I http://$EXTERNAL_IP
```

You should get `HTTP/1.1 200 OK` from nginx. You can also open `http://<EXTERNAL_IP>` in a browser.

Inspect the Service and its events:

```bash
kubectl describe service nginx-lb
kubectl get endpoints nginx-lb
```

The endpoints should list the two pod IPs.

---

## Lab 5: Use a flexible shape

Flexible shapes let you set minimum and maximum bandwidth. Create `nginx-lb-flex.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb-flex
  annotations:
    oci.oraclecloud.com/load-balancer-type: "lb"
    service.beta.kubernetes.io/oci-load-balancer-shape: "flexible"
    service.beta.kubernetes.io/oci-load-balancer-shape-flex-min: "10"
    service.beta.kubernetes.io/oci-load-balancer-shape-flex-max: "100"
spec:
  type: LoadBalancer
  selector:
    app: nginx-demo
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```

```bash
kubectl apply -f nginx-lb-flex.yaml
kubectl get service nginx-lb-flex -w
```

The `flex-min` and `flex-max` values are bandwidth in Mbps. OCI Load Balancers support bandwidth between 10 and 8,000 Mbps. Fixed-shape load balancers are deprecated for new creation, so use `flexible`.

---

## Lab 6: Create a private (internal) load balancer

An internal load balancer gets a private IP and is reachable only from inside the VCN (or networks connected to it). It needs a private subnet.

Find a private subnet OCID:

```bash
oci network subnet list \
  --compartment-id <COMPARTMENT_OCID> \
  --region ap-hyderabad-1 \
  --query 'data[].{name:"display-name",id:id,private:"prohibit-public-ip-on-vnic"}' \
  --output table
```

Create `nginx-lb-internal.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb-internal
  annotations:
    oci.oraclecloud.com/load-balancer-type: "lb"
    service.beta.kubernetes.io/oci-load-balancer-internal: "true"
    service.beta.kubernetes.io/oci-load-balancer-subnet1: "<PRIVATE_SUBNET_OCID>"
    service.beta.kubernetes.io/oci-load-balancer-shape: "flexible"
    service.beta.kubernetes.io/oci-load-balancer-shape-flex-min: "10"
    service.beta.kubernetes.io/oci-load-balancer-shape-flex-max: "100"
spec:
  type: LoadBalancer
  selector:
    app: nginx-demo
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```

```bash
kubectl apply -f nginx-lb-internal.yaml
kubectl get service nginx-lb-internal -w
```

`EXTERNAL-IP` will be a private address. Test it from a pod inside the cluster:

```bash
kubectl run curl-test --rm -it --restart=Never --image=curlimages/curl -- \
  curl -I http://<PRIVATE_LB_IP>
```

---

## Lab 7: Use a Network Load Balancer

A Network Load Balancer is a pass-through Layer 3/4 balancer with a regional VIP and no bandwidth setting. Use it for TCP/UDP traffic or when you need source IP preservation. Create `nginx-nlb.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nlb
  annotations:
    oci.oraclecloud.com/load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: nginx-demo
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```

```bash
kubectl apply -f nginx-nlb.yaml
kubectl get service nginx-nlb -w
```

Useful NLB annotations (they use a different prefix from load balancers):

| Annotation | Purpose |
| --- | --- |
| `oci-network-load-balancer.oraclecloud.com/internal: "true"` | Create an internal NLB |
| `oci-network-load-balancer.oraclecloud.com/subnet: "<SUBNET_OCID>"` | Place the NLB in a specific subnet |
| `oci-network-load-balancer.oraclecloud.com/is-preserve-source: "<value>"` | Preserve the client source IP |
| `oci-network-load-balancer.oraclecloud.com/health-check-retries`, `health-check-timeout`, `health-check-interval` | Health check tuning |

---

## Lab 8: Verify in OCI

Check what Kubernetes reports:

```bash
kubectl get services -o wide
kubectl get service nginx-lb -o yaml
```

Then check in OCI:

- Console: **Networking -> Load balancers** (and **Network load balancers**) in the cluster's compartment, region `ap-hyderabad-1`.
- Look at the listeners, backend sets and backends. The backends are your worker nodes on their node ports.

With the CLI:

```bash
oci lb load-balancer list \
  --compartment-id <COMPARTMENT_OCID> \
  --region ap-hyderabad-1 \
  --query 'data[].{name:"display-name",ip:"ip-addresses"[0]."ip-address",state:"lifecycle-state"}' \
  --output table
```

> **Do not edit the provisioned load balancers in the Console, CLI or API.** OKE reverts the changes or causes conflicts that can interrupt service. Change the annotations in the Service manifest and reapply instead.

---

## Lab 9: Update and clean up

### Update

Edit the Service manifest (for example change the flex bandwidth) and reapply:

```bash
kubectl apply -f nginx-lb-flex.yaml
kubectl describe service nginx-lb-flex
```

### Clean up

Deleting a Service deletes its OCI load balancer. This stops the charges.

```bash
kubectl delete service nginx-lb nginx-lb-flex nginx-lb-internal nginx-nlb
kubectl delete deployment nginx-demo
kubectl get services
```

Confirm in the Console that the load balancers are gone or being deleted. If a service refuses to delete, check the events with `kubectl describe service <name>`.

---

## Troubleshooting

| Symptom | Likely cause | Check |
| --- | --- | --- |
| `EXTERNAL-IP` stays `<pending>` | Load balancer is still being created, or creation failed | `kubectl describe service <name>`, read the Events |
| Event mentions subnet or no subnet found | Wrong or missing subnet annotation, or no suitable subnet | Set `oci-load-balancer-subnet1` to a valid subnet OCID in the cluster's VCN |
| Event mentions a limit or quota | Tenancy load balancer limit reached | Check service limits in the Console, or delete unused load balancers |
| Event mentions authorization or policy | Missing permissions for the cluster | Check the Kubernetes Engine policy documentation |
| `curl` times out, but the IP exists | Security list or NSG blocks traffic | Allow inbound on the service port to the load balancer subnet, and from it to the node port range on the worker subnet |
| `curl` returns 502/503 or connection refused | No healthy backends | `kubectl get endpoints <name>`, check pod labels match the Service selector |
| Internal LB unreachable | Testing from outside the VCN | Test from a pod in the cluster or a VM in the VCN |
| Console edits get reverted | OKE manages the load balancer | Change annotations in the manifest and reapply |

---

## Sources

- [Defining Kubernetes Services of Type LoadBalancer](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcreatingloadbalancer.htm)
- [Summary of Annotations for Load Balancers and Network Load Balancers](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcreatingloadbalancer_topic-Summaryofannotations.htm)
- [Provisioning OCI Load Balancers for Kubernetes Services of Type LoadBalancer](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcreatingloadbalancers-subtopic.htm)
- [OCI Cloud Controller Manager: load balancer annotations](https://github.com/oracle/oci-cloud-controller-manager/blob/master/docs/load-balancer-annotations.md)
