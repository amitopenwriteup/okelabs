# Lab: Access Control on OKE: Service Account Tokens and Kubernetes RBAC

**Duration:** ~60–75 minutes
**Platform:** Oracle Container Engine for Kubernetes (OKE), existing cluster

---

## Lab Objectives

1. Explain how access control works in OKE (OCI IAM + Kubernetes RBAC).
2. Add a **service account authentication token** to a kubeconfig.
3. Grant the Kubernetes **cluster-admin** ClusterRole to a user.
4. Create a **Role and RoleBinding** so a user and a group can read pods in one namespace.
5. Create a **ClusterRole and ClusterRoleBinding** so a user and a group can list secrets cluster-wide.

## Concepts: Access Control in Container Engine for Kubernetes

OKE uses two layers of access control that work together.

| Layer | Controlled by | Answers the question |
|---|---|---|
| **OCI IAM** | IAM policies | *Can this OCI user reach the cluster and its API endpoint at all?* (create, view, update clusters; generate a kubeconfig/token) |
| **Kubernetes RBAC** | Roles, ClusterRoles and bindings | *What can this identity do inside the cluster?* (read pods, list secrets, deploy, and so on) |

```
kubectl ──► token (OCI CLI exec, or service account token) ──► API server
                                                                │
                    Authentication: who are you?  ◄─────────────┤
                    Authorization (RBAC): may you do this?  ◄────┘
```

**Authentication options in a kubeconfig:**

- **OCI CLI token (default):** the kubeconfig runs `oci ce cluster generate-token`. The identity is the **OCI user** (its user OCID is the RBAC subject name).
- **Service account token:** a long-lived or short-lived bearer token for a Kubernetes service account. Useful for automation, CI/CD and tools that can't use the OCI CLI.

**RBAC objects:**

| Object | Scope | Purpose |
|---|---|---|
| `Role` | One namespace | Set of permissions (verbs on resources) |
| `ClusterRole` | Whole cluster | Permissions, including cluster-scoped resources |
| `RoleBinding` | One namespace | Gives a Role or ClusterRole to subjects within that namespace |
| `ClusterRoleBinding` | Whole cluster | Gives a ClusterRole to subjects across all namespaces |

**Subjects** are `User`, `Group` or `ServiceAccount`. For OCI users, the `User` name is the **user OCID** (`ocid1.user.oc1..xxxx`).

