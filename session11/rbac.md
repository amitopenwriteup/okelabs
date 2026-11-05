# OKE Lab: Kubernetes RBAC Access Control

**Topics**
1. Demo: Granting the Kubernetes RBAC `cluster-admin` cluster role
2. Demo: Creating a Kubernetes role and role binding to enable a user and group to read pods in a cluster

**Estimated time:** 45-60 minutes
**Level:** Intermediate

---

## Prerequisites

- An OCI tenancy and an OKE cluster (Basic or Enhanced)
- `kubectl` and the **OCI CLI** (Cloud Shell has both preinstalled)
- You are either a **tenancy administrator** or already hold the Kubernetes RBAC `cluster-admin` cluster role on the cluster. You need this to create roles and bindings.
- A second, **non-administrator** OCI IAM user to receive the permissions (for example `jdoe@acme.com`). If you cannot create users, ask your tenancy admin for one.
- An OCI IAM group that contains that user (for example `oke-pod-readers`)

---

# Part 0 - Concept: How OKE Access Control Works

OKE uses two layers of access control, and a user needs both to work with a cluster:

| Layer | What it controls | Where you configure it |
|---|---|---|
| **OCI IAM policy** | Whether the user can access the cluster at all (for example generate a kubeconfig, call the API server) | OCI Console > Identity & Security > Policies |
| **Kubernetes RBAC** | What the user can do inside the cluster (pods, deployments, secrets, namespaces) | `kubectl` with Role, ClusterRole, RoleBinding, ClusterRoleBinding |

Key points:

- A **Role** grants permissions inside one namespace. A **ClusterRole** grants permissions across the whole cluster.
- A **RoleBinding** attaches a Role to a user or group in one namespace. A **ClusterRoleBinding** attaches a ClusterRole to a user or group across the whole cluster.
- The `cluster-admin` ClusterRole is built in and gives super-user access to every resource in every namespace.
- Tenancy administrators already have enough privileges and **do not** need the `cluster-admin` role.
- In RBAC bindings, a user is identified by the **OCID of the IAM user** (`ocid1.user.oc1..`) and a group by the **OCID of the IAM group** (`ocid1.group.oc1..`).
- Each user must create **their own kubeconfig**. A kubeconfig that another user generated does not work for you.

---

# Part 1 - Demo: Granting the Kubernetes RBAC cluster-admin Cluster Role

In this demo, a non-administrator user is given `cluster-admin` on the cluster.

## 1.1 Create the user and group (tenancy administrator)

Skip any item that already exists.

1. Go to **Identity & Security > Domains**, open your domain, and click **Users > Create user**. Enter the user details (for example `jdoe@acme.com`) and click **Create**.
2. Open the user and **copy the user OCID** (`ocid1.user.oc1..`). Keep it for later.
3. Go to **Groups > Create group**, name it `oke-pod-readers`, and click **Create**.
4. Open the group, click **Assign user to group**, and add the user.
5. In the group details, **copy the group OCID** (`ocid1.group.oc1..`). Keep it for Part 2.

## 1.2 Allow the group to use the cluster (OCI IAM policy)

1. Go to **Identity & Security > Policies** and click **Create Policy**.
2. Name it `oke-cluster-access` and select the compartment that holds the cluster (or the root compartment).
3. Add this statement as **one line**:

```text
Allow group oke-pod-readers to use clusters in compartment <compartment-name>
```

> If your group lives in a non-default identity domain, write the group as `'<domain-name>'/oke-pod-readers`.

4. Click **Create**.

This policy gives the `CLUSTER_USE` permission. It lets the user connect to the cluster, but it does not let the user do anything inside it yet.

## 1.3 Connect to the cluster as the administrator

1. Go to **Developer Services > Kubernetes Clusters (OKE)** and open your cluster.
2. Click **Access Cluster**, choose **Cloud Shell Access** (or **Local Access**), and copy the command.
3. Run it. It looks like this:

```bash
oci ce cluster create-kubeconfig \
  --cluster-id <cluster-ocid> \
  --file $HOME/.kube/config \
  --region ap-hyderabad-1 \
  --token-version 2.0.0 \
  --kube-endpoint PUBLIC_ENDPOINT
```

4. Confirm you have access:

```bash
kubectl get nodes
```

## 1.4 Grant cluster-admin to the user

Run as the administrator. Replace the OCID with the user OCID you copied in step 1.1:

```bash
kubectl create clusterrolebinding jdoe-clst-adm \
  --clusterrole=cluster-admin \
  --user=<user-ocid>
```

Expected output:

```text
clusterrolebinding.rbac.authorization.k8s.io/jdoe-clst-adm created
```

Here `jdoe-clst-adm` is a name of your choice for the binding.

## 1.5 Verify

Check that the binding exists:

```bash
kubectl get clusterrolebinding jdoe-clst-adm
```

Then the user must generate their own kubeconfig and run:

```bash
kubectl get pods --all-namespaces
kubectl get nodes
```

Both commands succeed because `cluster-admin` allows every operation in every namespace.

## 1.6 Remove the binding

```bash
kubectl delete clusterrolebinding jdoe-clst-adm
```

**Checkpoint:** The user had full access to the cluster through `cluster-admin`, and you removed it.

> **Security note:** Grant `cluster-admin` only when truly needed. In Part 2 you grant the minimum permissions instead.

---

