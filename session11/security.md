# Lab: OKE Image Security: Scanning, Signing and Enforcing Signed Images

**Duration:** ~75–90 minutes
**Platform:** Oracle Container Engine for Kubernetes (OKE) and Oracle Cloud Infrastructure Registry (OCIR), existing cluster

---

## Lab Objectives

1. Scan a container image in OCIR for vulnerabilities with the OCI **Vulnerability Scanning Service**.
2. **Sign** a container image with a key in OCI Vault and **verify** the signature.
3. Understand how OKE **enforces the use of signed images** from OCIR.
4. Enable image verification on a cluster and prove that signed images run while unsigned images are blocked.

## Concepts

| Capability | Service | What it protects against |
|---|---|---|
| **Vulnerability scanning** | OCI Vulnerability Scanning Service (VSS) | Known CVEs in OS packages and libraries inside images |
| **Image signing** | OCI Vault (asymmetric key) + OCIR | Tampered or unapproved images (provenance and integrity) |
| **Signature enforcement** | OKE image verification policy | Running images that were never signed by your trusted key |

```
Build → Push to OCIR → Scan (CVEs) → Sign with Vault key → Deploy to OKE
                                                              │
                          OKE verifies signature against the trusted key
                          before the pod is admitted
```

**Important points:**

- Scanning tells you what is *vulnerable*; signing tells you what is *approved*. Use both.
- A signature is bound to the image **digest** (the immutable `sha256:...`), not to a mutable tag.
- Image verification policies on OKE apply to **Enhanced** clusters. Check your cluster type first.

## Prerequisites

- An existing OKE **Enhanced** cluster and working `kubectl` access as an administrator.
- OCI CLI configured, plus Docker (or Podman) on your workstation.
- Your **tenancy namespace** (Console → Tenancy details, or `oci os ns get`).
- An **auth token** for your user (Console → My profile → Auth tokens → Generate token) for OCIR login.
- Permission to create OCIR repositories, Vault keys, scan recipes and targets.

Set variables:

```bash
export REGION_KEY="bom"                       # OCIR region key, e.g. bom (Mumbai), iad (Ashburn), fra (Frankfurt)
export TENANCY_NS="$(oci os ns get --query 'data' --raw-output)"
export OCI_USER="<your-oci-username>"         # for federated users: oracleidentitycloudservice/<username>
export COMPARTMENT_OCID="ocid1.compartment.oc1..xxxx"
export CLUSTER_OCID="ocid1.cluster.oc1.<region>.xxxx"
export REPO="secure-demo/web"                 # repository path inside OCIR
export REGISTRY="${REGION_KEY}.ocir.io/${TENANCY_NS}"

kubectl get nodes
oci ce cluster get --cluster-id $CLUSTER_OCID --query 'data.type'    # expect "ENHANCED_CLUSTER"
```

---

## Part 0: Prepare a Demo Image in OCIR

### Step 0.1: Log In to OCIR

```bash
docker login ${REGION_KEY}.ocir.io -u "${TENANCY_NS}/${OCI_USER}"
# Password: paste your auth token
```

### Step 0.2: Build and Push Two Images

An older base image is useful because it will typically show vulnerabilities in the scan.

```bash
mkdir -p ~/image-demo && cd ~/image-demo
cat <<'EOF' > Dockerfile
FROM nginx:1.21
RUN echo "image security lab" > /usr/share/nginx/html/index.html
EOF

docker build -t ${REGISTRY}/${REPO}:v1 .
docker push ${REGISTRY}/${REPO}:v1
```

Create a second image that you will intentionally **leave unsigned**:

```bash
docker tag ${REGISTRY}/${REPO}:v1 ${REGISTRY}/${REPO}:unsigned
docker push ${REGISTRY}/${REPO}:unsigned
```

> If the repository doesn't exist yet, OCIR creates it on first push (as private by default in the tenancy root compartment). To use a specific compartment, create the repository first in the Console.

### Step 0.3: Collect Image Details

```bash
oci artifacts container image list --compartment-id $COMPARTMENT_OCID \
  --repository-name $REPO \
  --query 'data.items[].{id:id,digest:digest,version:version}' --output table
```

Record the **image OCID** and **digest** of the `v1` image:

```bash
export IMAGE_OCID="ocid1.containerimage.oc1.<region>.xxxx"
export IMAGE_DIGEST="sha256:xxxxxxxx"
```

> Both tags currently point to the same digest, since the same image was pushed twice. To make the unsigned test valid, rebuild the second image with a small change, as shown below, so that it has a different digest.

