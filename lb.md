# Lab: Expose an App on OKE with a Public OCI Load Balancer

**Region:** ap-hyderabad-1
**VCN:** `oke-vcn` (172.16.0.0/16)
**Goal:** Deploy a sample app on OKE and expose it to the internet through a Kubernetes `Service` of type `LoadBalancer`, which provisions a public OCI Load Balancer.

This lab continues from the *OKE Network Design* lab guide and reuses its resource names and CIDRs.

| Resource | Name | CIDR |
|---|---|---|
| VCN | `oke-vcn` | 172.16.0.0/16 |
| API endpoint subnet (public) | `oke-api-public` | 172.16.0.0/29 |
| Worker node subnet (private) | `oke-workers-private` | 172.16.1.0/24 |
| Load balancer subnet (public) | `oke-lb-public` | 172.16.2.0/24 |
| LB route table | `oke-lb-rt` | 0.0.0.0/0 -> `oke-igw` |
| LB NSG | `nsg-lb` | - |
| Workers NSG | `nsg-workers` | - |

---

## 1. Prerequisites in `oke-vcn`

Confirm each item before creating the Service.

### 1.1 Public load balancer subnet

- [ ] The subnet is in `oke-vcn`.
- [ ] It has a route to an Internet Gateway (`oke-lb-rt`: 0.0.0.0/0 -> `oke-igw`).
- [ ] It uses a public IPv4 address configuration (public subnet).
- [ ] Ideally it is a dedicated public subnet for load balancers (`oke-lb-public`, 172.16.2.0/24).

### 1.2 Worker node subnet

- [ ] Worker nodes can receive traffic from the load balancer subnet.
- [ ] Security lists or NSGs allow the Kubernetes NodePort range (commonly **30000-32767**) from the load balancer subnet CIDR (172.16.2.0/24).
- [ ] Outbound access is available if the application requires it (NAT Gateway / Service Gateway via `oke-workers-rt`).

Rules to confirm (same as Section 5 of the network guide):

**`nsg-lb`**

| Dir | Source / Dest | Protocol | Port | Purpose |
|---|---|---|---|---|
| In | 0.0.0.0/0 (or restricted) | TCP | 80, 443 | Public HTTP/HTTPS |
| Out | Workers 172.16.1.0/24 | TCP | 30000-32767, 10256 | NodePort and health checks |

**`nsg-workers`**

| Dir | Source / Dest | Protocol | Port | Purpose |
|---|---|---|---|---|
| In | LB 172.16.2.0/24 | TCP | 30000-32767 | NodePort from LB |
| In | LB 172.16.2.0/24 | TCP | 10256 | kube-proxy health check |

> If you use security lists instead of NSGs, add the same rules to the security lists of `oke-lb-public` and `oke-workers-private`. Use one method consistently.

### 1.3 OCI permissions

- [ ] The OKE cloud controller has permission to manage load balancers.
- The OKE cluster setup normally creates or uses the required policies. If provisioning fails with an authorization error, check the OKE dynamic group and IAM policies.

### 1.4 Cluster access

```bash
kubectl get nodes
```

All nodes should show `Ready`.

---

## 2. Create a public LoadBalancer Service

### 2.1 Application manifest

Save as `web-public.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-public
  annotations:
    service.beta.kubernetes.io/oci-load-balancer-internal: "false"
    service.beta.kubernetes.io/oci-load-balancer-shape: "flexible"
    service.beta.kubernetes.io/oci-load-balancer-shape-flex-min: "10"
    service.beta.kubernetes.io/oci-load-balancer-shape-flex-max: "100"
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
```

### 2.2 Apply it

```bash
kubectl apply -f web-public.yaml
```

### 2.3 Check the Service

```bash
kubectl get svc web-public --watch
```

When provisioning completes, the `EXTERNAL-IP` column should contain the public IP address or hostname:

```bash
kubectl get svc web-public
```

### 2.4 Test it

```bash
curl http://<EXTERNAL-IP>
```

Expected result: the default nginx welcome page.

---

## 3. Select a specific OCI subnet

If the cluster has multiple eligible subnets, specify the load balancer subnet explicitly:

```yaml
metadata:
  annotations:
    service.beta.kubernetes.io/oci-load-balancer-internal: "false"
    service.beta.kubernetes.io/oci-load-balancer-subnet1: "ocid1.subnet.oc1...."
```

Use the OCID of the **public** subnet in `oke-vcn` (`oke-lb-public`), **not** the worker subnet.

For a load balancer spanning multiple subnets, you can specify another subnet when supported by your OKE configuration:

```yaml
service.beta.kubernetes.io/oci-load-balancer-subnet1: "ocid1.subnet.oc1...."
service.beta.kubernetes.io/oci-load-balancer-subnet2: "ocid1.subnet.oc1...."
```

---

## 4. HTTPS example

Create a Kubernetes TLS secret:

```bash
kubectl create secret tls web-tls \
  --cert=tls.crt \
  --key=tls.key
```

Then expose port 443:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-https
  annotations:
    service.beta.kubernetes.io/oci-load-balancer-internal: "false"
    service.beta.kubernetes.io/oci-load-balancer-shape: "flexible"
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - name: https
      port: 443
      targetPort: 8443
      protocol: TCP
```

> **Notes**
> - `targetPort: 8443` assumes your pods listen on 8443. The nginx deployment above listens on 80, so use `targetPort: 80` for it, or change the container port.
> - This Service forwards TCP 443 as-is (pass-through). The `web-tls` secret is only used if your pods or an Ingress controller terminate TLS. To terminate TLS on the OCI Load Balancer itself, use the OCI SSL annotations (for example `service.beta.kubernetes.io/oci-load-balancer-ssl-ports` and `service.beta.kubernetes.io/oci-load-balancer-tls-secret`); check Oracle's OKE documentation for your version.

For production HTTPS, an OCI Load Balancer listener can also use an OCI Certificates service certificate. If you need routing for multiple domains or paths, use an OKE Ingress Controller instead of creating a separate public load balancer for every service.

---

## 5. Troubleshooting

Check the service events:

```bash
kubectl describe svc web-public
```

Check the cloud-controller-manager logs:

```bash
kubectl -n kube-system get pods | grep -i cloud
kubectl -n kube-system logs <cloud-controller-manager-pod>
```

Common causes of failure:

1. The selected subnet is private or has no Internet Gateway route.
2. The subnet is in a different VCN than the OKE worker nodes.
3. Security rules do not allow traffic to the worker NodePorts.
4. The selected subnet has insufficient IP addresses.
5. The service annotation selects an invalid or inaccessible subnet.
6. Required OKE IAM policies are missing.
7. A public IP quota or load balancer quota has been reached.

---

## 6. Clean up

To remove the OCI Load Balancer created by the Kubernetes service:

```bash
kubectl delete svc web-public
```

Deleting the LoadBalancer service removes the load balancer managed for that service. Remove the sample deployment too:

```bash
kubectl delete deployment web
```

---

## 7. Lab checklist

- [ ] `oke-lb-public` is public, in `oke-vcn`, and uses `oke-lb-rt` (route to `oke-igw`)
- [ ] `nsg-lb` and `nsg-workers` allow NodePort 30000-32767 and 10256 between the LB and workers
- [ ] `web-public` Service shows an `EXTERNAL-IP`
- [ ] `curl http://<EXTERNAL-IP>` returns the nginx page
- [ ] Service deleted and load balancer removed after the lab
