# Demo: Working with OCI Native Ingress Controller

Install the OCI Native Ingress Controller on an OKE cluster (`ap-hyderabad-1`) and use Kubernetes `Ingress` resources to route HTTP traffic through a single OCI Load Balancer to several services.

The OCI Native Ingress Controller watches `Ingress` resources and manages an OCI Load Balancer for you: listeners, backend sets and routing rules. Unlike a `LoadBalancer` Service (one load balancer per service), one Ingress setup can serve many hosts and paths from a single load balancer.

This guide authenticates the controller as an **OCI user** (an API key stored as a Kubernetes Secret), so it needs no IAM changes of its own.

> **Cost note:** the controller creates a billable OCI load balancer. Delete the Ingress resources at the end of the demo (Lab 10).

## What you will do

| Lab | Step | Done when |
| --- | --- | --- |
| 1 | Check prerequisites, get the OCIDs | Cluster, versions, network, subnet, `CLUSTER_OCID` and `LB_SUBNET_OCID` ready |
| 2 | Prepare the API key and auth config | Unencrypted key and `user-auth-config.yaml` ready |
| 3 | Install cert-manager, then the controller (add-on) | cert-manager pods `Running`, add-on `ACTIVE`, controller pod `1/1 Running` |
| 4 | Deploy two sample applications | Pods up, Services of type `NodePort` |
| 5 | Create IngressClassParameters and IngressClass | Both objects exist, subnet matches the add-on |
| 6 | Create an Ingress with host and path routing | Ingress shows an `ADDRESS` |
| 7 | Test the routes | `curl` returns responses from each app |
| 8 | (Optional) Add TLS | HTTPS listener serves the cert |
| 9 | Verify in OCI | Load balancer, backend sets and health look right |
| 10 | Clean up | Ingress, class, apps, add-on, Secret and load balancer removed |

---

## Lab 1: Prerequisites and OCIDs

