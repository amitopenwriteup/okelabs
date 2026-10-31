# Demo: Deploy Oracle MySQL DB System Service from Kubernetes

Create an OCI MySQL DB System from a Kubernetes custom resource using the OCI Service Operator for Kubernetes (OSOK) on an OKE cluster in `ap-hyderabad-1`.

**Bundle:** `ghcr.io/oracle/oci-service-operator-mysql-bundle:v2.1.0-alpha`
**Operator namespace:** `oci-service-operator-mysql-system`
**Resource:** `DbSystem` (`mysql.oracle.com/v1beta1`). It is the only resource this package exposes.
**Status:** Preview. Use a test environment.

> **Cost warning:** a MySQL DB System is a billable OCI resource, and provisioning takes a while. Delete it at the end of the demo (Lab 9).

## What you will do

| Lab | Step | Done when |
| --- | --- | --- |
| 1 | Check prerequisites | Cluster, OLM, network and IAM are ready |
| 2 | Install the MySQL bundle | CSV phase is `Succeeded` |
| 3 | Create the OCI credentials Secret | Operator logs show no auth errors |
| 4 | Create the admin Secret | Secret has `username` and `password` keys |
| 5 | Collect values and write the DbSystem YAML | Compartment, subnet and shape are known |
| 6 | Apply and watch | `lifecycleState` reaches `ACTIVE` |
| 7 | Connect to the database | `mysql` client connects |
| 8 | Verify in OCI | DB System is listed in the Console or CLI |
| 9 | Clean up | DB System deleted |

---

## Lab 1: Prerequisites

From earlier labs you should already have:

- An OKE cluster with a `Ready` node, and `kubectl` working from your VM or Cloud Shell
- `operator-sdk` installed (`operator-sdk version`)
- OLM installed (`kubectl get pods -n olm` all `Running`)

Specific to MySQL:

**1. Networking.** MySQL DB Systems need a **private subnet**, and public subnets are rejected. Use a subnet in the same VCN as your OKE cluster where possible. Oracle's MySQL networking setup guide covers the security list or NSG rules for port 3306.

**2. IAM policies.** The principal OSOK authenticates as needs these (taken from the OSOK MySQL guide):

For a user principal (the API-key method used in this guide):

```text
Allow group <OSOK_GROUP> to {SUBNET_READ, SUBNET_ATTACH, SUBNET_DETACH, VCN_READ, COMPARTMENT_INSPECT} in compartment <compartment_name>
Allow group <OSOK_GROUP> to manage mysql-family in compartment <compartment_name>
Allow group <OSOK_GROUP> to use tag-namespaces in tenancy
```

For an instance principal, use `dynamic-group <OSOK_DYNAMIC_GROUP>` in place of `group <OSOK_GROUP>`.

Quick check:

```bash
kubectl get nodes
operator-sdk version
kubectl get pods -n olm
```

---

## Lab 2: Install the MySQL bundle

Create the operator namespace first (if it already exists, `kubectl` says so and you can continue), then install the bundle into it:

```bash
kubectl create namespace oci-service-operator-mysql-system

operator-sdk run bundle ghcr.io/oracle/oci-service-operator-mysql-bundle:v2.1.0-alpha \
  --namespace oci-service-operator-mysql-system \
  --timeout 5m
```

The `--namespace` flag makes operator-sdk install the bundle, its catalog and the operator into `oci-service-operator-mysql-system`. Without the flag, operator-sdk uses the namespace of your current `kubectl` context, which is usually `default`, so always pass it explicitly. The image path follows the same pattern that worked for the Object Storage bundle (`ghcr.io/oracle/...`). If the pull fails, try `v2.0.0-alpha`, or confirm the exact tag on the package page: <https://oracle.github.io/oci-service-operator/reference/packages/mysql/>.

The `WARN ... unknown type: application/vnd.oci.empty.v1+json` lines and the long `log.SetLogger(...) was never called` stack trace are harmless. They are not failures unless a `FATA` line follows.