```bash
cat <<'EOF' > Dockerfile.unsigned
FROM nginx:1.21
RUN echo "this image is NOT signed" > /usr/share/nginx/html/index.html
EOF
docker build -f Dockerfile.unsigned -t ${REGISTRY}/${REPO}:unsigned .
docker push ${REGISTRY}/${REPO}:unsigned
```

Re-run the `image list` command, and record the **unsigned** image's digest:

```bash
export UNSIGNED_DIGEST="sha256:yyyyyyyy"
```

---

## Part 1: Scan a Container Image for Vulnerabilities

### Step 1.1: Create IAM Policies for the Scanning Service

The scanning service needs permission to read your repositories and compartments:

```
Allow service vulnerability-scanning-service to read repos in tenancy
Allow service vulnerability-scanning-service to read compartments in tenancy
```

Your user (or its group) also needs access to manage scan resources, for example:

```
Allow group <iam-group> to manage vss-family in compartment <compartment-name>
Allow group <iam-group> to read repos in compartment <compartment-name>
```

### Step 1.2: Create a Container Scan Recipe

**Console:** Identity & Security → **Scanning** → **Container Scan Recipes** → **Create container scan recipe**. Choose a name (for example `oke-demo-recipe`) and a scan level.

**CLI (alternative):**

```bash
oci vulnerability-scanning container-scan-recipe create \
  --compartment-id $COMPARTMENT_OCID \
  --display-name oke-demo-recipe \
  --scan-settings '{"scanLevel":"STANDARD"}'
```

### Step 1.3: Create a Container Scan Target

The target ties the recipe to the repository to be scanned.

**Console:** Scanning → **Targets** → **Create target** → choose **Container** type, select the recipe, the compartment and the repository (`secure-demo/web`).

**CLI (alternative):**

```bash
export RECIPE_OCID="ocid1.containerscanrecipe.oc1..xxxx"

oci vulnerability-scanning container-scan-target create \
  --compartment-id $COMPARTMENT_OCID \
  --display-name oke-demo-target \
  --container-scan-recipe-id $RECIPE_OCID \
  --target-registry '{"compartmentId":"'$COMPARTMENT_OCID'","type":"OCIR","repositories":["'$REPO'"]}'
```

### Step 1.4: Wait for the Scan and View Results

Scans start after the target is created and when new images are pushed to the repository. This can take several minutes.

**Console:** Scanning → **Reports** → **Container image scans**. Open the scan for your image to see the severity summary and CVE list.

**CLI:**

```bash
oci vulnerability-scanning container-scan-result list --compartment-id $COMPARTMENT_OCID \
  --query 'data.items[].{image:"image",highest:"highest-problem-severity",total:"problem-count"}' --output table
```

### Step 1.5: Analyze the Report

For each finding, note:

- **Severity** (Critical, High, Medium, Low).
- **CVE ID** and the affected package and version.
- Whether a **fixed version** is available.

### Step 1.6: Remediate and Rescan

Update the base image and rebuild:

```bash
cat <<'EOF' > Dockerfile
FROM nginx:stable
RUN echo "image security lab - patched" > /usr/share/nginx/html/index.html
EOF
docker build -t ${REGISTRY}/${REPO}:v2 .
docker push ${REGISTRY}/${REPO}:v2
```

Wait for the new scan and compare the vulnerability counts of `v1` and `v2`.

> A typical policy is to block promotion (or signing) of any image with Critical or High findings.

---

## Part 2: Sign and Verify a Container Image

### Step 2.1: Create a Vault and a Signing Key

**Console:** Identity & Security → **Vault** → create a vault (or use an existing one) → **Keys** → **Create key**:

| Setting | Value |
|---|---|
| Protection mode | HSM (or Software) |
| Key shape: Algorithm | **RSA** |
| Key shape: Length | 2048 or 4096 bits |
| Name | `oke-image-signing-key` |

