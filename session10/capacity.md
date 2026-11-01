# Demo: Using Capacity Reservations to Provision Managed Nodes on OKE

Create an OCI Compute capacity reservation, then create an OKE managed node pool that launches its worker nodes from that reserved capacity. You will verify that the nodes really consumed the reservation, scale the pool, and see how to opt out of a reservation.

A capacity reservation guarantees that compute capacity is available in an availability domain (AD) when you need it, for example during disaster recovery or an unexpected workload spike. OKE lets you point a managed node pool's placement configuration at a reservation, so new nodes are created from reserved capacity instead of competing for on-demand capacity.

> **Cost note:** reserved capacity is held for you whether or not instances are running in it, so a reservation can be billable. Check the Compute capacity reservation documentation for current billing rules, and delete the reservation at the end of the demo (Lab 9).

## What you will do

| Lab | Step | Done when |
| --- | --- | --- |
| 1 | Check prerequisites | Cluster, subnet, AD and shape chosen |
| 2 | Set up IAM policies | Group and OKE service policies exist |
| 3 | Create a capacity reservation | Reservation is `ACTIVE` with a reserved count |
| 4 | Create a node pool that uses the reservation | Node pool is `ACTIVE`, nodes `Ready` |
| 5 | Verify the nodes use the reservation | Reservation used count increased |
| 6 | Scale the node pool | New nodes also come from the reservation |
| 7 | (Optional) Multiple ADs, one reservation each | Placement config has one reservation per AD |
| 8 | (Optional) Opt out of a reservation | Node pool launches on on-demand capacity |
| 9 | Clean up | Node pool and reservation deleted |

---

## Key rules to remember

- The capacity reservation must **already exist** before you reference it in a node pool.
- The node pool's **shape, availability domain and fault domain must match** the reservation's instance type, availability domain and fault domain.
- If a node pool uses several ADs, you can set a **different reservation for each AD**.
- Changing the reservation on a node pool affects **new nodes only**. Existing worker nodes are not changed.
- If a **default capacity reservation** exists for the AD, it is used automatically unless you pick another reservation or opt out.

---

## Lab 1: Prerequisites

- An OKE cluster and `kubectl` working from Cloud Shell or a VM
- The OCI CLI configured
- A worker node subnet in the cluster's VCN
- A decision on the AD, shape and count to reserve

Set variables (adjust the region and values to your tenancy):

```bash
export REGION=ap-hyderabad-1
export CLUSTER_OCID='<CLUSTER_OCID>'
export COMPARTMENT_OCID='<COMPARTMENT_OCID>'
export SUBNET_OCID='<WORKER_SUBNET_OCID>'

oci ce cluster create-kubeconfig \
  --cluster-id "$CLUSTER_OCID" \
  --region "$REGION" \
  --file "$HOME/.kube/config" \
  --token-version 2.0.0

kubectl get nodes
```

List the availability domains and pick one (copy the full name, for example `Uocm:AP-HYDERABAD-1-AD-1`):

```bash
oci iam availability-domain list \
  --compartment-id "$COMPARTMENT_OCID" --region $REGION \
  --query 'data[].name' --output table
```

Find a node image for your Kubernetes version and shape. In the Console, the easiest route is **Developer Services -> Kubernetes Clusters -> your cluster -> Node pools -> Add node pool** and note the image offered. From the CLI:

```bash
oci ce node-pool-options get --node-pool-option-id all --region $REGION \
  --query 'data.sources[].{name:"source-name",id:"image-id"}' --output table
```

Choose a shape that is available in your region and AD. This lab uses a flexible shape as an example:

```bash
export AD_NAME='<AVAILABILITY_DOMAIN_NAME>'
export NODE_SHAPE='VM.Standard.E4.Flex'
export OCPUS=2
export MEMORY_GBS=16
export IMAGE_OCID='<NODE_IMAGE_OCID>'
export K8S_VERSION='<KUBERNETES_VERSION>'   # for example the version your cluster runs
```

---

## Lab 2: Set up IAM policies

Two sets of permissions are needed.

**1. For the person (group) creating or editing the node pool:**

```text
Allow group <group-name> to use compute-capacity-reservations in <location>
```

If the group is not in the default identity domain, prefix it with the domain name, in the format `group '<identity-domain-name>'/'group-name'`. You can also use `group id <group-ocid>`.

**2. For OKE itself, so it can launch worker nodes from reserved capacity:**

```text
Allow service oke to use compute-capacity-reservations in tenancy
Allow any-user to use compute-capacity-reservations in tenancy where request.principal.type = 'nodepool'
```

