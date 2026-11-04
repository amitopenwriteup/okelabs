# OKE Lab: Image Security

**Topics**
1. Demo: Scan a container image for vulnerabilities
2. Demo: Sign and verify a container image
3. Concept: Enforcing the use of signed images from OCIR
4. Demo: Enforcing the use of signed images from OCIR

**Estimated time:** 60-90 minutes
**Level:** Intermediate

---

## Prerequisites

- An OCI tenancy and a user with permissions to manage OCIR, Vault, Vulnerability Scanning and OKE
- An **Enhanced** OKE cluster (image verification is not available on Basic clusters)
- `kubectl` configured to access the cluster
- The OCIR repository from the previous lab (for example `oke-demo/webapp`) that **already contains the images you pushed earlier**. No new push is needed; this lab reuses them:
  - `v1` - the image you will scan and sign
  - `v2` - an image you will leave **unsigned** for the enforcement test
- Note your repository name and compartment; replace `oke-demo/webapp` in this lab with your own repository name
- An **Auth Token** for your user (Profile > My profile > Auth tokens)

---

# Part 1 - Demo: Scan a Container Image for Vulnerabilities

OCI **Vulnerability Scanning Service (VSS)** scans images stored in OCIR and reports known CVEs.

## 1.1 Verify the repository contents

1. Open **Developer Services > Container Registry**.
2. Select your compartment and open the repository `oke-demo/webapp`.
3. Confirm both `v1` and `v2` are listed.

## 1.2 Create the IAM policy for the scanning service

Go to **Identity & Security > Policies > Create Policy** and add:

```text
allow service vulnerability-scanning-service to read repos in tenancy
allow service vulnerability-scanning-service to read compartments in tenancy
```

## 1.3 Create a Container Scan Recipe

1. Go to **Identity & Security > Scanning > Container image scan recipes**.
2. Click **Create container scan recipe**.
3. Enter:
   - Name: `oke-demo-recipe`
   - Compartment: your compartment
   - Scan level: `Standard`
   - Images to scan: `2` (scans both `v1` and `v2`; with `1` only the most recent image is scanned)
4. Click **Create**.

## 1.4 Create a Container Scan Target

1. Go to **Scanning > Container image scan targets** and click **Create target**.
2. Enter:
   - Name: `oke-demo-target`
   - Recipe: `oke-demo-recipe`
   - Registry compartment: the compartment holding the repository
   - Repository: the same repository where you pushed the images in the previous lab (for example `oke-demo/webapp`)
3. Click **Create**.

> You do not need to push the images again. The target scans what is already in the repository. If no report appears after a few minutes, push a new tag or edit and save the target to trigger a scan.

## 1.5 View the results

1. Wait a few minutes for the scan to complete.
2. Go to **Scanning > Container image scan reports**.
3. Open the report for your repository (for example `oke-demo/webapp`) and review the results for both `v1` and `v2`.
4. Review the severity summary (Critical / High / Medium / Low), CVE IDs, affected packages and fixed versions.

## 1.6 Remediate

- Rebuild the image from a patched base image, push it as a new version, and re-scan.
- Confirm the high and critical counts drop.

**Checkpoint:** You can see a vulnerability report for your OCIR image.

---

# Part 2 - Demo: Sign and Verify a Container Image

Image signing proves **who** published an image and that it has **not been modified**. The signing key is stored in **OCI Vault**.

## 2.1 Create a Vault and a signing key

1. Go to **Identity & Security > Vault** and click **Create Vault** (name: `oke-signing-vault`).
2. Open the vault and click **Create Key**:
   - Name: `image-signing-key`
   - Protection mode: HSM (or Software)
   - Key shape - Algorithm: **RSA**, Length: **2048 bits**
3. Wait for the key to become **Enabled**.

## 2.2 Sign the image

1. Go to **Developer Services > Container Registry** and open `oke-demo/webapp`.
2. Click the image **v1**.
3. Open the **Image signatures** tab and click **Create image signature**.
4. Select:
   - Vault compartment and **Vault**: `oke-signing-vault`
   - **Key**: `image-signing-key` and its key version
   - Signing algorithm: `SHA_256_RSA_PKCS_PSS`
   - Description: `oke-demo v1 signature`
5. Click **Create**.

## 2.3 Verify the signature