> The key must be an **asymmetric RSA** key (a symmetric AES key can't sign images). Note the key OCID and the **key version OCID**.

```bash
export KEY_OCID="ocid1.key.oc1.<region>.xxxx"
export KEY_VERSION_OCID="ocid1.keyversion.oc1.<region>.xxxx"
```

You can also list them:

```bash
oci kms management key list --compartment-id $COMPARTMENT_OCID \
  --endpoint https://<vault-management-endpoint> --output table
```

### Step 2.2: Grant Permission to Use the Key

Your user (or group) needs to use the key to sign:

```
Allow group <iam-group> to use keys in compartment <compartment-name>
Allow group <iam-group> to manage repos in compartment <compartment-name>
```

### Step 2.3: Sign the Image

Sign the image by its **OCID** (which is bound to its digest):

```bash
oci artifacts container image-signature sign-upload \
  --compartment-id $COMPARTMENT_OCID \
  --kms-key-id $KEY_OCID \
  --kms-key-version-id $KEY_VERSION_OCID \
  --signing-algorithm SHA_256_RSA_PKCS_PSS \
  --image-id $IMAGE_OCID \
  --description "Approved release v1"
```

**Expected:** JSON output describing the new image signature (ID, key, algorithm, image ID).

### Step 2.4: Verify the Signature Exists

```bash
oci artifacts container image-signature list \
  --compartment-id $COMPARTMENT_OCID \
  --image-id $IMAGE_OCID \
  --output table

oci artifacts container image-signature get --image-signature-id <signature-ocid>
```

Check that:

- The signature's **key ID** and **key version** match your signing key.
- The **image ID** is the image you intended to sign.

You can also see the signature in the Console under **Developer Services → Container Registry → repository → image → Signatures**.

### Step 2.5: Confirm the Unsigned Image Has No Signature

```bash
export UNSIGNED_IMAGE_OCID="ocid1.containerimage.oc1.<region>.yyyy"
oci artifacts container image-signature list \
  --compartment-id $COMPARTMENT_OCID --image-id $UNSIGNED_IMAGE_OCID
```

**Expected:** an empty list.

### Step 2.6: What Happens If the Image Changes?

A signature belongs to one digest. If you rebuild and push `v1` again with different contents, the new digest has **no signature**, so it will fail verification later. This is how signing detects tampering.

---

## Part 3: Enforcing the Use of Signed Images from OCIR

### Overview

An OKE **image verification policy** makes the cluster check, when a pod is created, that each image has a valid signature made by one of the **trusted keys** you list. Pods whose images are unsigned (or signed by another key) are **rejected**.

Prerequisites for enforcement:

- Enhanced cluster.
- Images stored in OCIR.
- A signing key in OCI Vault, with the signatures uploaded as in Part 2.
- An IAM policy that lets the **cluster** use the key to verify.

### Step 3.1: Allow the Cluster to Use the Key

Create this policy (replace OCIDs and names):

```
Allow any-user to use keys in compartment <compartment-name> where all {request.principal.type = 'cluster', request.principal.id = '<CLUSTER_OCID>'}
```

Without it, the cluster can't read the key and verification fails.

### Step 3.2: Create the Image Pull Secret

So pods can pull from the private OCIR repository:

```bash
kubectl create namespace secure-demo

kubectl -n secure-demo create secret docker-registry ocirsecret \
  --docker-server=${REGION_KEY}.ocir.io \
  --docker-username="${TENANCY_NS}/${OCI_USER}" \
  --docker-password='<AUTH_TOKEN>' \
  --docker-email='you@example.com'
```

### Step 3.3: Deploy Both Images *Before* Enforcement (Baseline)

Confirm that both images run when no policy exists. Using the **digest** reference is the most reliable way to make sure the signed image is the one that is checked:

```bash
cat <<EOF > baseline.yaml
apiVersion: v1
kind: Pod
metadata:
  name: signed-pod
  namespace: secure-demo
spec:
  imagePullSecrets:
    - name: ocirsecret
  containers:
    - name: web
      image: ${REGISTRY}/${REPO}@${IMAGE_DIGEST}
---
apiVersion: v1
kind: Pod
metadata:
  name: unsigned-pod
  namespace: secure-demo
spec:
  imagePullSecrets:
    - name: ocirsecret
  containers:
    - name: web
      image: ${REGISTRY}/${REPO}@${UNSIGNED_DIGEST}
EOF
kubectl apply -f baseline.yaml
kubectl -n secure-demo get pods
```

**Expected:** both pods start (`Running`). Delete them before the next step:

```bash
kubectl delete -f baseline.yaml
```

### Step 3.4: Enable Image Verification on the Cluster

**Console:** Cluster details → **Image verification** (or **Edit cluster**) → enable **Image verification** → **Add key** → choose the vault, key and compartment → save.

**CLI (alternative):**

```bash
oci ce cluster update --cluster-id $CLUSTER_OCID \
  --image-policy-config '{"isPolicyEnabled": true, "keyDetails": [{"kmsKeyId": "'$KEY_OCID'"}]}'
```

Track the update:

```bash
oci ce cluster get --cluster-id $CLUSTER_OCID \
  --query 'data.{state:"lifecycle-state",policy:"image-policy-config"}'
```

Wait for the cluster to return to `ACTIVE`.

> **Rollout tip:** Enable verification in a non-production cluster first. System add-on images in `kube-system` and other namespaces must also meet the policy, depending on how the policy and namespace exemptions are set up. Confirm the current behavior and exemption options in the OKE documentation before enabling it on a shared cluster.

### Step 3.5: Test with the Signed Image (Should Succeed)

```bash
cat <<EOF > signed.yaml
apiVersion: v1
kind: Pod
metadata:
  name: signed-pod
  namespace: secure-demo
spec:
  imagePullSecrets:
    - name: ocirsecret
  containers:
    - name: web
      image: ${REGISTRY}/${REPO}@${IMAGE_DIGEST}
EOF
kubectl apply -f signed.yaml
kubectl -n secure-demo get pod signed-pod
```

**Expected:** the pod is created and reaches `Running`.

### Step 3.6: Test with the Unsigned Image (Should Be Blocked)

```bash
cat <<EOF > unsigned.yaml
apiVersion: v1
kind: Pod
metadata:
  name: unsigned-pod
  namespace: secure-demo
spec:
  imagePullSecrets:
    - name: ocirsecret
  containers:
    - name: web
      image: ${REGISTRY}/${REPO}@${UNSIGNED_DIGEST}
EOF
kubectl apply -f unsigned.yaml
```

**Expected:** the request is **denied** with an admission error saying that image signature verification failed (no valid signature from a trusted key). The pod is not created:

```bash
kubectl -n secure-demo get pods
```

### Step 3.7: Sign the Second Image and Retry (Optional)

```bash
oci artifacts container image-signature sign-upload \
  --compartment-id $COMPARTMENT_OCID \
  --kms-key-id $KEY_OCID --kms-key-version-id $KEY_VERSION_OCID \
  --signing-algorithm SHA_256_RSA_PKCS_PSS \
  --image-id $UNSIGNED_IMAGE_OCID

kubectl apply -f unsigned.yaml
```

**Expected:** after signing, the same pod is now admitted.

### Step 3.8: Disable or Adjust the Policy (Cleanup / Rollback)

```bash
oci ce cluster update --cluster-id $CLUSTER_OCID \
  --image-policy-config '{"isPolicyEnabled": false}'
```

---

## Verification Checklist

- [ ] The image was pushed to OCIR and the scan report lists vulnerabilities by severity.
- [ ] A patched image (`v2`) shows fewer findings than `v1`.
- [ ] A signature exists for the signed image, with the correct key.
- [ ] The unsigned image has no signatures.
- [ ] After enabling the policy, the signed image's pod runs.
- [ ] The unsigned image's pod is rejected, and runs only after signing.

## Troubleshooting Quick Reference

| Problem | Likely Cause | Action |
|---|---|---|
| `docker login` fails (unauthorized) | Wrong username format or token | Use `<tenancy-namespace>/<username>` (add the identity-provider prefix for federated users) and a valid auth token |
| Scan never starts | Missing service policy, or wrong target | Check the two `vulnerability-scanning-service` policies and the target's repository |
| No scan results for the new image | Scan pending | Wait; confirm the image is in the target's compartment and repository |
| `sign-upload` returns authorization error | Missing key or repo permissions | Add `use keys` and `manage repos` policies |
| Signing fails on key type | Symmetric key | Create an **RSA asymmetric** key |
| Signed pod is rejected | Cluster can't use the key, or the signature is for a different digest/key | Check the cluster principal policy; confirm that the image digest has a signature from the trusted key |
| Image pull error (`ErrImagePull`) | Missing or wrong pull secret | Recreate `ocirsecret` and reference it in `imagePullSecrets` |
| System pods fail after enabling the policy | Their images aren't signed by your key | Review the exemption settings and the guidance in the OKE docs; test on a non-production cluster |
| Tag-based reference fails verification | Tag moved to a different digest | Deploy by **digest** (`image@sha256:...`) |

## Cleanup

```bash
kubectl delete namespace secure-demo

# Disable enforcement if you don't want to keep it
oci ce cluster update --cluster-id $CLUSTER_OCID --image-policy-config '{"isPolicyEnabled": false}'

# Optional: remove scan resources
oci vulnerability-scanning container-scan-target delete --container-scan-target-id <target-ocid> --force
oci vulnerability-scanning container-scan-recipe delete --container-scan-recipe-id $RECIPE_OCID --force

# Optional: delete demo images and signatures from OCIR via the Console or:
oci artifacts container image delete --image-id $IMAGE_OCID --force
oci artifacts container image delete --image-id $UNSIGNED_IMAGE_OCID --force
```

Schedule the signing key for deletion in the Vault if it was created only for this lab, and remove the IAM policies you added.

## Summary

| Topic | Key Takeaway |
|---|---|
| Vulnerability scanning | Create a recipe and a target for an OCIR repository; review CVEs, rebuild, rescan |
| Signing | Use an asymmetric RSA key in Vault; the signature is bound to the image digest |
| Enforcement | An OKE image verification policy rejects pods whose images lack a valid signature from a trusted key |
| Best practice | Scan first, sign only approved images, deploy by digest, and test enforcement on non-production clusters |