You also need permission to create and manage the reservation itself in the Compute service. See the Compute capacity reservation documentation for the exact policy: <https://docs.oracle.com/iaas/Content/Compute/Tasks/reserve-capacity.htm>.

Policies can take a minute or two to apply.

---

## Lab 3: Create a capacity reservation

Reserve capacity in the AD you chose. This example reserves two instances of the flexible shape:

```bash
oci compute capacity-reservation create \
  --availability-domain "$AD_NAME" \
  --compartment-id "$COMPARTMENT_OCID" \
  --display-name oke-demo-reservation \
  --region $REGION \
  --instance-reservation-configs "[{\"instanceShape\":\"$NODE_SHAPE\",\"instanceShapeConfig\":{\"ocpus\":$OCPUS,\"memoryInGBs\":$MEMORY_GBS},\"reservedCount\":2}]"
```

Notes:

- `reservedCount` is the number of instances you are reserving for this shape.
- For a fixed (non-flex) shape, omit `instanceShapeConfig`.
- You can also set a fault domain inside the instance reservation config if you want to pin it.
- Flag and field names can change between CLI versions. If the command errors, check `oci compute capacity-reservation create --help`.
- You can also create the reservation in the Console: **Compute -> Capacity reservations -> Create capacity reservation**.

Save the reservation OCID and check its state:

```bash
export RESERVATION_OCID='<CAPACITY_RESERVATION_OCID>'

oci compute capacity-reservation get \
  --capacity-reservation-id "$RESERVATION_OCID" --region $REGION
```

Expect `lifecycle-state` to be `ACTIVE`, a reserved instance count of 2 and a used instance count of 0.

> **Default reservations:** a reservation created with the default option is used automatically by every instance launched in that AD and tenancy, including OKE nodes. This demo uses an explicit, non-default reservation so the choice is visible in the node pool configuration.

---

## Lab 4: Create a node pool that uses the reservation

Add `capacityReservationId` to the node pool's placement configuration. The AD, subnet and shape must match the reservation.

```bash
oci ce node-pool create \
  --cluster-id "$CLUSTER_OCID" \
  --compartment-id "$COMPARTMENT_OCID" \
  --name reserved-pool \
  --node-image-id "$IMAGE_OCID" \
  --kubernetes-version "$K8S_VERSION" \
  --node-shape "$NODE_SHAPE" \
  --node-shape-config "{\"ocpus\":$OCPUS,\"memoryInGBs\":$MEMORY_GBS}" \
  --size 1 \
  --region $REGION \
  --placement-configs "[{\"availability-domain\":\"$AD_NAME\",\"capacityReservationId\":\"$RESERVATION_OCID\",\"subnet-id\":\"$SUBNET_OCID\"}]"
```

This creates a work request. Wait until the node pool is active and the node joins the cluster:

```bash
oci ce node-pool list --compartment-id "$COMPARTMENT_OCID" \
  --cluster-id "$CLUSTER_OCID" --region $REGION \
  --query 'data[].{name:name,id:id,state:"lifecycle-state"}' --output table

kubectl get nodes -w
```

**Console alternative:** Cluster -> **Node pools** -> **Add node pool**. Under **Node placement configuration**, choose the AD and worker node subnet that match the reservation, then open **Advanced options** and set:

- **Capacity type:** Capacity reservation
- **Capacity reservation:** select `oke-demo-reservation`

If you do not explicitly pick a reservation and a default reservation exists for the AD, the default is used.

---

## Lab 5: Verify the nodes use the reservation

Check the reservation's used count. It should now be 1:

```bash
oci compute capacity-reservation get \
  --capacity-reservation-id "$RESERVATION_OCID" --region $REGION \
  --query 'data.{state:"lifecycle-state",reserved:"reserved-instance-count",used:"used-instance-count"}'
```

Confirm on the node's compute instance. Get the instance OCID from the node's provider ID, then read its capacity reservation:

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,PROVIDER:.spec.providerID

oci compute instance get --instance-id <INSTANCE_OCID> --region $REGION \
  --query 'data.{name:"display-name",shape:shape,reservation:"capacity-reservation-id"}'
```

`reservation` should equal `$RESERVATION_OCID`. In the Console, the reservation's details page shows the instance under it.

---

## Lab 6: Scale the node pool

Scale to two nodes. Both should be launched from the reservation:

```bash
export NODE_POOL_OCID='<NODE_POOL_OCID>'

oci ce node-pool update \
  --node-pool-id "$NODE_POOL_OCID" --size 2 --region $REGION