> **Note on groups:** Kubernetes `Group` subjects match groups presented by the authenticator. Check the current OKE documentation to see how OCI IAM groups are (or aren't) passed to Kubernetes RBAC for your authentication method. In this lab you will test group bindings safely using **impersonation** (`--as-group`) and service account groups, which behave the same way inside RBAC.

## Prerequisites

- An existing OKE cluster and working `kubectl` access as an **administrator** (cluster creator or an existing cluster-admin).
- OCI CLI configured.
- The OCID of at least one OCI user to use as the demo user. Find it in the Console (**Identity & Security → Users → user details → OCID**) or:

```bash
oci iam user list --query 'data[].{name:name,id:id}' --output table
```

Set variables:

```bash
export DEMO_USER_OCID="ocid1.user.oc1..xxxxxxxx"     # the user you will grant access to
export DEMO_GROUP="dev-readers"                       # a Kubernetes group name for the demo
export NS="rbac-demo"
```

Confirm your current access:

```bash
kubectl config current-context
kubectl auth can-i '*' '*' --all-namespaces      # should print "yes" for an admin
kubectl create namespace $NS
```

### OCI IAM policy (background)

For a non-admin OCI user to use `kubectl` at all, IAM must allow them to access the cluster, for example:

```
Allow group <iam-group> to use cluster-family in compartment <compartment-name>
```

IAM gets the user *to* the API server; RBAC then decides what they can do. A user with IAM access but no RBAC binding is authenticated but **forbidden** from doing anything useful.

---

## Part 1: Adding a Service Account Authentication Token to a Kubeconfig

### Step 1.1: Create a Service Account

```bash
kubectl -n kube-system create serviceaccount kubeconfig-sa
```

### Step 1.2: Give the Service Account Permissions

For this demo, bind it to cluster-admin. In real use, grant the **least privilege** needed.

```bash
kubectl create clusterrolebinding kubeconfig-sa-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:kubeconfig-sa
```

### Step 1.3: Create a Token Secret

Since Kubernetes 1.24, token Secrets are no longer created automatically. Create one explicitly:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: kubeconfig-sa-token
  namespace: kube-system
  annotations:
    kubernetes.io/service-account.name: kubeconfig-sa
type: kubernetes.io/service-account-token
EOF
```

Wait a few seconds, then confirm the token was populated:

```bash
kubectl -n kube-system describe secret kubeconfig-sa-token
```

> **Short-lived alternative:** `kubectl -n kube-system create token kubeconfig-sa --duration=1h` issues a time-limited token without a stored Secret. Prefer this for anything temporary.

### Step 1.4: Extract the Token

```bash
export SA_TOKEN=$(kubectl -n kube-system get secret kubeconfig-sa-token -o jsonpath='{.data.token}' | base64 -d)
echo ${SA_TOKEN:0:20}...      # show only the beginning; treat the full token as a password
```

### Step 1.5: Add the Token to a Kubeconfig

Work on a **separate kubeconfig copy** so you don't disturb your normal login:

```bash
cp ~/.kube/config ~/.kube/config-sa
export KUBECONFIG=~/.kube/config-sa

CURRENT_CONTEXT=$(kubectl config current-context)
CLUSTER_NAME=$(kubectl config view --minify -o jsonpath='{.contexts[0].context.cluster}')

kubectl config set-credentials kubeconfig-sa --token=$SA_TOKEN
kubectl config set-context sa-context --cluster=$CLUSTER_NAME --user=kubeconfig-sa
kubectl config use-context sa-context
```

### Step 1.6: Verify

```bash
kubectl config get-contexts
kubectl get nodes
kubectl auth whoami 2>/dev/null || kubectl auth can-i '*' '*' --all-namespaces
```

**Expected:** commands succeed using the service account identity (`system:serviceaccount:kube-system:kubeconfig-sa`), without involving the OCI CLI.

Switch back to your normal kubeconfig:

```bash
unset KUBECONFIG
kubectl config current-context
```

> **Security:** anyone holding the token has the service account's permissions until the Secret/ServiceAccount is deleted. Store it in a secret manager, never in Git, and rotate regularly.

---

## Part 2: Granting the Kubernetes RBAC cluster-admin ClusterRole

`cluster-admin` is a built-in ClusterRole granting full control over all resources in all namespaces. Grant it sparingly.

### Step 2.1: Check That the User Has No Rights Yet

Use impersonation (requires your admin identity):

```bash
kubectl auth can-i list pods --all-namespaces --as=$DEMO_USER_OCID
```

**Expected:** `no`.

### Step 2.2: Create the ClusterRoleBinding

Declarative form:

```bash
cat <<EOF > cluster-admin-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: demo-user-cluster-admin
subjects:
  - kind: User
    name: ${DEMO_USER_OCID}
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f cluster-admin-binding.yaml
```

Or imperatively:

```bash
kubectl create clusterrolebinding demo-user-cluster-admin \
  --clusterrole=cluster-admin --user=$DEMO_USER_OCID
```

### Step 2.3: Verify

```bash
kubectl get clusterrolebinding demo-user-cluster-admin -o yaml
kubectl auth can-i '*' '*' --all-namespaces --as=$DEMO_USER_OCID
kubectl auth can-i delete nodes --as=$DEMO_USER_OCID
```

**Expected:** `yes` for each.

### Step 2.4: Revoke It

Don't leave this in place after the demo:

```bash
kubectl delete clusterrolebinding demo-user-cluster-admin
kubectl auth can-i list pods --all-namespaces --as=$DEMO_USER_OCID     # back to "no"
```

---

## Part 3: Role and RoleBinding to Read Pods in a Namespace (User and Group)

**Goal:** a user and a group can `get`, `list` and `watch` pods in the `rbac-demo` namespace only.

### Step 3.1: Create Sample Pods

```bash
kubectl -n $NS run web --image=nginx:stable --restart=Never
kubectl -n default run other --image=nginx:stable --restart=Never
kubectl -n $NS get pods
```

### Step 3.2: Create the Role

```bash
cat <<EOF > pod-reader-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: ${NS}
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
EOF
kubectl apply -f pod-reader-role.yaml
```

### Step 3.3: Create the RoleBinding (User and Group)

```bash
cat <<EOF > pod-reader-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: ${NS}
subjects:
  - kind: User
    name: ${DEMO_USER_OCID}
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: ${DEMO_GROUP}
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f pod-reader-binding.yaml
```

### Step 3.4: Verify the User

```bash
kubectl auth can-i list pods -n $NS --as=$DEMO_USER_OCID          # yes
kubectl auth can-i get pods -n $NS --as=$DEMO_USER_OCID           # yes
kubectl auth can-i delete pods -n $NS --as=$DEMO_USER_OCID        # no  (verb not granted)
kubectl auth can-i list pods -n default --as=$DEMO_USER_OCID      # no  (other namespace)
kubectl auth can-i list secrets -n $NS --as=$DEMO_USER_OCID       # no  (other resource)

kubectl get pods -n $NS --as=$DEMO_USER_OCID
kubectl get pods -n default --as=$DEMO_USER_OCID                  # Forbidden
```

### Step 3.5: Verify the Group

Simulate a member of the group by impersonating an arbitrary user in that group:

```bash
kubectl auth can-i list pods -n $NS --as=someone-else --as-group=$DEMO_GROUP         # yes
kubectl auth can-i list pods -n $NS --as=someone-else --as-group=unrelated-group     # no
```

### Step 3.6: Inspect

```bash
kubectl -n $NS get role,rolebinding
kubectl -n $NS describe rolebinding read-pods
```

---

## Part 4: ClusterRole and ClusterRoleBinding to List Secrets Cluster-Wide (User and Group)

**Goal:** a user and a group can `get` and `list` secrets in **all** namespaces.

> **Warning:** reading secrets exposes credentials and tokens. In RBAC, `list` and `get` on secrets reveal secret *contents*. Grant this only where truly required.

### Step 4.1: Create Sample Secrets

```bash
kubectl -n $NS create secret generic app-secret --from-literal=password=S3cr3t
kubectl -n default create secret generic other-secret --from-literal=token=abc123
```

### Step 4.2: Check Current Access

```bash
kubectl auth can-i list secrets --all-namespaces --as=$DEMO_USER_OCID     # no
```

### Step 4.3: Create the ClusterRole

```bash
cat <<'EOF' > secret-reader-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]
EOF
kubectl apply -f secret-reader-clusterrole.yaml
```

### Step 4.4: Create the ClusterRoleBinding (User and Group)

```bash
cat <<EOF > secret-reader-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
  - kind: User
    name: ${DEMO_USER_OCID}
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: ${DEMO_GROUP}
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f secret-reader-binding.yaml
```

### Step 4.5: Verify

```bash
# User
kubectl auth can-i list secrets --all-namespaces --as=$DEMO_USER_OCID       # yes
kubectl auth can-i list secrets -n kube-system --as=$DEMO_USER_OCID         # yes
kubectl auth can-i delete secrets -n $NS --as=$DEMO_USER_OCID               # no
kubectl auth can-i list configmaps -n $NS --as=$DEMO_USER_OCID              # no

