# Lab: OSOK Object Storage Bucket on OKE (ap-hyderabad-1)

Install the OCI Service Operator for Kubernetes (OSOK) Object Storage bundle on an OKE cluster, then create an OCI Object Storage bucket from a Kubernetes custom resource.

**Bundle:** `ghcr.io/oracle/oci-service-operator-objectstorage-bundle:v2.1.0-alpha`
**Operator namespace:** `oci-service-operator-objectstorage-system`
**Scope:** the package exposes one resource, `Bucket` (`objectstorage.oracle.com/v1beta1`). Status: Preview, so use a test environment.

## Prerequisites

- An OKE cluster with at least one `Ready` worker node
- A VM or Cloud Shell with `kubectl` pointed at the cluster (`kubectl get nodes` works)
- `operator-sdk` installed (`operator-sdk version`)
- OLM installed (pods in namespace `olm` are `Running`)
- An OCI user with an API key (tenancy OCID, user OCID, fingerprint, private key, passphrase)
- An IAM policy that lets the user manage buckets in the demo compartment:

  ```text
  Allow group <osok-operator-group> to manage object-family in compartment <demo-compartment-name>
  ```

Quick check:

```bash
kubectl get nodes
operator-sdk version
kubectl get pods -n olm
```

---

## Lab 1: Install operator-sdk (skip if already done)

```bash
export ARCH=$(case $(uname -m) in x86_64) echo -n amd64 ;; aarch64) echo -n arm64 ;; *) echo -n $(uname -m) ;; esac)
export OS=$(uname | awk '{print tolower($0)}')
export OPERATOR_SDK_VERSION=v1.41.1
export OPERATOR_SDK_DL_URL=https://github.com/operator-framework/operator-sdk/releases/download/${OPERATOR_SDK_VERSION}

curl -fLO ${OPERATOR_SDK_DL_URL}/operator-sdk_${OS}_${ARCH}
curl -fLO ${OPERATOR_SDK_DL_URL}/checksums.txt
grep operator-sdk_${OS}_${ARCH} checksums.txt | sha256sum -c -
chmod +x operator-sdk_${OS}_${ARCH}
mv operator-sdk_${OS}_${ARCH} /usr/local/bin/operator-sdk
hash -r
operator-sdk version
```

Use a real version number. Pasting a placeholder such as `<vX.Y.Z>` makes bash fail, and curl would save an error page as the binary.

## Lab 2: Install OLM (skip if already done)

```bash
operator-sdk olm install --version 0.20.0
operator-sdk olm status
kubectl get pods -n olm
```

Expected: `catalog-operator`, `olm-operator`, `operatorhubio-catalog` and two `packageserver` pods are `Running`.

---

## Lab 3: Install the OSOK Object Storage bundle

```bash
operator-sdk run bundle ghcr.io/oracle/oci-service-operator-objectstorage-bundle:v2.1.0-alpha --timeout 5m
```

This pulls the bundle image, builds a catalog, and creates a CatalogSource, OperatorGroup and Subscription so OLM installs the operator. The package creates the namespace `oci-service-operator-objectstorage-system`.

**Warnings you can ignore:**

```text
WARN[0003] reference for unknown type: application/vnd.oci.empty.v1+json ...
INFO[0034] Generated a valid File-Based Catalog
[controller-runtime] log.SetLogger(...) was never called; logs will not be displayed.
Detected at: ... (long stack trace)
```

These are cosmetic. The image was pulled and the catalog was built. The trace is not a failure unless a `FATA` line follows.

**Success looks like:**

```text
INFO[...] Found ClusterServiceVersion ... phase: Succeeded
INFO[...] OLM has successfully installed "oci-service-operator-objectstorage.v2.1.0-alpha"
```

**If it fails:**

| Message | Fix |
| --- | --- |
| `timed out waiting for the condition` | Delete the bundle pod and rerun: `kubectl get pods -A \| grep bundle`, then `kubectl delete pod <name> -n <namespace>` |
| `install plan is not available for the subscription` | `operator-sdk cleanup oci-service-operator-objectstorage -n oci-service-operator-objectstorage-system`, then reinstall |
| Image not found or `401 Unauthorized` | Check the path and tag. Try `v2.0.0-alpha`. |

---

## Lab 4: Verify the installation

```bash
kubectl get csv -A
kubectl get catalogsource,subscription,installplan -A | grep -i objectstorage
kubectl get pods -n oci-service-operator-objectstorage-system
kubectl get crds | grep -i objectstorage
kubectl explain bucket.spec
```

Done when:

- CSV `oci-service-operator-objectstorage.v2.1.0-alpha` is `Succeeded`
- a controller pod is `Running` (it may report auth errors until Lab 5)
- CRD `buckets.objectstorage.oracle.com` is listed

If the CSV is not `Succeeded`:

```bash
kubectl describe csv -n oci-service-operator-objectstorage-system
kubectl get events -n oci-service-operator-objectstorage-system --sort-by=.lastTimestamp
```

---

## Lab 5: Create the credentials Secret and restart the operator

The operator reads a Secret named `ocicredentials` from its own namespace. Required keys: `tenancy`, `user`, `fingerprint`, `region`, `passphrase`, `privatekey`. The `passphrase` is mandatory (use an empty string if the key has none; an `ENCRYPTED PRIVATE KEY` header means it has one).

**Recommended: build the Secret from the key file**, which avoids YAML indentation errors.

