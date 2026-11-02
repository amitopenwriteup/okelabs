# Demo: Working with OCI Native Ingress Controller

Install the OCI Native Ingress Controller on an OKE cluster (`ap-hyderabad-1`) and use Kubernetes `Ingress` resources to route HTTP traffic through a single OCI Load Balancer to several services.

The OCI Native Ingress Controller watches `Ingress` resources and manages an OCI Load Balancer for you: listeners, backend sets and routing rules. Unlike a `LoadBalancer` Service (one load balancer per service), one Ingress setup can serve many hosts and paths from a single load balancer.

> **Cost note:** the controller creates a billable OCI load balancer. Delete the Ingress resources at the end of the demo (Lab 10).

## What you will do

| Lab | Step | Done when |
| --- | --- | --- |
| 1 | Check prerequisites | Cluster, versions, network and subnet ready |
| 2 | Set up IAM for the controller | Principal and policies exist |
| 3 | Install the controller (add-on) | Add-on `lifecycle-state` is `ACTIVE`, pods `Running` |
| 4 | Deploy two sample applications | Pods and Services are up |
| 5 | Create IngressClassParameters and IngressClass | Both objects exist |
| 6 | Create an Ingress with host and path routing | Ingress shows an `ADDRESS` |
| 7 | Test the routes | `curl` returns responses from each app |
| 8 | (Optional) Add TLS | HTTPS listener serves the cert |
| 9 | Verify in OCI | Load balancer and backend sets appear in the Console |
| 10 | Clean up | Ingress, class, apps and load balancer removed |

---

## Lab 1: Prerequisites

