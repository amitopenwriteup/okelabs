# Lab: Using Capacity Reservations to Provision Managed Nodes in OKE

**Estimated time:** 30–40 minutes
**Level:** Intermediate

> **Note:** OCI Console labels and policy syntax change over time. Items most likely to differ are marked **(verify)**.

---

## Concept

A **capacity reservation** guarantees that compute capacity (shape, OCPUs, memory) is available in a given availability domain (AD). Without it, node pool creation or scale-up can fail with an *out of host capacity* error. Attaching a reservation to a managed node pool makes new nodes launch from the reserved capacity.

| Without reservation | With reservation |
|---|---|
| Capacity is best-effort | Capacity is held for you |
| Scale-up may fail in busy ADs | Scale-up draws from the reserved pool |
| No extra reservation object | Reservation is a separate object, billed per current OCI rules (verify) |

## Objective

Create a capacity reservation, allow OKE to use it, create a managed node pool that consumes it, and verify the reserved capacity is used.

## Prerequisites

| Requirement | Details |
|---|---|
| Existing OKE cluster | With a worker subnet and `kubectl` access (Cloud Shell recommended) |
| Compartment | A compartment where you can manage compute and cluster resources |
| Service limits | Enough compute limit for 3 small flex-shape VMs |
| SSH public key | Needed when creating the node pool |

---

## Step 1 – Create a capacity reservation

1. Sign in to the OCI Console and select your **region**.
2. Open the navigation menu → **Compute → Capacity reservations**.
3. Select your **compartment** and click **Create capacity reservation**.
4. Fill in the details:

| Field | Example value |
|---|---|
| Name | `oke-lab-reservation` |
| Compartment | Your lab compartment |
| Availability domain | AD-1 (must match the node pool's AD) |
| Fault domain (optional) | Leave unspecified |
| Reservation configuration → Shape | `VM.Standard.E4.Flex` |
| OCPUs | 1 |
| Memory (GB) | 8 |
| Reserved instance count | 3 |

5. Click **Create** and wait until the state is **Active**.

---

## Step 2 – Allow OKE to use the reservation

OKE launches instances on your behalf, so it must be permitted to use the reservation.

1. Open the navigation menu → **Identity & Security → Policies**.
2. Select the correct compartment and click **Create Policy**.
3. Name: `oke-capacity-reservation-policy`.
4. In the policy builder (manual editor), add **(verify exact syntax in current docs)**:

```
Allow any-user to use compute-capacity-reservations in compartment <compartment-name>
  where request.principal.type = 'cluster'
```

5. Click **Create**.

---

## Step 3 – Create a node pool that uses the reservation

1. Open **Developer Services → Containers & Artifacts → Kubernetes Clusters (OKE)** and click your cluster.
2. Under **Resources**, click **Node pools → Add node pool**.
3. Configure the pool:

| Field | Value |
|---|---|
| Name | `pool-reserved` |
| Node type | Managed |
| Kubernetes version | Same as the control plane |
| Availability domain | **AD-1** (the reservation's AD) |
| Worker node subnet | Your existing worker subnet |
| Shape | `VM.Standard.E4.Flex` (must match the reservation) |
| OCPUs / Memory | 1 OCPU / 8 GB (must match the reservation) |
| Image | Oracle Linux (OKE image) |
| Number of nodes | 2 |
| SSH public key | Paste your public key |

4. Expand **Show advanced options** (or the node placement section).
5. Enable **Use capacity reservation** and select `oke-lab-reservation`.
6. Click **Add**.

> **Important:** The shape, OCPU count and memory of the node pool must match the reservation configuration, and the pool must be placed in the same AD.

---

## Step 4 – Verify the reservation is used

1. Wait until the node pool state is **Active**.
2. In the console, go to **Compute → Capacity reservations → oke-lab-reservation**.
3. Check the reservation details and **Instances** tab. The two worker nodes should appear, and the **used** count should increase to 2 of 3.
4. In Cloud Shell, confirm the nodes joined the cluster:

```bash
kubectl get nodes -o wide
kubectl get nodes -l name=pool-reserved
```

Expected: two nodes in `Ready` state.

---

## Step 5 – Scale up using the remaining reserved capacity

1. Open the cluster → **Node pools → pool-reserved → Scale pool**.
2. Change **Number of nodes** from 2 to 3 and click **Scale**.
3. Watch the nodes:

```bash
kubectl get nodes --watch
```

4. Return to the reservation page. Used count should now be 3 of 3.

---

## Step 6 – (Optional) Observe behavior beyond the reservation

1. Scale `pool-reserved` to 4 nodes.
2. Observe whether the fourth node launches from general capacity or fails with a capacity error. The result depends on your configuration and current AD capacity.
3. Scale back to 3 or lower.

---

## Validation Checklist

- [ ] Capacity reservation is **Active**
- [ ] Policy allowing OKE to use the reservation is in place
- [ ] Node pool `pool-reserved` created with the reservation attached
- [ ] Reservation usage count matches the number of nodes
- [ ] Node pool scaled to use the remaining reserved slot

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Node pool creation fails with authorization error | Missing reservation policy | Review the policy in Step 2 and the compartment it applies to |
| Reservation not selectable in node pool | Shape, AD or compartment mismatch | Match shape and AD exactly; check the compartment |
| Nodes launch but used count does not change | Reservation not attached to the pool | Edit the pool placement configuration and confirm the reservation |
| Scale-up beyond reserved count fails | No general capacity in the AD | Increase the reservation count or add another AD |

---

## Clean-up

1. Delete the node pool: **Cluster → Node pools → pool-reserved → More actions → Delete node pool**.
2. Delete the reservation: **Compute → Capacity reservations → oke-lab-reservation → Delete** (stops reserved-capacity charges).
3. Remove the temporary policy `oke-capacity-reservation-policy`.

---

## Review Questions

1. What problem does a capacity reservation solve for node pool provisioning?
2. Which properties of the node pool must match the reservation?
3. Why does OKE need an IAM policy to use a capacity reservation?
4. What happens to the reservation's used count when you scale the pool up or down?
