# Workshop Lab: Create an Oracle Linux 9 VM in OCI and Connect with SSH

## Overview

In this lab you will generate an SSH key pair, allow SSH traffic through a Network Security Group (NSG), create an Oracle Linux 9 virtual machine in the `oke-api-public` subnet of your VCN, and connect to it over SSH using the key you created.

**Estimated time:** 20 to 30 minutes

## Objectives

By the end of this lab you will be able to:

- Generate an RSA SSH key pair with `ssh-keygen`
- Create an NSG rule that allows inbound SSH
- Launch an Oracle Linux 9 compute instance in a public subnet
- Attach your public key and the NSG to the instance
- Connect to the instance with `ssh -i`

## Prerequisites

| Requirement | Details |
|---|---|
| OCI access | A user able to create compute instances and network security groups |
| Networking | An existing VCN containing a public subnet named `oke-api-public` |
| Local machine | A terminal with the `ssh` and `ssh-keygen` commands (Linux, macOS, or Windows with OpenSSH) |

> **Security note:** Using source `0.0.0.0/0` opens SSH to the entire internet. This is acceptable for a short-lived lab, but for real environments restrict the source to your own public IP (for example `203.0.113.10/32`).

---

## Task 1: Generate the SSH Key Pair

### Step 1.1: Run ssh-keygen

```bash
ssh-keygen -t rsa
```

### Step 1.2: Answer the prompts

| Prompt | What to enter |
|---|---|
| Enter file in which to save the key | `test` |
| Enter passphrase (empty for no passphrase) | Press **Enter** (no passphrase) |
| Enter same passphrase again | Press **Enter** |

This creates two files in the current directory:

| File | Purpose |
|---|---|
| `test` | Private key. Keep it secret and never share it. |
| `test.pub` | Public key. This is the one you upload to OCI. |

### Step 1.3: Display the public key

```bash
cat test.pub
```

Copy the full output. You will paste it in Task 3.

---

## Task 2: Create the Network Security Group

### Step 2.1: Open the NSG creation page

1. Sign in to the **OCI Console**.
2. Open the navigation menu and go to **Networking**, then **Virtual cloud networks**.
3. Click your VCN.
4. Under **Resources**, click **Network security groups**, then **Create network security group**.

### Step 2.2: Name the NSG

- **Name:** `nsg-api`
- Click **Next**.

### Step 2.3: Add the SSH rule

Under **Add security rules**, create an **ingress** rule with these values:

| Field | Value |
|---|---|
| Direction | Ingress |
| Source Type | CIDR |
| Source CIDR | `0.0.0.0/0` |
| IP Protocol | TCP |
| Source Port Range | All (leave blank) |
| Destination Port Range | `22` (SSH) |

Click **Create**.

**Expected result:** the NSG `nsg-api` appears in the list with one ingress rule for TCP port 22.

---

## Task 3: Create the Oracle Linux 9 VM

### Step 3.1: Start instance creation

1. Open the navigation menu and go to **Compute**, then **Instances**.
2. Click **Create instance**.

### Step 3.2: Name and image

- **Name:** enter a name such as `oci-lab-vm`.
- Under **Image and shape**, click **Change image**, select **Oracle Linux**, choose version **9**, and click **Select image**.
- Keep the default shape unless your instructor says otherwise.

### Step 3.3: Configure networking

In the **Networking** section:

| Field | Value |
|---|---|
| Virtual cloud network | Your VCN |
| Subnet | `oke-api-public` |
| Public IPv4 address | **Automatically assign public IPv4 address** (enabled) |

Under **Network security groups**, check **Use network security groups to control traffic**, then select `nsg-api`.

### Step 3.4: Add the SSH public key

In the **Add SSH keys** section:

1. Select **Paste public keys**.
2. Paste the contents of `test.pub` from Step 1.3.

### Step 3.5: Create the instance

Click **Create** and wait until the instance state changes from **Provisioning** to **Running**.

### Step 3.6: Note the public IP address

On the instance details page, copy the **Public IP address**.

---

## Task 4: Connect to the VM with SSH

### Step 4.1: Restrict the private key permissions

SSH refuses keys that other users can read.

```bash
chmod 600 test
```

### Step 4.2: Connect

```bash
ssh -i test opc@<your-public-ip>
```

Replace `<your-public-ip>` with the address from Step 3.6. The default user on Oracle Linux images is `opc`.

If asked `Are you sure you want to continue connecting`, type `yes`.

### Step 4.3: Verify the connection

```bash
cat /etc/oracle-release
whoami
```

**Expected result:** the release shows Oracle Linux 9 and `whoami` prints `opc`.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Connection timed out` | NSG not attached, rule missing, or subnet has no route to an internet gateway | Confirm `nsg-api` is attached to the instance VNIC and the rule allows TCP 22. Check the subnet's route table and security list. |
| `Permission denied (publickey)` | Wrong key, wrong user, or key pasted incorrectly | Use `-i test` and user `opc`. Confirm the pasted public key was a single unbroken line. |
| `UNPROTECTED PRIVATE KEY FILE` | Private key permissions too open | Run `chmod 600 test`. |
| No public IP on the instance | Public IP assignment was not enabled, or the subnet is private | Use a public subnet and enable automatic public IPv4 assignment. |

---

## Summary

You have:

- Generated an RSA key pair named `test` with no passphrase
- Created the `nsg-api` NSG allowing inbound SSH
- Launched an Oracle Linux 9 VM in the `oke-api-public` subnet with your public key
- Connected to it with `ssh -i test opc@<public-ip>`

## Cleanup

To avoid charges and reduce exposure:

1. In the Console, go to **Compute**, then **Instances**, open your instance, and choose **More actions**, then **Terminate**.
2. Delete the `nsg-api` NSG once no resources use it.
3. Delete the local key files if no longer needed:

```bash
rm -f test test.pub
```