# Part 2 - Demo: Creating a Role and Role Binding to Read Pods

In this demo, a user and a group get read-only access to pods in the **`default` namespace**, and nothing else.

## 2.1 Check the namespace

This demo uses the built-in `default` namespace, so there is nothing to create. You do not need to deploy any pod.

```bash
kubectl get namespace default
```

## 2.2 Create the Role

`pod-reader-role.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
```

Apply it:

```bash
kubectl apply -f pod-reader-role.yaml
kubectl get role pod-reader -n default
```

This role allows `get`, `watch` and `list` on pods in `default` only. It does not allow creating or deleting pods, and it does not cover other resources.

## 2.3 Create the RoleBinding for a user and a group

`pod-reader-rolebinding.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: default
subjects:
  # A single IAM user, identified by the user OCID
  - kind: User
    name: <user-ocid>
    apiGroup: rbac.authorization.k8s.io
  # An IAM group, identified by the group OCID
  - kind: Group
    name: <group-ocid>
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Replace:

- `<user-ocid>` with the IAM user OCID (`ocid1.user.oc1..`)
- `<group-ocid>` with the IAM group OCID (`ocid1.group.oc1..`)

Apply and check:

```bash
kubectl apply -f pod-reader-rolebinding.yaml
kubectl get rolebinding pod-reader-binding -n default
```

> You can bind just a user, just a group, or both, as in this example. Binding a **group** is easier to manage, because you add or remove people in IAM and the Kubernetes side stays unchanged.

## 2.4 Check the binding and test with impersonation (as administrator)

Confirm the Role and RoleBinding exist and point at the right OCIDs:

```bash
kubectl get role pod-reader -n default
kubectl describe rolebinding pod-reader-binding -n default
```

In the output, check that:

- **Role** is `pod-reader`
- **Subjects** list `User` with your user OCID and `Group` with your group OCID
- **Namespace** is `default`

Then test what the user can do. Use the **complete** user OCID. A shortened or mistyped OCID tests a different identity and gives misleading results.

```bash
# Allowed: read pods in default
kubectl auth can-i list pods -n default --as=<user-ocid>

# Denied: delete pods
kubectl auth can-i delete pods -n default --as=<user-ocid>

# Denied: read secrets
kubectl auth can-i list secrets -n default --as=<user-ocid>
```

Expected results:

| Command | Expected |
|---|---|
| `list pods` | `yes` |
| `delete pods` | `no` |
| `list secrets` | `no` |

> **Note:** On OKE, a denied check can print extra text after the answer, for example `no - default: Authorization failed.` This is normal for a `no` result and not an error. Only the **allowed** check matters for troubleshooting: if `list pods` returns `no`, check the OCID and the binding, then test as the real user in step 2.5.

## 2.5 Verify as the real user

1. Sign in as the non-administrator user and open **Developer Services > Kubernetes Clusters (OKE)** > your cluster > **Access Cluster**.
2. Run the shown `oci ce cluster create-kubeconfig ...` command with the **user's own** credentials (for example in their own Cloud Shell).
3. Run:

```bash
kubectl get pods -n default
```

Expected: the command succeeds. If no pods are running in `default`, it prints `No resources found in default namespace.` That is still a success, because the point is that the command is **allowed** and not `Forbidden`.

4. Try operations that should be blocked:

```bash
kubectl get deployments -n default
kubectl get pods -n kube-system
kubectl get secrets -n default
```

Expected: each returns a `Forbidden` error, for example `pods is forbidden: User "ocid1.user..." cannot list resource "pods" ...`.

**Checkpoint:** The user and group can read pods in `default` and nothing else.

## 2.6 Clean up

```bash
kubectl delete rolebinding pod-reader-binding -n default
kubectl delete role pod-reader -n default
```

In the Console (if no longer needed):
1. Delete the `oke-cluster-access` policy.
2. Remove the user from the group and delete the test user and group.

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `Forbidden` for a user who should have access | RoleBinding uses the wrong OCID, wrong namespace, or the user is not in the bound group |
| User cannot connect to the cluster at all or cannot create a kubeconfig | Missing OCI IAM policy (`Allow group ... to use clusters ...`), or the group is in a different identity domain |
| `error: You must be logged in to the server (Unauthorized)` | The kubeconfig was created by a different user, or the token expired; regenerate the kubeconfig as that user |
| `clusterroles ... is forbidden` when creating a role or binding | You are not a tenancy admin and do not have `cluster-admin`; ask an administrator |
| `kubectl auth can-i ... --as=...` prints `no - default: Authorization failed` | Normal for actions that are not allowed (for example `delete pods`). If `list pods` shows this, verify the full user OCID matches the binding, then test as the real user (step 2.5) |
| `can-i` results do not match the binding | The `--as` OCID is incomplete or mistyped; copy the full OCID from the user's details page |
| Binding created but behaves as if missing | Check the name and namespace: `kubectl get rolebinding -n default -o yaml` |

## Summary

| Stage | Tool | Outcome |
|---|---|---|
| IAM access | OCI IAM policy (`use clusters`) | User can reach the cluster |
| Full access | `ClusterRoleBinding` to `cluster-admin` | User is cluster super-user |
| Least privilege | `Role` + `RoleBinding` | User and group can only read pods in one namespace |
| Verify | `kubectl describe rolebinding`, `kubectl auth can-i --as=...` and real user login | Allowed and denied actions confirmed |
