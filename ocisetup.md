# Workshop Lab: Installing and Configuring OCI CLI and kubectl on Oracle Linux 9

## Overview

In this lab you will install the Oracle Cloud Infrastructure (OCI) Command Line Interface on an Oracle Linux 9 machine, configure it to authenticate against your tenancy in the **India South (Hyderabad)** region, verify connectivity, and install `kubectl` so you can manage Kubernetes clusters (such as OKE) from the same machine.

**Estimated time:** 20 to 30 minutes

## Objectives

By the end of this lab you will be able to:

- Install the OCI CLI on Oracle Linux 9
- Generate an API signing key pair and register the public key in the OCI Console
- Create a working OCI CLI configuration file
- Verify the CLI can reach your tenancy
- Install `kubectl` for your CPU architecture

## Prerequisites

| Requirement | Details |
|---|---|
| Operating system | Oracle Linux 9 (x86_64 or aarch64) |
| Access | A user with `sudo` (or root) privileges |
| Network | Outbound internet access to Oracle yum repositories and `dl.k8s.io` |
| OCI account | An OCI tenancy and a user able to sign in to the Console |

Keep these values handy before you start. You will need them in Task 2:

| Value | Where to find it |
|---|---|
| **User OCID** | Console: Profile icon, **My profile** (or **User settings**), copy the OCID |
| **Tenancy OCID** | Console: Profile icon, **Tenancy: <name>**, copy the OCID |
| **Region** | India South (Hyderabad), identifier `ap-hyderabad-1` |

---

## Task 1: Install the OCI CLI

### Step 1.1: Enable the Oracle Linux developer repository

```bash
sudo dnf -y install oraclelinux-developer-release-el9
```

### Step 1.2: Install the OCI CLI package

```bash
sudo dnf -y install python39-oci-cli
```

### Step 1.3: Verify the installation

```bash
oci --version
```

**Expected result:** a version number is printed (for example `3.x.x`). If you see `command not found`, confirm the package installed successfully in Step 1.2.

---

## Task 2: Configure the OCI CLI

### Step 2.1: Start the interactive setup

```bash
oci setup config
```

### Step 2.2: Answer the prompts

The setup wizard asks for the following. Use the values you collected in the prerequisites.

| Prompt | What to enter |
|---|---|
| Location for your config | Press **Enter** to accept the default (`/root/.oci/config`, or `~/.oci/config` for a normal user) |
| User OCID | Paste your **user OCID** |
| Tenancy OCID | Paste your **tenancy OCID** |
| Region | Enter the number shown for **ap-hyderabad-1** in the region list (in this lab it is **10**; the number can differ between CLI versions, so always match it to the name) |
| Generate a new API signing RSA key pair? | `Y` |
| Directory for your keys | Press **Enter** to accept the default (`/root/.oci`) |
| Name for your key | Press **Enter** to accept the default (`oci_api_key`) |
| Passphrase for your private key | Enter a passphrase, or type `N/A` for no passphrase |

Example output at the end of the wizard:

```text
Enter a directory for your keys to be created [/root/.oci]:
Enter a name for your key [oci_api_key]:
Public key written to: /root/.oci/oci_api_key_public.pem
Enter a passphrase for your private key ("N/A" for no passphrase):
```

> **Tip:** Use a passphrase for any machine shared with other people. Use `N/A` only for disposable lab machines.

### Step 2.3: Display the public key

```bash
cat /root/.oci/oci_api_key_public.pem
```

Copy the entire output, including the `-----BEGIN PUBLIC KEY-----` and `-----END PUBLIC KEY-----` lines.

### Step 2.4: Upload the public key to your OCI user

1. Sign in to the **OCI Console**.
2. Click the **Profile** icon (top right) and select **My profile**.
3. Under **Resources** on the left, click **Tokens and keys**.
4. Click **Add API key**.
5. Choose **Paste a public key**, paste the key from Step 2.3, and click **Add**.
6. A configuration file preview appears. Confirm the fingerprint matches the one in your `~/.oci/config`, then close the dialog.

### Step 2.5: Secure the key and config permissions

```bash
oci setup repair-file-permissions --file /root/.oci/config
oci setup repair-file-permissions --file /root/.oci/oci_api_key.pem
```

---

## Task 3: Verify the Configuration

Retrieve your tenancy's Object Storage namespace. This is a lightweight call that proves authentication works.

```bash
oci os ns get
```

**Expected result:**

```json
{
  "data": "your-namespace-name"
}
```

### Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `NotAuthenticated` (401) | Public key not uploaded, or fingerprint mismatch | Re-check Step 2.4 and compare fingerprints with `~/.oci/config` |
| `NotAuthorizedOrNotFound` (404) | Wrong user or tenancy OCID | Recheck OCIDs in `~/.oci/config` |
| Permission warnings on key files | Key files readable by other users | Run the commands in Step 2.5 |
| Wrong region used | Incorrect region selected in the wizard | Edit `region=` in `~/.oci/config` to `ap-hyderabad-1` |

---

## Task 4: Install kubectl

### Step 4.1: Install curl

```bash
sudo dnf install -y curl
```

### Step 4.2: Check your CPU architecture

```bash
uname -m
```

- `x86_64` means download the **amd64** binary.
- `aarch64` means download the **arm64** binary.

### Step 4.3: Download kubectl

For **arm64** (aarch64):

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/arm64/kubectl"
```

For **amd64** (x86_64), replace `arm64` with `amd64`:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

### Step 4.4: Install the binary

```bash
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

> **Note:** The original instructions used mode `0777` and the path `/bin/kubectl`. Mode `0755` is recommended because `0777` lets any user modify the binary. If you prefer the original path, use `/bin/kubectl` instead of `/usr/local/bin/kubectl`.

### Step 4.5: Verify kubectl

```bash
kubectl version --client
```

**Expected result:** the client version is printed.

---

## Task 5 (Optional): Connect kubectl to an OKE Cluster

If you have an OKE cluster in Hyderabad, generate a kubeconfig:

```bash
oci ce cluster create-kubeconfig \
  --cluster-id <cluster-ocid> \
  --file $HOME/.kube/config \
  --region ap-hyderabad-1 \
  --token-version 2.0.0 \
  --kube-endpoint PUBLIC_ENDPOINT

kubectl get nodes
```

---

## Summary

You have:

- Installed the OCI CLI on Oracle Linux 9
- Generated an API key pair and registered it with your OCI user
- Configured the CLI for the Hyderabad region and verified access with `oci os ns get`
- Installed `kubectl` for your CPU architecture

## Cleanup

This lab changes only the local machine. To remove everything:

```bash
sudo dnf -y remove python39-oci-cli
sudo rm -f /usr/local/bin/kubectl
rm -rf ~/.oci
```

Also delete the API key from **Profile, Tokens and keys** in the OCI Console.

## References

- [OCI CLI documentation](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/cliconcepts.htm)
- [Installing kubectl on Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