- An OKE cluster with a `Ready` worker node and `kubectl` working from Cloud Shell or a VM
- A Kubernetes version supported by the controller (the controller's own docs list 1.27 through 1.36 at the time of writing, so check the current compatibility list)
- Cluster networking set up for load balancers: either OCI VCN-native pod networking (NPN) or Flannel, with security rules that allow the load balancer to reach the workers and pods
- A subnet for the load balancer in the cluster's VCN: **public** for an internet-facing load balancer, or **private** for an internal one
- The OCI CLI configured with an API key (`~/.oci/config`)
- cert-manager, which you install in Lab 3 **before** the controller

### Step 1: Get the compartment OCID

```bash
export REGION=ap-hyderabad-1

# Tenancy OCID: set automatically as $OCI_TENANCY in Cloud Shell.
# Elsewhere, read it from your OCI CLI config:
export OCI_TENANCY=$(grep -m1 '^tenancy' ~/.oci/config | cut -d= -f2 | tr -d ' ')

oci iam compartment list \
  --compartment-id "$OCI_TENANCY" \
  --compartment-id-in-subtree true \
  --access-level ACCESSIBLE \
  --all \
  --query 'data[].{name:name,id:id}' --output table

export COMPARTMENT_OCID='<YOUR_COMPARTMENT_OCID>'
```

If the cluster is in the root compartment, use `export COMPARTMENT_OCID="$OCI_TENANCY"`. If you cannot list compartments (no IAM read access), copy the OCID from the Console instead.

### Step 2: Get the cluster OCID

```bash
oci ce cluster list \
  --compartment-id "$COMPARTMENT_OCID" \
  --region "$REGION" \
  --lifecycle-state ACTIVE \
  --query 'data[].{name:name,id:id}' --output table

export CLUSTER_NAME='<YOUR_CLUSTER_NAME>'

export CLUSTER_OCID=$(oci ce cluster list \
  --compartment-id "$COMPARTMENT_OCID" \
  --region "$REGION" \
  --name "$CLUSTER_NAME" \
  --lifecycle-state ACTIVE \
  --query 'data[0].id' --raw-output)

echo "$CLUSTER_OCID"
```

The output must start with `ocid1.cluster.oc1.`. If it prints `null` or nothing, the name or compartment is wrong, or the cluster is not `ACTIVE`.

### Step 3: Connect kubectl and check the cluster

```bash
oci ce cluster create-kubeconfig \
  --cluster-id "$CLUSTER_OCID" \
  --region "$REGION" \
  --file "$HOME/.kube/config" \
  --token-version 2.0.0

kubectl get nodes
kubectl version
```

### Step 4: Choose the load balancer subnet and get its OCID

```bash
oci network subnet list --compartment-id "$COMPARTMENT_OCID" --region "$REGION" \
  --query 'data[].{name:"display-name",id:id,private:"prohibit-public-ip-on-vnic"}' --output table
```

| `private` column | Subnet type | Use it for |
| --- | --- | --- |
| `False` | Public | An internet-facing load balancer (`isPrivate: false` in Lab 5) |
| `True` | Private | An internal load balancer (`isPrivate: true` in Lab 5), reachable only from inside the VCN |

Do not reuse the Kubernetes API endpoint subnet for the load balancer if you can avoid it: its security rules are usually not set up for ports 80/443 or for reaching the workers. A dedicated load balancer subnet is the cleaner choice.

Set the variable with a **real** OCID, then check it before going on:

```bash
export LB_SUBNET_OCID='<YOUR_LOAD_BALANCER_SUBNET_OCID>'
echo "$COMPARTMENT_OCID"
echo "$LB_SUBNET_OCID"
```

Both lines must print an OCID. If `LB_SUBNET_OCID` is empty or still contains placeholder text, the add-on update in Lab 3 fails with `configurations[1].value size must be between 1 and 8191`.

---

## Lab 2: Prepare the API key and auth config

The controller calls OCI APIs, so it needs an identity. Here it acts as an **OCI user**, using that user's API key, which you store in a Kubernetes Secret in Lab 3. You can use the user your OCI CLI already runs as.

That user's group must already be allowed to manage load balancers, certificates and networking in the compartment. If a permission is missing, the controller logs a `401`, `403` or `404` with the operation name (see Troubleshooting), and the fix is to have a tenancy admin grant it to the user's group (see "Permissions the user needs" at the end of this lab).

> This puts an API private key into the cluster. That is fine for a short demo. Use a dedicated user for anything longer, and delete the Secret and key copies at the end (Lab 10).

**1. Read the values from your OCI CLI config:**

```bash
USER_OCID=$(grep -m1 '^user' ~/.oci/config | cut -d= -f2 | tr -d ' ')
FINGERPRINT=$(grep -m1 '^fingerprint' ~/.oci/config | cut -d= -f2 | tr -d ' ')
KEY_FILE=$(grep -m1 '^key_file' ~/.oci/config | cut -d= -f2 | tr -d ' ')
KEY_FILE=${KEY_FILE/#\~/$HOME}

echo "$USER_OCID" "$FINGERPRINT" "$KEY_FILE"
head -1 "$KEY_FILE"
```

**2. Check the first line of the key.** The controller cannot use an encrypted key. If the Secret holds one, the controller pod crashes with `failed to create client, can not create client, bad configuration: failed to parse private key`.

| First line | What to do |
| --- | --- |
| `-----BEGIN ENCRYPTED PRIVATE KEY-----` (or `Proc-Type: 4,ENCRYPTED` on the next line) | Decrypt it (below) |
| `-----BEGIN RSA PRIVATE KEY-----` or `-----BEGIN PRIVATE KEY-----` | Fine, use it as is |
| `-----BEGIN OPENSSH PRIVATE KEY-----` or `ssh-rsa ...` | This is an SSH key, not an OCI API key. Use the API key file named in `~/.oci/config` |
| `-----BEGIN PUBLIC KEY-----` | You picked the public key. Use the private key file |

To make an unencrypted copy of an encrypted key (the fingerprint stays the same, so nothing changes in OCI). If the key's passphrase is stored in your config, `grep pass_phrase ~/.oci/config` shows it on your screen only:

```bash
openssl rsa -in "$KEY_FILE" -out "$HOME/nic_api_key.pem"
chmod 600 "$HOME/nic_api_key.pem"
head -1 "$HOME/nic_api_key.pem"
export KEY_FILE="$HOME/nic_api_key.pem"
```

The first line must not contain `ENCRYPTED`.

**3. Create the auth config file:**

```bash
cat > user-auth-config.yaml <<EOF
auth:
  region: $REGION
  tenancy: $OCI_TENANCY
  compartment: $COMPARTMENT_OCID
  user: $USER_OCID
  fingerprint: $FINGERPRINT
  passphrase: ""
EOF

cat user-auth-config.yaml
```

Every field must have a value. `passphrase` stays empty because the key is unencrypted now. You create the Secret from this file in Lab 3.

### Permissions the user needs

Check this only if the controller later logs a `401`, `403` or `404`. A tenancy admin can grant these to the group the user belongs to (replace `<GROUP_NAME>` and `<COMPARTMENT_OCID>`). When you use an OCID, the statement must say `in compartment id <ocid>`:

```text
Allow group <GROUP_NAME> to manage load-balancers in compartment id <COMPARTMENT_OCID>
Allow group <GROUP_NAME> to use virtual-network-family in compartment id <COMPARTMENT_OCID>
Allow group <GROUP_NAME> to manage cabundles in compartment id <COMPARTMENT_OCID>
Allow group <GROUP_NAME> to manage cabundle-associations in compartment id <COMPARTMENT_OCID>
Allow group <GROUP_NAME> to manage leaf-certificates in compartment id <COMPARTMENT_OCID>
Allow group <GROUP_NAME> to manage leaf-certificate-versions in compartment id <COMPARTMENT_OCID>
Allow group <GROUP_NAME> to read leaf-certificate-bundles in compartment id <COMPARTMENT_OCID>
Allow group <GROUP_NAME> to manage certificate-associations in compartment id <COMPARTMENT_OCID>
Allow group <GROUP_NAME> to read certificate-authorities in compartment id <COMPARTMENT_OCID>
Allow group <GROUP_NAME> to manage certificate-authority-associations in compartment id <COMPARTMENT_OCID>
Allow group <GROUP_NAME> to read certificate-authority-bundles in compartment id <COMPARTMENT_OCID>
Allow group <GROUP_NAME> to read public-ips in tenancy
Allow group <GROUP_NAME> to manage floating-ips in tenancy
Allow group <GROUP_NAME> to manage waf-family in compartment id <COMPARTMENT_OCID>
Allow group <GROUP_NAME> to read cluster-family in compartment id <COMPARTMENT_OCID>
Allow group <GROUP_NAME> to use tag-namespaces in tenancy
```

These are the controller's sample statements with the subject changed to a group. Confirm the current list on Oracle's prerequisites page: <https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengsettingupnativeingresscontroller-addon-prereqs.htm>. Policies can take a minute or two to apply. The three tenancy-level statements (`public-ips`, `floating-ips`, `tag-namespaces`) are the ones most likely to be missing.

---

## Lab 3: Install cert-manager, then the controller

Do these steps **in this order**. The controller's webhook certificate comes from cert-manager. If cert-manager is missing when the controller is installed, the controller pod stays in `ContainerCreating` with `secret "oci-native-ingress-controller-tls" not found` (see "Fix: pod stuck on the missing TLS Secret" below).

### Step 1: Install cert-manager

```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/latest/download/cert-manager.yaml

kubectl get pods -n cert-manager -w
```

Wait until the three cert-manager pods (`cert-manager`, `cert-manager-cainjector`, `cert-manager-webhook`) are `Running`, then press Ctrl+C. To wait in a script:

```bash
kubectl wait --for=condition=Available deployment --all -n cert-manager --timeout=180s
```

Do not continue until this succeeds.

### Step 2: Install the controller as an OKE add-on

Create the add-on file from your variables. `authType` and `authSecretName` tell the controller to use the Secret you create in Step 3. Check that no `value` is empty before you install.

```bash
cat > native-ingress-controller-addon.json <<EOF
{
  "addonName": "NativeIngressController",
  "configurations": [
    {"key": "compartmentId", "value": "$COMPARTMENT_OCID"},
    {"key": "loadBalancerSubnetId", "value": "$LB_SUBNET_OCID"},
    {"key": "authType", "value": "user"},
    {"key": "authSecretName", "value": "oci-config"}
  ]
}
EOF

cat native-ingress-controller-addon.json
```

The key is `loadBalancerSubnetId` (not `subnetId`). Other arguments, such as `logVerbosity`, are listed here: <https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/configuration-arguments-nic.htm>.

Install:

```bash
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

To change the configuration of an add-on that is already installed, use `update-addon`. It **replaces** the whole configuration, so include every key:

```bash
oci ce cluster update-addon \
  --addon-name NativeIngressController \
  --cluster-id "$CLUSTER_OCID" \
  --configurations file://addon-config.json
```

where `addon-config.json` is a JSON array of the `{"key": ..., "value": ...}` entries shown above.

### Step 3: Create the auth Secret

The Secret must be in the add-on's namespace, which exists once the add-on is installed:

```bash
kubectl get ns native-ingress-controller-system

kubectl create secret generic oci-config \
  --from-file=config=user-auth-config.yaml \
  --from-file=private-key="$KEY_FILE" \
  -n native-ingress-controller-system

kubectl rollout restart deployment -n native-ingress-controller-system
```

The Secret holds exactly two keys, `config` and `private-key`. Until it exists, the controller pod crashes or waits. If you need to replace it, delete it first with `kubectl delete secret oci-config -n native-ingress-controller-system`.

### Step 4: Verify the controller

```bash
kubectl get pods -n native-ingress-controller-system
kubectl get ingressclass
```

The controller pod should be `1/1 Running` with no climbing restart count. If the namespace differs on your cluster, find it with `kubectl get pods -A | grep -i ingress`.

Read the log once. Normal output ends with the controller acquiring the lease and starting its loops:

```bash
kubectl logs -n native-ingress-controller-system -l app.kubernetes.io/name=oci-native-ingress-controller --tail=40 \
  | grep -v -E "caches populated|shared_informer|reflector|warnings.go"
```

- `failed to acquire lease` / `lock is held by ...` lines are **normal** while a restarted pod waits for the previous instance's lease to expire (about a minute). A pod that has not yet become the leader only logs these lines. If the lease holder ID keeps changing, a controller is crashing: check `kubectl logs ... --previous` and see Troubleshooting.
- `successfully acquired lease`, then `Controller loop...` and `Fetching auth config provider for type: user` means authentication is being set up. A `401`, `403` or `404` after that points to a permission problem.

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
| No `cert-manager` namespace or CRDs | cert-manager missing | Install it (Step 1), wait for its three pods to be `Running` |
| cert-manager is running, but step 1 shows no Certificate or Issuer | cert-manager was installed after the controller, so the objects were never created | Remove the add-on and install it again (below) |
| Certificate exists but is not `Ready` | Issuance is failing | `kubectl describe certificate -n native-ingress-controller-system`, then `kubectl describe certificaterequest -n native-ingress-controller-system` and read the events |

To remove and reinstall the add-on:

```bash
oci ce cluster disable-addon \
  --addon-name NativeIngressController \
  --cluster-id "$CLUSTER_OCID" \
  --is-remove-existing-add-on true \
  --force

oci ce cluster list-addons --cluster-id "$CLUSTER_OCID"
```

When `NativeIngressController` no longer appears in the list, run the `install-addon` command from Step 2 again, then recreate the `oci-config` Secret (Step 3) if it was removed with the namespace. You can also do the removal in the Console: **Kubernetes Clusters -> your cluster -> Add-ons -> NativeIngressController -> Remove**.

**3. Check the result:**

```bash
kubectl get secret oci-native-ingress-controller-tls -n native-ingress-controller-system
kubectl get pods -n native-ingress-controller-system -w
```

Once the Secret exists, the kubelet mounts it on its next retry and the pod starts. If it does not, delete the pod and the Deployment recreates it:

```bash
kubectl delete pod -n native-ingress-controller-system -l app.kubernetes.io/name=oci-native-ingress-controller
```

If the pod starts but fails afterwards, read its logs (`--previous` for the last crashed run). The next likely causes are an unusable private key (Lab 2) or a `401`, `403` or `404` from a missing permission.

---

## Lab 4: Deploy two sample applications

Create a namespace, and two echo servers that report which pod answered:

```bash
kubectl create namespace ingress-demo
```

Create `echo-apps.yaml`. The Services are `NodePort`: on a **Flannel** cluster the controller registers the worker nodes as load balancer backends on each Service's NodePort, so a `ClusterIP` Service leaves the backend sets empty and the load balancer returns `502 Bad Gateway`.

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
  type: NodePort
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
  type: NodePort
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

All four pods should be `Running`, and each Service should show `TYPE NodePort` with a port like `80:3xxxx/TCP`.

If you already created the Services as `ClusterIP`, switch them in place:

```bash
kubectl patch svc testecho1 -n ingress-demo -p '{"spec":{"type":"NodePort"}}'
kubectl patch svc testecho2 -n ingress-demo -p '{"spec":{"type":"NodePort"}}'
```

---

## Lab 5: Create IngressClassParameters and IngressClass

`IngressClassParameters` tells the controller how to build the OCI load balancer. The `IngressClass` links Ingress resources to the controller and to those parameters. Create `ingress-class.yaml` from your variables. Set `isPrivate` to match the subnet you chose in Lab 1: `false` for a public subnet, `true` for a private one.

```bash
export LB_PRIVATE=false     # true if LB_SUBNET_OCID is a private subnet

cat > ingress-class.yaml <<EOF
apiVersion: "ingress.oraclecloud.com/v1beta1"
kind: IngressClassParameters
metadata:
  name: ingressparms-demo
  namespace: ingress-demo
spec:
  compartmentId: "$COMPARTMENT_OCID"
  subnetId: "$LB_SUBNET_OCID"
  loadBalancerName: "native-ic-demo-lb"
  isPrivate: $LB_PRIVATE
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
EOF

cat ingress-class.yaml
```

| Field | Purpose |
| --- | --- |
| `compartmentId`, `subnetId` | Where the OCI load balancer is created. Use the same subnet as the add-on's `loadBalancerSubnetId` |
| `loadBalancerName` | Name shown in OCI |
| `isPrivate` | `false` for a public load balancer, `true` for an internal one |
| `minBandwidthMbps`, `maxBandwidthMbps` | Flexible load balancer bandwidth range |

```bash
kubectl apply -f ingress-class.yaml
kubectl get ingressclass
kubectl get ingressclassparameters -n ingress-demo
```

If you previously created a default IngressClass in the cluster, drop the `is-default-class` annotation here and set `ingressClassName` on the Ingress instead.

To change the subnet or visibility of an IngressClassParameters you already applied:

```bash
kubectl patch ingressclassparameters ingressparms-demo -n ingress-demo --type merge \
  -p "{\"spec\":{\"subnetId\":\"$LB_SUBNET_OCID\",\"isPrivate\":$LB_PRIVATE}}"

kubectl get ingressclassparameters ingressparms-demo -n ingress-demo -o yaml | grep -E 'subnetId|isPrivate'
```

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

`ADDRESS` is empty until the controller has created the load balancer, usually 1 to 3 minutes after the controller pod is healthy and has become the leader. For a public load balancer the address is a public IP, and for an internal one it is a private IP. Routing rules, when paths overlap: the longest match wins, and an exact match beats a prefix match of the same length.

Inspect:

```bash
kubectl describe ingress demo-ingress -n ingress-demo
kubectl get events -n ingress-demo --sort-by=.lastTimestamp
```

Check that exactly one load balancer was created. The controller log can briefly show a second "Creating load balancer" request right after saving the first one's ID, so confirm there is no duplicate:

```bash
kubectl get ingressclass oci-native-ingress-class \
  -o jsonpath='{.metadata.annotations.oci-native-ingress\.oraclecloud\.com/id}{"\n"}'

oci lb load-balancer list --compartment-id "$COMPARTMENT_OCID" --region "$REGION" \
  --query 'data[].{name:"display-name",state:"lifecycle-state",id:id}' --output table
```

If there are two `native-ic-demo-lb` entries, the one whose `id` matches the IngressClass annotation is in use. The other is an orphan that you pay for, so delete it with `oci lb load-balancer delete --load-balancer-id <ORPHAN_LB_OCID> --force` after checking its ID.

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

- If `LB_IP` is empty, the Ingress has no address yet. Go back to Lab 6 and Troubleshooting.
- The first two should return the echoserver output from `testecho1` and `testecho2`.
- The third path has no rule, so you should get a 404 or another non-200 response.
- The `Host` header is what matches the rule. Alternatively, add a `hosts` entry for `foo.bar.com`.
- With an **internal** (private) load balancer, run the `curl` commands from a machine inside the VCN, such as the jumpbox, and make sure the subnet security rules allow port 80.

Show load balancing across pods by repeating a request and watching the `hostname` or pod name in the output change.

**If you get `502 Bad Gateway`:** the load balancer is up but has no healthy backends. Check, in order:

```bash
kubectl get svc -n ingress-demo     # must be NodePort on Flannel

LB_OCID=$(kubectl get ingressclass oci-native-ingress-class \
  -o jsonpath='{.metadata.annotations.oci-native-ingress\.oraclecloud\.com/id}')

oci lb backend-set list --load-balancer-id "$LB_OCID" \
  --query 'data[].{name:name,backends:backends[].{ip:"ip-address",port:port}}'

oci lb load-balancer-health get --load-balancer-id "$LB_OCID" \
  --query 'data.{status:status,critical:"critical-state-backend-set-names"}'
```

- **Empty `backends` in the `bs_...` sets, health `UNKNOWN`:** the Services are `ClusterIP`. Patch them to `NodePort` (Lab 4), wait a minute, and check again. If they are already `NodePort`, force a resync with `kubectl annotate ingress demo-ingress -n ingress-demo resync="$(date +%s)" --overwrite` and read the controller log. `default_ingress` is the default backend set and stays empty when the Ingress has no default backend, which is normal.
- **Backends listed but health `CRITICAL`:** the network rules are blocking the load balancer. The worker node subnet's security list or NSG must allow the load balancer subnet's CIDR on TCP 30000-32767 (the NodePort range), and the load balancer subnet must allow egress to the workers on the same range.

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

Check the load balancer the controller created, and its health:

```bash
oci lb load-balancer list \
  --compartment-id "$COMPARTMENT_OCID" --region "$REGION" \
  --query 'data[].{name:"display-name",state:"lifecycle-state",ip:"ip-addresses"[0]."ip-address"}' --output table

oci lb load-balancer-health get --load-balancer-id "$LB_OCID" \
  --query 'data.{status:status,critical:"critical-state-backend-set-names"}'
```

`status` should be `OK` once the backends pass their health checks.

In the Console: **Networking -> Load balancers**, select `native-ic-demo-lb`, and look at:

- **Listeners:** HTTP on port 80, plus HTTPS if you did Lab 8
- **Backend sets:** one per service, with healthy backends (your worker node IPs on the Services' NodePorts)
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

Confirm in the Console that the load balancer is gone or deleting. Then remove the add-on and, if you no longer need it, cert-manager:

```bash
oci ce cluster disable-addon \
  --addon-name NativeIngressController \
  --cluster-id "$CLUSTER_OCID" \
  --is-remove-existing-add-on true

oci ce cluster list-addons --cluster-id "$CLUSTER_OCID"

kubectl delete -f https://github.com/jetstack/cert-manager/releases/latest/download/cert-manager.yaml
```

Remove the credentials. The decrypted key is an unprotected copy of your API key:

```bash
kubectl delete secret oci-config -n native-ingress-controller-system
rm -f "$HOME/nic_api_key.pem" user-auth-config.yaml native-ingress-controller-addon.json addon-config.json
```

Notes:

- `--is-remove-existing-add-on true` removes the add-on and its resources from the cluster. With `false`, the add-on is only disabled and its existing resources stay in place. Add `--force` to skip the confirmation prompt.
- If the add-on is already removed, the `oci-config` Secret may be gone with its namespace, and the `kubectl delete secret` command then reports "not found", which is fine.
- You can also remove the add-on from the Console (**Cluster -> Add-ons**).
- To keep the load balancer after deleting the IngressClass, set `oci-native-ingress.oraclecloud.com/delete-protection-enabled: "true"` on the IngressClass first.
- Certificates created from Kubernetes Secrets are **not** removed when an Ingress is deleted. Delete them in the OCI Certificates service.
- Delete the demo certificate files (`tls.key`, `tls.crt`, `ca.crt`).

---

## Troubleshooting

| Symptom | Likely cause | Check or fix |
| --- | --- | --- |
| `CLUSTER_OCID` is empty or `null` | Wrong cluster name or compartment, or cluster not `ACTIVE` | Re-run the `oci ce cluster list` command in Lab 1, Step 2 |
| `update-addon` fails with `configurations[1].value size must be between 1 and 8191` | A configuration value is empty, usually `LB_SUBNET_OCID` | `cat` the JSON file and `echo` the variables. Set a real OCID (not placeholder text) and regenerate the file |
| Ingress `ADDRESS` stays empty | Controller not running or not the leader, or an error while creating the load balancer | `kubectl get pods -n native-ingress-controller-system`, then controller logs (including `--previous`) |
| Controller pod `ContainerCreating`, `secret "oci-native-ingress-controller-tls" not found` | cert-manager missing, or installed after the controller | See "Fix: pod stuck on the missing TLS Secret" in Lab 3 |
| Controller crashes with `failed to create client, can not create client, bad configuration: failed to parse private key` | The Secret holds an encrypted or wrong private key | `head -1` the key. It must not say `ENCRYPTED`. Decrypt it (Lab 2), recreate the `oci-config` Secret, and restart the deployment |
| Secret errors such as `Empty Configuration` or `error retrieving secret` | `oci-config` missing, in the wrong namespace, or missing `config` / `private-key` keys | `kubectl get secret oci-config -n native-ingress-controller-system -o yaml` shows both keys |
| Logs show `401`, `403` or `404` / `NotAuthorizedOrNotFound` with an operation name | The OCI user is missing that permission, or the user, tenancy or fingerprint in `user-auth-config.yaml` is wrong | Check the values in the Secret's `config`, and ask a tenancy admin to grant the permission to the user's group (Lab 2, "Permissions the user needs") |
| Logs repeat `failed to acquire lease` / `lock is held by ...` | A restarted pod waiting for the old leader's lease, or a crash loop | Wait about a minute. If the holder ID keeps changing, read `--previous` logs. If one stale holder never expires, `kubectl delete lease oci-native-ingress-controller -n native-ingress-controller-system` |
| Ingress is ignored | Wrong `ingressClassName`, or class not linked to the controller | `kubectl get ingressclass`, `spec.controller` must be `oci.oraclecloud.com/native-ingress-controller` |
| Parameters not found | IngressClass `parameters` point to the wrong namespace or name | Match `namespace` and `name` to the IngressClassParameters |
| Two load balancers with the same name | Race while the controller saved the first load balancer's ID | Keep the one whose `id` matches the IngressClass annotation `oci-native-ingress.oraclecloud.com/id`, delete the other |
| `curl` returns 404 | No rule matches host, path or `pathType` | Send the right `Host` header, check `Exact` versus `Prefix` |
| `curl` times out | Security rules block the load balancer or clients, or a private load balancer is called from outside the VCN | Allow clients to port 80/443, and call internal load balancers from inside the VCN |
| `502`/`503` from the load balancer, empty backend sets | Services are `ClusterIP` on a Flannel cluster | Patch the Services to `NodePort` (Lab 4) |
| `502`/`503`, backends present, health `CRITICAL` | Security rules block the load balancer from the workers | Allow the load balancer subnet CIDR to the workers on TCP 30000-32767, and egress on the same range |
| TLS errors | Bad Secret keys, or backend protocol mismatch | Secret needs `ca.crt`, `tls.crt`, `tls.key`; set `backend-tls-enabled: "false"` for HTTP backends |
| Console changes revert | The controller owns the load balancer | Change manifests, not the OCI resource |

---

## Sources

- [Installing the OCI Native Ingress Controller as a Cluster Add-on](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengsettingupnativeingresscontroller-addon-installing-creating-resources.htm)
- [Prerequisites for deploying the OCI Native Ingress Controller as a Cluster Add-on](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengsettingupnativeingresscontroller-addon-prereqs.htm)
- [OCI Native Ingress Controller configuration arguments](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/configuration-arguments-nic.htm)
- [Creating IngressClassParameters, IngressClass, and Ingress Resources](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengsettingupnativeingresscontroller-createresources.htm)
- [oracle/oci-native-ingress-controller: Getting Started](https://github.com/oracle/oci-native-ingress-controller/blob/main/GettingStarted.md)
- [Introducing Oracle Cloud Native Ingress controller for Kubernetes](https://blogs.oracle.com/cloud-infrastructure/oracle-cloud-native-ingress-controller-kubernetes)