Verify:

```bash
kubectl get csv -A
kubectl get pods -n oci-service-operator-mysql-system
kubectl get crds | grep -i mysql
kubectl explain dbsystem.spec
```

Done when:

- the CSV `oci-service-operator-mysql.v2.1.0-alpha` is `Succeeded`
- a controller pod in `oci-service-operator-mysql-system` is `Running`
- a CRD such as `dbsystems.mysql.oracle.com` is listed

If it fails:

| Message | Fix |
| --- | --- |
| `timed out waiting for the condition` | Delete the bundle pod and rerun: `kubectl get pods -A \| grep bundle`, then `kubectl delete pod <name> -n <namespace>` |
| `install plan is not available for the subscription` | `operator-sdk cleanup oci-service-operator-mysql -n oci-service-operator-mysql-system`, then reinstall |
| Image not found or `401 Unauthorized` | Check the path and tag |

---

## Lab 3: Create the OCI credentials Secret

Each OSOK package runs in its own namespace and reads its own `ocicredentials` Secret, so a Secret created for the Object Storage operator does not carry over. Create it in the MySQL operator namespace.

Required keys: `tenancy`, `user`, `fingerprint`, `region`, `passphrase` (mandatory, use an empty string if the key has none), `privatekey`.

```bash
# Save the key with exactly one BEGIN and one END line
nano /root/oci_api_key.pem
chmod 600 /root/oci_api_key.pem

export NS=oci-service-operator-mysql-system

kubectl -n "$NS" create secret generic ocicredentials \
  --from-literal=tenancy='<TENANCY_OCID>' \
  --from-literal=user='<USER_OCID>' \
  --from-literal=fingerprint='<API_KEY_FINGERPRINT>' \
  --from-literal=region='ap-hyderabad-1' \
  --from-literal=passphrase='<KEY_PASSPHRASE>' \
  --from-file=privatekey=/root/oci_api_key.pem

kubectl -n "$NS" describe secret ocicredentials
```

Restart the controller so it reads the Secret, then check the logs:

```bash
kubectl -n "$NS" rollout restart deployment
kubectl -n "$NS" get pods
DEP=$(kubectl -n "$NS" get deploy -o jsonpath='{.items[0].metadata.name}')
kubectl -n "$NS" logs deploy/$DEP --tail=50
```

There should be no `401` or authentication errors.

---

## Lab 4: Create the admin credentials Secret

The DB System's admin username and password are read from a Kubernetes Secret in the **same namespace as the DbSystem resource** (not the operator namespace). The Secret needs the keys `username` and `password`.

This demo puts the DbSystem in the `default` namespace:

```bash
kubectl create secret generic admin-secret \
  --namespace default \
  --from-literal=username='admin' \
  --from-literal=password='<STRONG_PASSWORD>'
```

Use a strong password that meets MySQL DB System password requirements, and never put it in the DbSystem YAML itself. The resource refers to the Secret by name only.

---

## Lab 5: Collect values and write the DbSystem YAML

Collect three required values:

```bash
# 1. Compartment OCID for the demo
#    Console: Identity -> Compartments

# 2. Private subnet OCID (same VCN as OKE if possible)
oci network subnet list --compartment-id <COMPARTMENT_OCID> --region ap-hyderabad-1 \
  --query 'data[].{name:"display-name",id:id,private:"prohibit-public-ip-on-vnic"}' --output table

# 3. A valid MySQL shape for your tenancy
oci mysql shape list --compartment-id <COMPARTMENT_OCID> --region ap-hyderabad-1 \
  --query 'data[].name'
```

Pick a subnet where `private` is `true`. Choose a shape from the list. The shape in the official sample, `MySQL.2`, is only a placeholder and may not be available in your tenancy.

Create `demo-mysql.yaml`:

```yaml
apiVersion: mysql.oracle.com/v1beta1
kind: DbSystem
metadata:
  name: osok-demo-mysql
  namespace: default
spec:
  compartmentId: <COMPARTMENT_OCID>
  shapeName: <SHAPE_NAME>
  subnetId: <PRIVATE_SUBNET_OCID>
  adminUsername:
    secret:
      secretName: admin-secret
  adminPassword:
    secret:
      secretName: admin-secret
```

Required fields are `compartmentId`, `shapeName` and `subnetId`. These optional fields are commonly useful:

| Field | Purpose |
| --- | --- |
| `displayName`, `description` | Name and description shown in OCI |
| `dataStorageSizeInGBs` | Initial data volume size |
| `availabilityDomain`, `faultDomain` | Placement of the primary endpoint |
| `isHighlyAvailable` | Create a highly available DB System with three instances |
| `mysqlVersion` | MySQL version |
| `port`, `portX` | MySQL and X Plugin ports |
| `nsgIds` | Network Security Groups for the VNIC |
| `deletionPolicy`, `backupPolicy` | Delete and backup behavior (see the API reference) |

Confirm field names against your installed CRD before applying:

```bash
kubectl explain dbsystem.spec
```

Full field reference: <https://oracle.github.io/oci-service-operator/reference/api/mysql/v1beta1/#kind-dbsystem>

---

## Lab 6: Apply and watch

```bash
kubectl apply -f demo-mysql.yaml
kubectl get dbsystem -n default -w
kubectl describe dbsystem osok-demo-mysql -n default
kubectl get events -n default --sort-by=.lastTimestamp
```

Creating a DB System takes several minutes. Watch the `status` for:

| Status field | Meaning |
| --- | --- |
| `lifecycleState` | Current state of the DB System, which should end as `ACTIVE` |
| `lifecycleDetails` | Extra detail about the state |
| `id` | The DB System OCID |
| `ipAddress`, `port`, `portX`, `endpoints` | Where to connect |

Check the operator logs if it stays in an error state:

```bash
NS=oci-service-operator-mysql-system
DEP=$(kubectl -n $NS get deploy -o jsonpath='{.items[0].metadata.name}')
kubectl -n $NS logs deploy/$DEP --tail=100
```

---

## Lab 7: Connect to the database