# Group
kubectl auth can-i list secrets --all-namespaces --as=someone-else --as-group=$DEMO_GROUP   # yes
kubectl auth can-i list secrets -n default --as=someone-else --as-group=unrelated-group     # no

# Actually read
kubectl get secrets -A --as=$DEMO_USER_OCID
```

### Step 4.6: Compare Scopes (Optional)

Bind the same ClusterRole with a **RoleBinding** to limit it to a single namespace:

```bash
kubectl -n $NS create rolebinding secrets-ns-only --clusterrole=secret-reader --user=$DEMO_USER_OCID
```

A RoleBinding that references a ClusterRole grants those permissions only in the binding's namespace. This is the least-privilege way to reuse a ClusterRole.

### Step 4.7: Review Who Can Do What

```bash
kubectl get clusterrolebinding,rolebinding -A | grep -E "demo|read-"
kubectl auth can-i --list --as=$DEMO_USER_OCID -n $NS
```

---

## Verification Checklist

- [ ] A kubeconfig context using a service account token works (`kubectl get nodes`).
- [ ] The demo user got `cluster-admin`, then lost it after you deleted the binding.
- [ ] The user and group can list pods **only** in `rbac-demo`, and cannot delete pods or read secrets.
- [ ] The user and group can list secrets in **all** namespaces but cannot delete them.
- [ ] You can explain the difference between a Role/RoleBinding and a ClusterRole/ClusterRoleBinding.

## Troubleshooting Quick Reference

| Problem | Likely Cause | Action |
|---|---|---|
| `error: You must be logged in to the server (Unauthorized)` | Token invalid, expired or deleted | Recreate the token; confirm the Secret has a `token` value |
| Secret has no token data | Secret created before the ServiceAccount, or wrong annotation | Check `kubernetes.io/service-account.name` and recreate |
| `Forbidden: cannot list resource` | No matching RBAC binding | `kubectl auth can-i ... --as=...`; review bindings |
| Binding exists but access denied | Wrong subject name (user OCID typo) or wrong namespace | `kubectl describe rolebinding/clusterrolebinding`; check exact OCID |
| `--as` is forbidden | Your own identity lacks impersonation rights | Run as cluster-admin |
| User can't reach cluster at all | Missing OCI IAM policy or kubeconfig | Add the IAM policy; regenerate kubeconfig with `oci ce cluster create-kubeconfig` |
| Group binding has no effect for real users | The authenticator isn't presenting that group | Verify group handling in the OKE docs; bind by user OCID or use a supported group mechanism |
| Role works in one namespace only | Expected behavior | Use a ClusterRoleBinding for cluster-wide access |

## Cleanup

```bash
kubectl delete clusterrolebinding read-secrets-global kubeconfig-sa-admin demo-user-cluster-admin --ignore-not-found
kubectl delete clusterrole secret-reader --ignore-not-found
kubectl delete secret other-secret -n default --ignore-not-found
kubectl delete pod other -n default --ignore-not-found
kubectl -n kube-system delete secret kubeconfig-sa-token --ignore-not-found
kubectl -n kube-system delete serviceaccount kubeconfig-sa --ignore-not-found
kubectl delete namespace $NS          # removes the Role, RoleBindings, pods and secrets inside
rm -f ~/.kube/config-sa
unset KUBECONFIG SA_TOKEN
```

## Summary

| Topic | Key Takeaway |
|---|---|
| Access model | IAM decides who can reach the cluster; RBAC decides what they can do inside it |
| Service account token | Lets automation authenticate without the OCI CLI; protect it like a password and prefer short-lived tokens |
| cluster-admin | Full control everywhere; grant rarely and revoke when done |
| Role + RoleBinding | Namespaced permissions (read pods in one namespace) |
| ClusterRole + ClusterRoleBinding | Cluster-wide permissions (list secrets everywhere); use a RoleBinding to scope a ClusterRole to one namespace |
| Testing | `kubectl auth can-i --as=<user> [--as-group=<group>]` checks RBAC without logging in as that user |
