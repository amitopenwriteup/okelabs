# Lab: Adding a Service Account Authentication Token to a Kubeconfig File in OKE

**Estimated time:** 40–50 minutes
**Level:** Intermediate

> **Note:** OCI Console labels and Kubernetes behaviour change between versions. Items most likely to differ are marked **(verify)**.

---

## Concept

By default, the kubeconfig that OKE generates does **not** contain a password or token. It contains a command (the OCI CLI) that fetches a short-lived token for **your OCI user** each time you run `kubectl`. That is secure, but it ties access to an OCI identity and needs the OCI CLI on the machine.

A **service account token** is the alternative. You create a Kubernetes service account, give it permissions with RBAC, and put its token into a kubeconfig. This is useful for:

- Automation and CI/CD pipelines that cannot use an interactive OCI login
- Tools or machines where the OCI CLI is not installed
- Giving a person or tool access limited to one namespace

| | Default OKE kubeconfig | Service account token kubeconfig |
|---|---|---|
| Identity | Your OCI IAM user | A Kubernetes service account |
| Credential | Short-lived token fetched by the OCI CLI | Token stored in the file |
| Needs OCI CLI at run time | Yes | No |
| Permissions controlled by | OCI IAM policy and Kubernetes RBAC | Kubernetes RBAC only |
| Risk if the file leaks | Token expires quickly | Anyone with the file has the account's access until the token is revoked or expires |

## Rules You Must Know

| Rule | Why it matters |
|---|---|
| Treat the token like a password | It is a bearer credential. Anyone holding it can act as the service account. |
| Give **least privilege** | Bind the account to a namespace role rather than a cluster-wide one |
| Prefer **short-lived** tokens | They expire on their own and limit damage from a leak |
| Never commit the kubeconfig to source control | Keep it out of Git and shared drives |

## Objective

1. Create a service account and give it namespace-limited permissions.
2. Generate a short-lived token.
3. Add the token to a kubeconfig as a new user and context.
4. Verify access and its limits.
5. Revoke the access and clean up.

## Prerequisites

| Requirement | Details |
|---|---|
| Existing OKE cluster | With a working `kubectl` connection (for example from Part 1, Lab 1) |
| Admin access | Your current kubeconfig must be allowed to create service accounts and role bindings |
| `kubectl` | Version close to the cluster version |
| Cluster API reachable | The machine using the new kubeconfig must be able to reach the API endpoint |

---

## Step 1 – Inspect Your Current Kubeconfig

```bash
kubectl config current-context
kubectl config get-contexts
kubectl config view --minify
```

In the output, find these items and note them:

| Item | Where to find it | Example |
|---|---|---|
| **Cluster name** | `clusters[].name` | `cluster-cabcdef1234` |
| **Server URL** | `clusters[].cluster.server` | `https://<ip>:6443` |
| **Existing user** | `users[].user.exec` | An `exec` block that runs `oci ce cluster generate-token` |

Get the cluster name directly (you need it in Step 5):

```bash
kubectl config view --minify -o jsonpath='{.clusters[0].name}'; echo
```

> The `exec` block is the default OCI token mechanism. You will leave it in place and add a **second** user beside it.

---

## Step 2 – Create a Namespace and Service Account

```bash
kubectl create namespace sa-demo
kubectl create serviceaccount sa-demo-user -n sa-demo
kubectl get serviceaccount sa-demo-user -n sa-demo
```

---

## Step 3 – Grant Permissions with RBAC

Allow the service account to manage ordinary workloads, but only in `sa-demo`:

```bash
kubectl create rolebinding sa-demo-edit \
  --clusterrole=edit \
  --serviceaccount=sa-demo:sa-demo-user \
  -n sa-demo
```

Check what the account can do, without using its token yet:

```bash
kubectl auth can-i create deployments -n sa-demo \
  --as=system:serviceaccount:sa-demo:sa-demo-user
kubectl auth can-i get pods -n kube-system \
  --as=system:serviceaccount:sa-demo:sa-demo-user
```

Expected: `yes` for the first command and `no` for the second.

---

## Step 4 – Generate the Token

```bash
TOKEN=$(kubectl create token sa-demo-user -n sa-demo --duration=2h)
echo "${#TOKEN} characters"
```

The token expires after the requested duration **(verify the maximum your cluster allows)**. Repeat the command to get a new one.

> **Important:** Avoid printing the token with `echo $TOKEN` in shared terminals or recorded sessions. The commands above print only its length.

---

## Step 5 – Add the Token to the Kubeconfig

You can write into your current kubeconfig, or (recommended) a separate file, so the original stays untouched.

### 5.1 Make a separate kubeconfig file (recommended)

```bash
cp ~/.kube/config ~/.kube/config-sa-demo
chmod 600 ~/.kube/config-sa-demo
export KUBECONFIG=~/.kube/config-sa-demo
```

All commands below now read and write `config-sa-demo`. To go back to your normal file later, run `unset KUBECONFIG`.

### 5.2 Add the user, the context, and select it

```bash
CLUSTER_NAME=$(kubectl config view --minify -o jsonpath='{.clusters[0].name}')

kubectl config set-credentials sa-demo-user --token="$TOKEN"

kubectl config set-context sa-demo-context \
  --cluster="$CLUSTER_NAME" \
  --user=sa-demo-user \
  --namespace=sa-demo

kubectl config use-context sa-demo-context
```

What each command does:

| Command | Effect |
|---|---|
| `set-credentials` | Adds a user entry that holds the bearer token |
| `set-context` | Joins the cluster, the new user, and a default namespace |
| `use-context` | Makes that context the active one |