- An OKE cluster with a `Ready` worker node and `kubectl` working from Cloud Shell or a VM
- A Kubernetes version supported by the controller (the controller's own docs list 1.27 through 1.36 at the time of writing, so check the current compatibility list)
- Cluster networking set up for load balancers: either OCI VCN-native pod networking (NPN) or Flannel, with security rules that allow the load balancer to reach the workers and pods
- A **public subnet** for the load balancer in the cluster's VCN (or a private subnet for an internal load balancer)
- The OCI CLI configured, if you install the add-on from the CLI
- cert-manager installed in the cluster before the controller (see Lab 3)

Check:

```bash
export REGION=ap-hyderabad-1
export CLUSTER_OCID='<YOUR_CLUSTER_OCID>'

oci ce cluster create-kubeconfig \
  --cluster-id "$CLUSTER_OCID" \
  --region "$REGION" \
  --file "$HOME/.kube/config" \
  --token-version 2.0.0

kubectl get nodes
kubectl version
```

Collect two OCIDs you need later:

```bash
# Compartment OCID: Console -> Identity -> Compartments

# Load balancer subnet OCID (public subnet in the cluster's VCN)
oci network subnet list --compartment-id $COMPARTMENT_OCID --region $REGION \
  --query 'data[].{name:"display-name",id:id,private:"prohibit-public-ip-on-vnic"}' --output table
```

---

## Lab 2: Set up IAM for the controller

The controller calls OCI APIs, so it needs an identity principal. Options:

| Principal | `authType` | Notes |
| --- | --- | --- |
| Instance principal (default) | `instance` | Uses a dynamic group that matches the worker nodes |
| Workload identity | `workloadIdentity` | Requires an Enhanced cluster |
| User principal | `user` | Uses an API key stored as a Kubernetes Secret |

This lab uses instance principal. Create a dynamic group matching your worker nodes' compartment, for example:

```text
ALL {instance.compartment.id = '<COMPARTMENT_OCID>'}
```

Then add policies. These are the sample statements from the controller's Getting Started guide. Replace `<subject>` with `dynamic-group <DYNAMIC_GROUP_NAME>`, and `<compartment-id>` as appropriate:

```text
Allow <subject> to manage load-balancers in compartment <compartment-id>
Allow <subject> to use virtual-network-family in compartment <compartment-id>
Allow <subject> to manage cabundles in compartment <compartment-id>
Allow <subject> to manage cabundle-associations in compartment <compartment-id>
Allow <subject> to manage leaf-certificates in compartment <compartment-id>
Allow <subject> to manage leaf-certificate-versions in compartment <compartment-id>
Allow <subject> to read leaf-certificate-bundles in compartment <compartment-id>
Allow <subject> to manage certificate-associations in compartment <compartment-id>
Allow <subject> to read certificate-authorities in compartment <compartment-id>
Allow <subject> to manage certificate-authority-associations in compartment <compartment-id>
Allow <subject> to read certificate-authority-bundles in compartment <compartment-id>
Allow <subject> to read public-ips in tenancy
Allow <subject> to manage floating-ips in tenancy
Allow <subject> to manage waf-family in compartment <compartment-id>
Allow <subject> to read cluster-family in compartment <compartment-id>
Allow <subject> to use tag-namespaces in tenancy
```

The authoritative list for the add-on is on Oracle's prerequisites page, so confirm it there: <https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengsettingupnativeingresscontroller-addon-prereqs.htm>. Policies can take a minute or two to apply.

---

## Lab 3: Install the controller as an OKE add-on

Create `native-ingress-controller-addon.json`:

```json
{
  "addonName": "NativeIngressController",
  "configurations": [
    {
      "key": "compartmentId",
      "value": "<COMPARTMENT_OCID>"
    },
    {
      "key": "loadBalancerSubnetId",
      "value": "<LOAD_BALANCER_SUBNET_OCID>"
    }
  ]
}
```

The key is `loadBalancerSubnetId` (not `subnetId`). The default `authType` is `instance`. For workload identity or user principal, add:

```json
{
  "key": "authType",
  "value": "workloadIdentity"
}
```

Valid values are `instance` (default), `workloadIdentity` and `user`. Other arguments, such as `logVerbosity`, are listed here: <https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/configuration-arguments-nic.htm>.

Install:

```bash
oci ce cluster list-addons --cluster-id "$CLUSTER_OCID"

oci ce cluster install-addon \
  --addon-name NativeIngressController \
  --cluster-id "$CLUSTER_OCID" \
  --from-json file://./native-ingress-controller-addon.json
```

This creates a work request. Check until the add-on is active:

```bash
oci ce cluster list-addons --cluster-id "$CLUSTER_OCID" \
  --query 'data[].{name:name,state:"lifecycle-state",version:"current-installed-version"}' --output table
```

`lifecycle-state` should be `ACTIVE`. You can also install the add-on from the Console: **Developer Services -> Kubernetes Clusters -> your cluster -> Add-ons**.

> **Install cert-manager first.** The controller's webhook certificate comes from cert-manager. If cert-manager is missing when the controller is installed, the pod stays in `ContainerCreating` with `secret "oci-native-ingress-controller-tls" not found` (see "Fix: pod stuck on the missing TLS Secret" below). The controller's own guide installs cert-manager before the controller, so do the same here. I haven't confirmed that Oracle's add-on prerequisites page says this, so treat it as a safe default.

```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/latest/download/cert-manager.yaml
kubectl get pods -n cert-manager -w
```

Wait until the three cert-manager pods (`cert-manager`, `cert-manager-cainjector`, `cert-manager-webhook`) are `Running`, then install the add-on.

Verify the controller pods:

```bash
kubectl get pods -n native-ingress-controller-system
kubectl get ingressclass
```

The add-on's pods should be `Running`. If the namespace differs on your cluster, find it with `kubectl get pods -A | grep -i ingress`.

### Fix: pod stuck on the missing TLS Secret

**Symptom:** the controller pod stays in `ContainerCreating` for a long time, and `kubectl describe pod` shows:

```text
Warning  FailedMount  kubelet  MountVolume.SetUp failed for volume "cert" : secret "oci-native-ingress-controller-tls" not found
```

The `node.kubernetes.io/unreachable:NoExecute op=Exists for 300s` toleration shown near it is a default and is not the problem.

**Cause:** the Secret `oci-native-ingress-controller-tls` holds the controller's webhook certificate, which cert-manager issues. If cert-manager is not installed, or was installed after the controller, nothing creates the Secret. The most likely reason is a missing or late cert-manager, but confirm it with the checks below.

**1. Check cert-manager and the controller's Certificate:**

```bash
kubectl get pods -n cert-manager
kubectl get crd | grep cert-manager
kubectl get certificate,issuer,certificaterequest -n native-ingress-controller-system
```

**2. Fix by case:**

| Check result | Case | Fix |
| --- | --- | --- |
| No `cert-manager` namespace or CRDs | cert-manager missing | Install it with the `kubectl apply` command above, then wait for its three pods to be `Running` |
| cert-manager is running, but step 1 shows no Certificate or Issuer | cert-manager was installed after the controller, so the objects were never created | Remove the add-on and install it again (below) |
| Certificate exists but is not `Ready` | Issuance is failing | `kubectl describe certificate -n native-ingress-controller-system`, then `kubectl describe certificaterequest -n native-ingress-controller-system` and read the events |

To remove and reinstall the add-on, use the Console: **Kubernetes Clusters -> your cluster -> Add-ons -> NativeIngressController -> Remove**, wait for it to disappear from `oci ce cluster list-addons`, then run `oci ce cluster install-addon` again with your JSON file.

**3. Check the result:**

```bash
kubectl get secret oci-native-ingress-controller-tls -n native-ingress-controller-system
kubectl get pods -n native-ingress-controller-system -w
```

Once the Secret exists, the kubelet mounts it on its next retry and the pod starts. If it does not, delete the pod and the Deployment recreates it:

```bash
kubectl delete pod -n native-ingress-controller-system -l app.kubernetes.io/name=oci-native-ingress-controller
```

If the pod starts but fails afterwards, read its logs. The next likely cause is an IAM error (401, 403 or 404) from missing policies, covered in Lab 2.

### Alternative: install with Helm

This installs from the open-source repository instead of the add-on. It needs cert-manager first, because the controller's webhooks use its certificates:

```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/latest/download/cert-manager.yaml

git clone https://github.com/oracle/oci-native-ingress-controller.git
cd oci-native-ingress-controller
helm install oci-native-ingress-controller helm/oci-native-ingress-controller

kubectl get pods -n native-ingress-controller-system -o wide -w
```

Do not run both the add-on and the Helm install on the same cluster.

---

## Lab 4: Deploy two sample applications

Create a namespace, and two echo servers that report which pod answered:

```bash
kubectl create namespace ingress-demo
```

Create `echo-apps.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: testecho1
  namespace: ingress-demo
  labels:
    app: testecho1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: testecho1
  template:
    metadata:
      labels:
        app: testecho1
    spec:
      containers:
        - name: testecho1
          image: registry.k8s.io/echoserver:1.4
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: testecho1
  namespace: ingress-demo
spec:
  selector:
    app: testecho1
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: testecho2
  namespace: ingress-demo
  labels:
    app: testecho2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: testecho2
  template:
    metadata:
      labels:
        app: testecho2
    spec:
      containers:
        - name: testecho2
          image: registry.k8s.io/echoserver:1.4
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: testecho2
  namespace: ingress-demo
spec:
  selector:
    app: testecho2
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

```bash
kubectl apply -f echo-apps.yaml
kubectl get pods,svc -n ingress-demo
```

All four pods should be `Running`.

---

## Lab 5: Create IngressClassParameters and IngressClass

`IngressClassParameters` tells the controller how to build the OCI load balancer. The `IngressClass` links Ingress resources to the controller and to those parameters. Create `ingress-class.yaml` and replace the OCIDs:

```yaml
apiVersion: "ingress.oraclecloud.com/v1beta1"
kind: IngressClassParameters
metadata:
  name: ingressparms-demo
  namespace: ingress-demo
spec:
  compartmentId: "<COMPARTMENT_OCID>"
  subnetId: "<LOAD_BALANCER_SUBNET_OCID>"
  loadBalancerName: "native-ic-demo-lb"
  isPrivate: false
  maxBandwidthMbps: 400
  minBandwidthMbps: 100
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: oci-native-ingress-class
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: oci.oraclecloud.com/native-ingress-controller
  parameters:
    scope: Namespace
    namespace: ingress-demo
    apiGroup: ingress.oraclecloud.com
    kind: ingressclassparameters
    name: ingressparms-demo
```

| Field | Purpose |
| --- | --- |
| `compartmentId`, `subnetId` | Where the OCI load balancer is created |
| `loadBalancerName` | Name shown in OCI |
| `isPrivate` | `false` for a public load balancer, `true` for an internal one |
| `minBandwidthMbps`, `maxBandwidthMbps` | Flexible load balancer bandwidth range |

```bash
kubectl apply -f ingress-class.yaml
kubectl get ingressclass
kubectl get ingressclassparameters -n ingress-demo
```

If you previously created a default IngressClass in the cluster, drop the `is-default-class` annotation here and set `ingressClassName` on the Ingress instead.

---

## Lab 6: Create an Ingress with host and path routing

Create `demo-ingress.yaml`. It routes two paths on one host to the two services:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  namespace: ingress-demo
spec:
  ingressClassName: oci-native-ingress-class
  rules:
    - host: "foo.bar.com"
      http:
        paths:
          - pathType: Exact
            path: "/testecho1"
            backend:
              service:
                name: testecho1
                port:
                  number: 80
          - pathType: Exact
            path: "/testecho2"
            backend:
              service:
                name: testecho2
                port:
                  number: 80
```

```bash
kubectl apply -f demo-ingress.yaml
kubectl get ingress -n ingress-demo -w
```

`ADDRESS` is empty until the load balancer is created. Wait for a public IP. Routing rules, when paths overlap: the longest match wins, and an exact match beats a prefix match of the same length.

Inspect:

```bash
kubectl describe ingress demo-ingress -n ingress-demo
kubectl get events -n ingress-demo --sort-by=.lastTimestamp
```

Optional: enable pod readiness gates so rolling updates only proceed once the load balancer sees new pods as healthy:

```bash
kubectl label ns ingress-demo podreadiness.ingress.oraclecloud.com/pod-readiness-gate-inject=enabled
```

Label the namespace before the pods are created, or restart the deployments afterwards.

---

## Lab 7: Test the routes

```bash
LB_IP=$(kubectl get ingress demo-ingress -n ingress-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "$LB_IP"

curl -s -H "Host: foo.bar.com" http://$LB_IP/testecho1
curl -s -H "Host: foo.bar.com" http://$LB_IP/testecho2
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: foo.bar.com" http://$LB_IP/nothere
```

- The first two should return the echoserver output from `testecho1` and `testecho2`.
- The third path has no rule, so you should get a 404 or another non-200 response.
- The `Host` header is what matches the rule. Alternatively, add a `hosts` entry for `foo.bar.com`.

Show load balancing across pods by repeating a request and watching the `hostname` or pod name in the output change.

---

## Lab 8 (optional): Add TLS

Terminate HTTPS at the load balancer using a Kubernetes TLS Secret. The controller creates the OCI certificate and CA bundle from it.

Create a self-signed certificate for the demo:

```bash
openssl req -x509 -nodes -newkey rsa:2048 -days 30 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=foo.bar.com" -addext "subjectAltName=DNS:foo.bar.com"
cp tls.crt ca.crt

kubectl create secret generic demo-tls-secret \
  --type=kubernetes.io/tls \
  -n ingress-demo \
  --from-file=tls.crt --from-file=tls.key --from-file=ca.crt
```

The Secret needs `ca.crt`, `tls.crt` and `tls.key`. If `ca.crt` is empty, `tls.crt` must hold the full chain with the server certificate first.

Create `demo-ingress-tls.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress-tls
  namespace: ingress-demo
  annotations:
    oci-native-ingress.oraclecloud.com/backend-tls-enabled: "false"
spec:
  ingressClassName: oci-native-ingress-class
  tls:
    - hosts:
        - foo.bar.com
      secretName: demo-tls-secret
  rules:
    - host: "foo.bar.com"
      http:
        paths:
          - pathType: Exact
            path: "/testecho1"
            backend:
              service:
                name: testecho1
                port:
                  number: 80
```

The `backend-tls-enabled: "false"` annotation tells the load balancer to talk plain HTTP to the backends, since the echo servers do not serve TLS.

```bash
kubectl apply -f demo-ingress-tls.yaml
kubectl get ingress -n ingress-demo
curl -sk --resolve foo.bar.com:443:$LB_IP https://foo.bar.com/testecho1
```

`-k` accepts the self-signed certificate. As an alternative to the Secret, you can point an Ingress at an existing OCI certificate with the annotation `oci-native-ingress.oraclecloud.com/certificate-ocid: <CERTIFICATE_OCID>`.

Useful Ingress annotations:

| Annotation | Purpose |
| --- | --- |
| `oci-native-ingress.oraclecloud.com/protocol` | Listener protocol: `HTTP2`, `GRPC` or `TCP` |
| `oci-native-ingress.oraclecloud.com/certificate-ocid` | Use an OCI certificate for TLS termination |
| `oci-native-ingress.oraclecloud.com/backend-tls-enabled` | `"false"` for plain-HTTP backends |
| `oci-native-ingress.oraclecloud.com/http-listener-port`, `https-listener-port` | Listener ports |
| `oci-native-ingress.oraclecloud.com/healthcheck-path` | Backend health check path |
| `oci-native-ingress.oraclecloud.com/id` | Import an existing load balancer |

IngressClass annotations include `oci-native-ingress.oraclecloud.com/waf-policy-ocid` (attach a WAF policy), `network-security-group-ids`, `reserved-private-ip-address-id` (needs `isPrivate: true`), tag annotations, and `delete-protection-enabled`.

---

## Lab 9: Verify in OCI

Check the load balancer the controller created:

```bash
oci lb load-balancer list \
  --compartment-id <COMPARTMENT_OCID> --region $REGION \
  --query 'data[].{name:"display-name",state:"lifecycle-state",ip:"ip-addresses"[0]."ip-address"}' --output table
```

In the Console: **Networking -> Load balancers**, select `native-ic-demo-lb`, and look at:

- **Listeners:** HTTP on port 80, plus HTTPS if you did Lab 8
- **Backend sets:** one per service, with healthy backends
- **Routing policies:** the host and path rules from your Ingress

> **Do not edit this load balancer in the Console, CLI or API.** The controller owns it and reverts changes. Change the Ingress or IngressClass manifests and reapply instead.

What to show in the demo:

1. `kubectl get ingress` with its address.
2. The two `curl` routes answering from different apps.
3. The same routes as listeners, backend sets and routing rules in the Console.
4. Adding a third path to the Ingress, reapplying, and seeing the new rule appear in OCI.

---

## Lab 10: Clean up

Delete Ingress resources **first**, so the controller removes listeners and backend sets. Then the class, apps and namespace:

```bash
kubectl delete ingress --all -n ingress-demo
kubectl delete -f ingress-class.yaml
kubectl delete -f echo-apps.yaml
kubectl delete namespace ingress-demo
```

Confirm in the Console that the load balancer is gone or deleting. Notes:

- To keep the load balancer after deleting the IngressClass, set `oci-native-ingress.oraclecloud.com/delete-protection-enabled: "true"` on the IngressClass first.
- Certificates created from Kubernetes Secrets are **not** removed when an Ingress is deleted. Delete them in the OCI Certificates service, or set `certDeletionGracePeriodInDays` in the Helm `values.yaml`.
- Remove the add-on from the Console (**Cluster -> Add-ons**), or with the CLI `oci ce cluster disable-addon`. For a Helm install, run `helm uninstall oci-native-ingress-controller`.
- Delete the demo certificate files (`tls.key`, `tls.crt`, `ca.crt`).

---

## Troubleshooting

| Symptom | Likely cause | Check |
| --- | --- | --- |
| Ingress `ADDRESS` stays empty | Load balancer still being created, or controller error | `kubectl describe ingress`, controller pod logs |
| Controller pod stuck in `ContainerCreating`, `secret "oci-native-ingress-controller-tls" not found` | cert-manager missing, or installed after the controller | See "Fix: pod stuck on the missing TLS Secret" in Lab 3 |
| Controller pods not running | Add-on not active, or auth failed | `oci ce cluster list-addons`, `kubectl get pods -n native-ingress-controller-system` |
| Controller logs show 401/403/404 | Missing IAM policy or wrong principal | Re-check Lab 2 policies and `authType` |
| Ingress is ignored | Wrong `ingressClassName`, or class not linked to the controller | `kubectl get ingressclass`, `spec.controller` must be `oci.oraclecloud.com/native-ingress-controller` |
| Parameters not found | IngressClass `parameters` point to the wrong namespace or name | Match `namespace` and `name` to the IngressClassParameters |
| `curl` returns 404 | No rule matches host, path or `pathType` | Send the right `Host` header, check `Exact` versus `Prefix` |
| `curl` times out | Security rules block the load balancer or workers | Allow load balancer subnet to workers and pods, and clients to port 80/443 |
| 502/503 from the load balancer | No healthy backends | `kubectl get endpoints -n ingress-demo`, check pod readiness and health check path |
| TLS errors | Bad Secret keys, or backend protocol mismatch | Secret needs `ca.crt`, `tls.crt`, `tls.key`; set `backend-tls-enabled: "false"` for HTTP backends |
| Console changes revert | The controller owns the load balancer | Change manifests, not the OCI resource |

---

## Sources

- [Installing the OCI Native Ingress Controller as a Cluster Add-on](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengsettingupnativeingresscontroller-addon-installing-creating-resources.htm)
- [OCI Native Ingress Controller configuration arguments](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/configuration-arguments-nic.htm)
- [Creating IngressClassParameters, IngressClass, and Ingress Resources](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengsettingupnativeingresscontroller-createresources.htm)
- [oracle/oci-native-ingress-controller: Getting Started](https://github.com/oracle/oci-native-ingress-controller/blob/main/GettingStarted.md)
- [Introducing Oracle Cloud Native Ingress controller for Kubernetes](https://blogs.oracle.com/cloud-infrastructure/oracle-cloud-native-ingress-controller-kubernetes)
- [Installing the OCI Native Ingress Controller as a Standalone Program](https://docs.public.content.oci.oraclecloud.com/en-us/iaas/Content/ContEng/Tasks/contengsettingupnativeingresscontroller-installing-creating-resources.htm)
