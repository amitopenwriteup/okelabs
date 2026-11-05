# Lab: Autoscaling Kubernetes Node Pools with the Cluster Autoscaler in OKE

**Estimated time:** 30 minutes
**Level:** Intermediate

> **Note:** OCI Console labels, add-on configuration keys and policy syntax change over time. Items most likely to differ are marked **(verify)**.

---

## Concept

The **Cluster Autoscaler (CA)** automatically changes the number of worker nodes in a managed node pool:

| Event | CA action |
|---|---|
| Pods are `Pending` because no node has enough free CPU or memory | **Scale up**: adds nodes to the pool (up to the maximum) |
| Nodes are underused and their pods can run elsewhere | **Scale down**: drains and removes nodes (down to the minimum) |

The CA reacts to **pod resource requests**, not to actual CPU usage. Pods without resource requests do not trigger scale-up.

| Autoscaler | Scales | Trigger |
|---|---|---|
| Horizontal Pod Autoscaler (HPA) | Pod replicas | CPU, memory or custom metrics |
| Cluster Autoscaler | Worker nodes | Unschedulable pods, underused nodes |

HPA adds pods; if they cannot be scheduled, the Cluster Autoscaler adds nodes.

## Objective

Enable the Cluster Autoscaler on a managed node pool, trigger a scale-up with pending pods, trigger a scale-down by removing the workload, and test it together with HPA.

## Prerequisites

| Requirement | Details |
|---|---|
| Cluster type | **Enhanced** cluster (for the managed add-on) |
| Node pool | A managed node pool, for example `pool-autoscale` (created in Step 1) |
| Not supported | Virtual nodes |
| Access | `kubectl` through Cloud Shell |
| Service limits | Room for 2–4 extra small flex-shape VMs |

---

## Step 1 – Create a Node Pool for Autoscaling

1. Open **Developer Services → Containers & Artifacts → Kubernetes Clusters (OKE)** and select your cluster.
2. Under **Resources**, click **Node pools → Add node pool**.
3. Configure:

| Field | Value |
|---|---|
| Name | `pool-autoscale` |
| Node type | Managed |
| Kubernetes version | Same as the control plane |
| Worker node subnet | Your worker subnet |
| Shape | `VM.Standard.E5.Flex` |
| OCPUs / Memory | 1 OCPU / 8 GB |
| Number of nodes | 1 |
| SSH public key | Paste your public key |

4. Click **Add** and wait for the pool to become **Active**.
5. Copy the **node pool OCID** from the node pool details page. You need it in Step 3.

```bash
kubectl get nodes -l name=pool-autoscale
```

> **Tip:** If other node pools exist, the test pods in Step 4 may land on them. Use the `nodeSelector` shown in Step 4 so pods target only `pool-autoscale`.

---

## Step 2 – Grant the Autoscaler Permissions

Create a **dynamic group** that contains the worker nodes, then a policy for that group. The add-on's own note in the console says the instances (worker nodes) must be allowed to manage node pools.

1. Open **Identity & Security → Domains → (your domain) → Dynamic groups → Create dynamic group**.
2. Name: `oke-workers-dg`. Matching rule **(verify)**, replacing the OCID with your compartment's OCID:

```
ALL {instance.compartment.id = '<compartment-ocid>'}
```

3. Open **Identity & Security → Policies**, select the cluster's compartment, click **Create Policy**, and name it `oke-cluster-autoscaler-policy`.
4. Add these statements **(verify)**, replacing `<compartment-name>`:

```
Allow dynamic-group oke-workers-dg to manage cluster-node-pools in compartment <compartment-name>
Allow dynamic-group oke-workers-dg to manage instance-family in compartment <compartment-name>
Allow dynamic-group oke-workers-dg to use subnets in compartment <compartment-name>
Allow dynamic-group oke-workers-dg to read virtual-network-family in compartment <compartment-name>
Allow dynamic-group oke-workers-dg to use vnics in compartment <compartment-name>
Allow dynamic-group oke-workers-dg to inspect compartments in compartment <compartment-name>
```

5. Click **Create**.

> If your tenancy uses workload identity instead, the add-on documentation lists the equivalent policy and the matching authentication option. Follow the link in the add-on's warning note ("Working with the Cluster Autoscaler as a Cluster Add-on").

---

## Step 3 – Enable the Cluster Autoscaler Add-on

1. Open the cluster → **Resources → Add-ons → Manage add-ons**.
2. Select **Cluster Autoscaler** and enable it.
3. Choose how the add-on is updated: **Automatic updates** (the add-on updates itself) or **Choose a version** (you pin a specific add-on version). This lab uses **Automatic updates**.
4. Read the warning box. It repeats the `nodes` format and notes that a policy must allow the worker node instances to manage node pools, **plus other policy statements related to initializing worker nodes**. Use its link, "Working with the Cluster Autoscaler as a Cluster Add-on", to confirm the complete list.
5. In the **Configuration** section, open the **Option** dropdown, choose **nodes**, and enter the value **(verify)**:

| Option | Example value | Meaning |
|---|---|---|
| `nodes` | `1:4:<node-pool-ocid>` | `min-num-of-nodes:max-num-of-nodes:OCID-of-nodepool` |

For more than one pool, separate the entries with commas:

```
1:4:<first-nodepool-ocid>,1:3:<second-nodepool-ocid>
```

6. Click **Save changes** and wait for the add-on status to become **Active**.
7. Verify in Cloud Shell:

```bash
kubectl get pods -n kube-system | grep cluster-autoscaler
```


> **Important:** Once a pool is managed by the autoscaler, do not resize it manually. The pool size must stay within the `min`–`max` range.

---