kubectl get nodes -w
```

Check that the reservation shows 2 used of 2 reserved:

```bash
oci compute capacity-reservation get \
  --capacity-reservation-id "$RESERVATION_OCID" --region $REGION \
  --query 'data.{reserved:"reserved-instance-count",used:"used-instance-count"}'
```

Talking points for the demo:

1. The reservation is now fully used. A third node would not come from reserved capacity, so size reservations to match the node count you must guarantee.
2. Reserved capacity is only guaranteed for the shape and AD you reserved. Mixing shapes or ADs needs more reservation lines or more reservations.

---

## Lab 7 (optional): One reservation per availability domain

For a multi-AD node pool, list one placement entry per AD, each with its own reservation. An AD entry without `capacityReservationId` uses on-demand capacity (or the AD's default reservation, if one exists).

```json
[
  {
    "availability-domain": "<AD_1_NAME>",
    "capacityReservationId": "<RESERVATION_OCID_FOR_AD_1>",
    "subnet-id": "<SUBNET_OCID>"
  },
  {
    "availability-domain": "<AD_2_NAME>",
    "capacityReservationId": "<RESERVATION_OCID_FOR_AD_2>",
    "subnet-id": "<SUBNET_OCID>"
  }
]
```

Save it as `placement.json` and apply it when creating or updating a node pool:

```bash
oci ce node-pool update \
  --node-pool-id "$NODE_POOL_OCID" --region $REGION \
  --placement-configs file://./placement.json
```

Remember that the update affects new nodes only. Existing nodes keep their current properties. You can also set fault domains per entry with `"faultDomains": ["FAULT-DOMAIN-1"]`, which must match the reservation's fault domain if you pinned one.

---

## Lab 8 (optional): Opt out of a reservation

If a default reservation exists for an AD but you want a node pool to use on-demand capacity, set `capacityReservationId` to an empty string:

```json
[
  {
    "availability-domain": "<AD_NAME>",
    "capacityReservationId": "",
    "subnet-id": "<SUBNET_OCID>"
  }
]
```

In the Console, this is **Opt out of default reservation** under **Capacity reservation** in the node placement advanced options.

---

## Lab 9: Clean up

Delete the node pool first so the reserved instances are released, then delete the reservation:

```bash
oci ce node-pool delete --node-pool-id "$NODE_POOL_OCID" --region $REGION --force

# wait until the node pool and its instances are gone, then:
oci compute capacity-reservation delete \
  --capacity-reservation-id "$RESERVATION_OCID" --region $REGION --force
```

Notes:

- A reservation that still has instances running in it cannot be deleted. Check that the used count is 0 first.
- Deleting the reservation stops the billing for the reserved capacity.
- Remove any demo policies you created for this lab if they are no longer needed.

---

## Troubleshooting

| Symptom | Likely cause | Check |
| --- | --- | --- |
| Node pool creation fails with an authorization error | Missing policy for the user or for OKE | Re-check Lab 2 policies, including the `service oke` and `nodepool` statements |
| Nodes do not launch from the reservation | Shape, AD or fault domain does not match the reservation | Compare the node pool placement config with the reservation's instance config |
| Flex shape mismatch | OCPU or memory differs from the reserved configuration | Use the same `ocpus` and `memoryInGBs` as the reservation |
| Node pool uses an unexpected reservation | A default reservation exists for the AD | Set `capacityReservationId` explicitly, or opt out with an empty string |
| Scaling beyond the reserved count | Reservation is fully used | Increase `reservedCount` or add a reservation |
| Updated reservation has no effect on existing nodes | Changes only apply to new nodes | Cycle the nodes, or scale up and drain old ones |
| Reservation cannot be deleted | Instances still use it | Delete the node pool and wait until used count is 0 |
| `capacityReservationId` rejected | Reservation OCID is wrong or in another region or tenancy | `oci compute capacity-reservation get` with the same region |

---

## Sources

- [Using Capacity Reservations to Provision Managed Nodes (OKE)](https://docs.oracle.com/iaas/Content/ContEng/Tasks/contengmakingcapacityreservations.htm)
- [Capacity Reservations (Compute)](https://docs.oracle.com/iaas/Content/Compute/Tasks/reserve-capacity.htm)
- [NodePoolPlacementConfigDetails (OCI Python SDK)](https://docs.oracle.com/en-us/iaas/tools/python/latest/api/container_engine/models/oci.container_engine.models.NodePoolPlacementConfigDetails.html)
- [CreateComputeCapacityReservationDetails (OCI Python SDK)](https://docs.oracle.com/en-us/iaas/tools/python/latest/api/core/models/oci.core.models.CreateComputeCapacityReservationDetails.html)