1. Stay on the `v1` image page and open **Image signatures**.
2. Confirm a signature is listed with the key, key version and algorithm.
3. Open `v2` and confirm it shows **no signatures**.

**Checkpoint:** `v1` is signed; `v2` is unsigned.

---

# Part 3 - Concept: Enforcing the Use of Signed Images from OCIR

Signing alone does nothing unless the cluster **refuses unsigned images**. OKE provides an **image verification policy** (admission control) for **Enhanced** clusters.

How it works:

1. You select one or more Vault keys in the cluster's **Image verification** settings.
2. When a pod is created, the OKE admission controller checks the image in OCIR.
3. The image should be referenced by **digest** (`@sha256:...`), not only by tag, for reliable verification.
4. If a valid signature from a trusted key exists, the pod is admitted; otherwise it is **rejected**.

Key points:
- Works with images stored in **OCIR**
- Multiple keys can be trusted (for example dev and release keys)
- Policy can be turned on or off per cluster
- The cluster needs IAM permission to use the key to verify signatures

---

# Part 4 - Demo: Enforcing the Use of Signed Images from OCIR

## 4.1 Allow the cluster to use the key

Go to **Identity & Security > Policies > Create Policy** and add:

```text
ALLOW any-user to use keys in compartment <compartment-name>
  where all {request.principal.type = 'cluster',
             request.principal.id = '<cluster-ocid>'}
```

## 4.2 Enable the image verification policy

1. Go to **Developer Services > Kubernetes Clusters (OKE)** and open your cluster.
2. Find **Image verification** and click **Edit**.
3. Select **Enable image verification policy**.
4. Click **Add key**, then choose the vault, key and `image-signing-key`.
5. Save and wait until the cluster is **Active**.

## 4.3 Get the image digests

1. Go to **Container Registry > oke-demo/webapp**.
2. Open `v1` and copy its **digest** (`sha256:...`) - this is the signed image.
3. Open `v2` and copy its digest - this is the unsigned image.

## 4.4 Create the OCIR pull secret

```bash
kubectl create secret docker-registry ocirsecret \
  --docker-server=<region-key>.ocir.io \
  --docker-username='<tenancy-namespace>/<username>' \
  --docker-password='<auth-token>' \
  --docker-email='you@example.com'
```

## 4.5 Test 1 - Deploy the SIGNED image (expected: success)

`signed-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: signed-app
spec:
  containers:
    - name: app
      image: <region-key>.ocir.io/<tenancy-namespace>/oke-demo/webapp@sha256:<v1-digest>
  imagePullSecrets:
    - name: ocirsecret
```

```bash
kubectl apply -f signed-pod.yaml
kubectl get pod signed-app
```

Expected: the pod reaches `Running`.

## 4.6 Test 2 - Deploy the UNSIGNED image (expected: rejected)

`unsigned-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: unsigned-app
spec:
  containers:
    - name: app
      image: <region-key>.ocir.io/<tenancy-namespace>/oke-demo/webapp@sha256:<v2-digest>
  imagePullSecrets:
    - name: ocirsecret
```

```bash
kubectl apply -f unsigned-pod.yaml
```

Expected: the request is **denied** with an error saying the image has no valid signature / failed verification.

## 4.7 Clean up

```bash
kubectl delete pod signed-app --ignore-not-found
kubectl delete secret ocirsecret
```

> Do not delete the images or repository until you have finished the scan, signing and enforcement tests, because they need `v1` and `v2`.

In the console:
1. Cluster > **Image verification** > **Edit** > clear **Enable image verification policy** > Save.
2. Delete the scan target and recipe.
3. Delete test images/repository in Container Registry if no longer needed.
4. Schedule the Vault key for deletion if no longer needed.

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| No scan report appears | Missing VSS IAM policy, or target repository/compartment is wrong |
| Image verification option is missing | Cluster is Basic; upgrade to Enhanced |
| Signed image is still rejected | Cluster policy lacks key access, wrong key trusted, or image referenced by tag |
| `ImagePullBackOff` | Missing or incorrect `imagePullSecrets` |

## Summary

| Stage | Tool | Outcome |
|---|---|---|
| Scan | OCI Vulnerability Scanning | Know the CVEs in OCIR images |
| Sign | OCI Vault + Container Registry | Image has a trusted signature |
| Verify | Image signatures tab | Confirm the signature exists |
| Enforce | OKE image verification policy | Only signed images run in the cluster |