```bash
# Save the key with exactly one BEGIN and one END line
nano /root/oci_api_key.pem
chmod 600 /root/oci_api_key.pem

export NS=oci-service-operator-objectstorage-system

kubectl -n "$NS" create secret generic ocicredentials \
  --from-literal=tenancy='<TENANCY_OCID>' \
  --from-literal=user='<USER_OCID>' \
  --from-literal=fingerprint='<API_KEY_FINGERPRINT>' \
  --from-literal=region='ap-hyderabad-1' \
  --from-literal=passphrase='<KEY_PASSPHRASE>' \
  --from-file=privatekey=/root/oci_api_key.pem

kubectl -n "$NS" describe secret ocicredentials
```

**Alternative: YAML.** Every key line must be indented 4 spaces under `privatekey: |`, with one `BEGIN` and one `END` line and no stray characters:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ocicredentials
  namespace: oci-service-operator-objectstorage-system
type: Opaque
stringData:
  tenancy: "<TENANCY_OCID>"
  user: "<USER_OCID>"
  fingerprint: "<API_KEY_FINGERPRINT>"
  region: "ap-hyderabad-1"
  passphrase: "<KEY_PASSPHRASE>"
  privatekey: |
    -----BEGIN ENCRYPTED PRIVATE KEY-----
    <key lines, each indented 4 spaces>
    -----END ENCRYPTED PRIVATE KEY-----
```

```bash
kubectl apply -f osok-oci-credentials.yaml
```

Restart the controller and check the logs:

```bash
kubectl -n "$NS" rollout restart deployment
kubectl -n "$NS" get pods
DEP=$(kubectl -n "$NS" get deploy -o jsonpath='{.items[0].metadata.name}')
kubectl -n "$NS" logs deploy/$DEP --tail=50
```

There should be no authentication or `401` errors.

---

## Lab 6: Create a Bucket from Kubernetes

Check the real field names for your installed version first. The official sample manifest is only a placeholder (`spec: {}`):

```bash
kubectl explain bucket.spec
```

Collect the values:

```bash
oci os ns get --region ap-hyderabad-1        # Object Storage namespace
# Compartment OCID: Console -> Identity -> Compartments
```

Create `demo-bucket.yaml`. The field names below are the expected ones from the OCI Object Storage API, so confirm them with `kubectl explain` and adjust if needed:

```yaml
apiVersion: objectstorage.oracle.com/v1beta1
kind: Bucket
metadata:
  name: osok-demo-bucket
  namespace: default
spec:
  compartmentId: <COMPARTMENT_OCID>
  name: osok-demo-bucket
  namespace: <OBJECT_STORAGE_NAMESPACE>
```

```bash
kubectl apply -f demo-bucket.yaml
kubectl get bucket -n default -w
kubectl describe bucket osok-demo-bucket -n default
kubectl get events -n default --sort-by=.lastTimestamp
```

An unknown-field error on apply names the field to correct.

---

## Lab 7: Verify the bucket in OCI

```bash
kubectl get bucket osok-demo-bucket -n default -o yaml

oci os bucket list \
  --compartment-id <COMPARTMENT_OCID> \
  --namespace-name <OBJECT_STORAGE_NAMESPACE> \
  --region ap-hyderabad-1

oci os bucket get \
  --bucket-name osok-demo-bucket \
  --namespace-name <OBJECT_STORAGE_NAMESPACE> \
  --region ap-hyderabad-1
```

Console: **Storage -> Buckets**, choose the demo compartment.

Show in the demo:

1. The Kubernetes `Bucket` resource and its status.
2. The operator logs from Lab 5.
3. The bucket in the OCI Console.

---

## Lab 8: Clean up

```bash
kubectl delete bucket osok-demo-bucket -n default
oci os bucket list --compartment-id <COMPARTMENT_OCID> --namespace-name <OBJECT_STORAGE_NAMESPACE> --region ap-hyderabad-1

operator-sdk cleanup oci-service-operator-objectstorage -n oci-service-operator-objectstorage-system
kubectl delete secret ocicredentials -n oci-service-operator-objectstorage-system
```

A bucket that still contains objects cannot be deleted in OCI until it is emptied. Validate delete behavior in a test environment first.

Optionally remove OLM with `operator-sdk olm uninstall`. Delete the demo API key from the OCI user (Identity -> Users -> API keys) if it was exposed, and remove local key files.

---

## Troubleshooting

| Symptom | Likely cause | Check |
| --- | --- | --- |
| Controller `CrashLoopBackOff` | Missing or wrong `ocicredentials` Secret | `kubectl logs`, `describe secret` |
| `401` or `NotAuthenticated` | Wrong fingerprint, key or passphrase | Compare with the API key in the Console |
| `403` or `NotAuthorized` | Missing IAM policy | Add the policy from Prerequisites |
| Bucket has no status | Bad fields, or operator cannot reach OCI | `kubectl describe bucket`, operator logs |
| Unknown-field error on apply | Field name differs from the template | `kubectl explain bucket.spec` |
| CSV not `Succeeded` | Install incomplete | `kubectl describe csv`, delete the bundle pod, rerun |
| Secret not picked up | Wrong namespace or name | Must be `ocicredentials` in the operator namespace |

## Sources

- <https://oracle.github.io/oci-service-operator/reference/packages/objectstorage/>
- <https://oracle.github.io/oci-service-operator/guides/objectstorage/bucket/>
- <https://github.com/oracle/oci-service-operator/blob/main/docs/installation.md>
- <https://sdk.operatorframework.io/docs/installation/>