When the DbSystem reaches `Active`, OSOK creates and manages a Secret with the **same name as the DbSystem** (`osok-demo-mysql`, in the DbSystem's namespace). It holds the observed endpoint data: `InternalFQDN`, `MySQLPort`, `MySQLXProtocolPort`, `PrivateIPAddress`, `AvailabilityDomain`, `FaultDomain` and `Endpoints`. Check its exact contents on your cluster:

```bash
kubectl get secret osok-demo-mysql -n default
kubectl describe secret osok-demo-mysql -n default
```

Read the host and port (this prints secret values, so do it in a private terminal):

```bash
HOST=$(kubectl get secret osok-demo-mysql -n default -o jsonpath='{.data.InternalFQDN}' | base64 -d)
PORT=$(kubectl get secret osok-demo-mysql -n default -o jsonpath='{.data.MySQLPort}' | base64 -d)
echo "$HOST:$PORT"
```

Connect from a temporary client pod in the cluster. The DB System is on a private subnet, so connect from something inside the VCN:

```bash
kubectl run mysql-client --rm -it --restart=Never --image=mysql:8 -n default -- \
  mysql -h "$HOST" -P "$PORT" -u admin -p
```

Enter the admin password from Lab 4, then run:

```sql
SELECT VERSION();
SHOW DATABASES;
```

This client-pod method is the standard approach, but the OSOK guide does not document connection steps. If the connection times out, check the subnet security list or NSG for port 3306 from the cluster's subnet.

---

## Lab 8: Verify in OCI

Check through Kubernetes first:

```bash
kubectl get dbsystem osok-demo-mysql -n default -o yaml
```

Then confirm in OCI:

```bash
oci mysql db-system list \
  --compartment-id <COMPARTMENT_OCID> \
  --region ap-hyderabad-1 \
  --query 'data[].{name:"display-name",state:"lifecycle-state",id:id}' --output table

oci mysql db-system get --db-system-id <DB_SYSTEM_OCID> --region ap-hyderabad-1
```

Console: **Databases -> MySQL -> DB Systems**, in the demo compartment.

What to show in the demo:

1. The `DbSystem` resource and its status in Kubernetes.
2. The operator logs.
3. The DB System in the OCI Console.
4. A successful `mysql` connection from the client pod.

---

## Lab 9: Clean up

**Delete the DB System to stop charges.** The OSOK guide does not document delete behavior. The spec has a `deletionPolicy` field, so read its definition before deleting anything important:

<https://oracle.github.io/oci-service-operator/reference/api/mysql/v1beta1/#kind-dbsystem>

```bash
kubectl delete dbsystem osok-demo-mysql -n default
kubectl get dbsystem -n default -w
```

Then confirm in OCI that it is gone or deleting:

```bash
oci mysql db-system list --compartment-id <COMPARTMENT_OCID> --region ap-hyderabad-1 \
  --query 'data[].{name:"display-name",state:"lifecycle-state"}' --output table
```

If `kubectl delete` hangs because the controller cannot reconcile the delete, check the operator logs first. As a last resort you can remove the finalizer, but do that only after confirming the DB System was deleted in OCI, or it may be left running and billing:

```bash
kubectl patch dbsystem osok-demo-mysql -n default -p '{"metadata":{"finalizers":[]}}' --type=merge
```

Remove the remaining items:

```bash
kubectl delete secret admin-secret -n default
operator-sdk cleanup oci-service-operator-mysql -n oci-service-operator-mysql-system
kubectl delete secret ocicredentials -n oci-service-operator-mysql-system
```

Finally, delete the demo API key from the OCI user (Identity -> Users -> API keys) if it was exposed, and remove local copies of the key.

---

## Troubleshooting

| Symptom | Likely cause | Check |
| --- | --- | --- |
| Controller `CrashLoopBackOff` | Missing or wrong `ocicredentials` Secret | `kubectl logs`, `describe secret`, namespace must be `oci-service-operator-mysql-system` |
| `401` or `NotAuthenticated` | Wrong fingerprint, key or passphrase | Compare with the API key in the Console |
| `403` or `NotAuthorized` | Missing policy | Re-check the policies in Lab 1 |
| Error about the subnet | Public subnet, or wrong OCID | Use a private subnet in the right compartment |
| Shape not found or not supported | Placeholder shape such as `MySQL.2` | `oci mysql shape list` and use a listed name |
| Secret reference error | `admin-secret` missing, wrong namespace, or wrong keys | Needs `username` and `password` keys in the DbSystem's namespace |
| Unknown-field error on apply | Field name differs from the CRD | `kubectl explain dbsystem.spec` |
| Stuck in `CREATING` | DB System still provisioning, or an OCI-side error | `kubectl describe dbsystem`, operator logs, OCI work requests |
| Cannot connect | Port 3306 blocked, or client outside the VCN | Security list or NSG rules, test from a pod in the cluster |
| CSV not `Succeeded` | Install did not finish | `kubectl describe csv`, delete the bundle pod, rerun |

---

## Sources

- [OSOK MySQL DbSystem guide](https://oracle.github.io/oci-service-operator/mysql/)
- [MySQL package page](https://oracle.github.io/oci-service-operator/reference/packages/mysql/)
- [DbSystem API reference](https://oracle.github.io/oci-service-operator/reference/api/mysql/v1beta1/#kind-dbsystem)
- [DbSystem sample manifest](https://oracle.github.io/oci-service-operator/reference/samples/mysql/v1beta1/dbsystem/)
- [OSOK repository MySQL guide](https://github.com/oracle/oci-service-operator/blob/main/docs/mysql.md)
- [OSOK installation guide](https://github.com/oracle/oci-service-operator/blob/main/docs/installation.md)