### 5.3 Inspect the result

```bash
kubectl config get-contexts
kubectl config view --minify
```

The `users` section of the new context shows `token: REDACTED`. The file itself contains the real token, so protect it:

```bash
ls -l ~/.kube/config-sa-demo
```

Expected permissions: `-rw-------`.

---

## Step 6 – Verify Access and Its Limits

Run these with the new context active.

```bash
kubectl auth whoami
```

Expected: user `system:serviceaccount:sa-demo:sa-demo-user` **(the `whoami` command needs Kubernetes 1.27 or later; verify)**. If it is unavailable, use `kubectl auth can-i --list`.

**Things the account should be able to do (in `sa-demo`):**

```bash
kubectl create deployment web --image=nginx --replicas=2
kubectl get pods
kubectl scale deployment web --replicas=3
```

**Things it should not be able to do:**

```bash
kubectl get pods -n kube-system
kubectl get nodes
kubectl get pods -A
```

Expected errors look like this:

```
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:sa-demo:sa-demo-user" cannot list resource "pods" in API group "" in the namespace "kube-system"
```

Summary of the check:

| Command | Expected |
|---|---|
| `kubectl get pods` (default namespace `sa-demo`) | Works |
| `kubectl create deployment web ...` | Works |
| `kubectl get pods -n kube-system` | Forbidden |
| `kubectl get nodes` | Forbidden |

---

## Step 7 – Switch Between Identities

Return to your admin identity:

```bash
unset KUBECONFIG
kubectl config current-context
kubectl get nodes
```

Use the service account again at any time:

```bash
KUBECONFIG=~/.kube/config-sa-demo kubectl get pods
```

You can also merge several files in one session **(optional)**:

```bash
export KUBECONFIG=~/.kube/config:~/.kube/config-sa-demo
kubectl config get-contexts
kubectl config use-context sa-demo-context
```

---

## Step 8 – Use the Kubeconfig from Another Machine (Optional)

1. Copy `config-sa-demo` to the other machine over a secure channel, for example `scp`. Set permissions to `600` there.
2. Remove any other users and clusters you do not need to share, so the file carries only what is required.
3. Make sure the other machine can reach the cluster's API endpoint. For a **private** endpoint it needs network access to the VCN, for example through a VPN, FastConnect, or a bastion or jump host.
4. Run:

```bash
KUBECONFIG=~/config-sa-demo kubectl get pods
```

The other machine does **not** need the OCI CLI, because the service account token is used instead of the `exec` plugin.

---

## Step 9 – Rotate and Revoke Access

| Goal | Action |
|---|---|
| Let a short-lived token lapse | Do nothing; it expires after its duration |
| Replace a short-lived token | Run `kubectl create token ...` again and repeat `set-credentials` |
| Remove all access for the account | Delete the service account and its bindings (see Clean-up) |

Replace the token in the kubeconfig:

```bash
unset KUBECONFIG
NEW_TOKEN=$(kubectl create token sa-demo-user -n sa-demo --duration=2h)
KUBECONFIG=~/.kube/config-sa-demo kubectl config set-credentials sa-demo-user --token="$NEW_TOKEN"
```

Confirm revocation works. Delete the service account, then try the old kubeconfig:

```bash
kubectl delete serviceaccount sa-demo-user -n sa-demo
KUBECONFIG=~/.kube/config-sa-demo kubectl get pods
```

Expected: `error: You must be logged in to the server (Unauthorized)` **(verify; a deleted account's token is rejected)**.

---

## Validation Checklist

- [ ] Existing kubeconfig inspected; cluster name and server noted
- [ ] Service account created in `sa-demo`
- [ ] RBAC binding limited to the `sa-demo` namespace
- [ ] Token generated without printing it
- [ ] Token added as a new user and context in a separate kubeconfig file
- [ ] Allowed actions work and forbidden actions are denied
- [ ] Switched back to the admin identity
- [ ] Access revoked and the service account deleted

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Unauthorized` on every command | Expired or wrong token, or the account was deleted | Create a new token and update it with `set-credentials` |
| `Forbidden` on actions that should work | Missing or wrong RoleBinding | `kubectl get rolebinding -n sa-demo`; check the service account name and namespace in the binding |
| `Unable to connect to the server` | Cluster API not reachable from this machine | Check the endpoint type, security rules, VPN, or jump host |
| Certificate or x509 error | Cluster certificate data missing or edited | Re-copy the original kubeconfig and repeat Step 5 |
| `cannot create resource "serviceaccounts/token"` | Your admin identity lacks permission | Use an administrator identity to create the token |
| `kubectl create token` not found | Old `kubectl` or old cluster | Update `kubectl` and check the cluster version |
| New context still uses old identity | Wrong `KUBECONFIG` or context | Run `echo $KUBECONFIG` and `kubectl config current-context` |

---

## Clean-up

```bash
unset KUBECONFIG
kubectl delete namespace sa-demo --ignore-not-found
rm -f ~/.kube/config-sa-demo
unset TOKEN NEW_TOKEN
```

Deleting the namespace removes the service account, its role binding, and the sample deployment.

---

## Review Questions

1. How does the default OKE kubeconfig authenticate, and how does a service account token differ?
2. Why is a short-lived token safer than one that never expires?
3. Why is `edit` in a single namespace better than a cluster-wide role for most automation?
4. Why does the new kubeconfig work on a machine without the OCI CLI?
5. How do you revoke a service account's access completely?
6. Why should the kubeconfig file have `600` permissions?
