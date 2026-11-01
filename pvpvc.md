# Workshop: Storage for Kubernetes Clusters on OKE (Based on Existing Storage Classes)

**Region:** `ap-hyderabad-1`
**Approach:** Reuse the existing `oci-bv` StorageClass wherever possible. Create new classes only by **cloning** `oci-bv`.

---

## 0. Starting Point

```bash
export REGION=ap-hyderabad-1
export CLUSTER_OCID='ocid1.cluster.oc1.ap-hyderabad-1.aaaaaaaa2e3vt7rsr45wjuuqk5nvplxfrfiabgt6mirm3gc7bcmqdvg6r7ha'

kubectl get pv
kubectl get sc
```

```
No resources found

NAME               PROVISIONER                       RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
oci                oracle.com/oci                    Delete          Immediate              false                  23h
oci-bv (default)   blockvolume.csi.oraclecloud.com   Delete          WaitForFirstConsumer   true                   23h
```

| Class | What we do with it |
|---|---|
| `oci-bv` (default) | **Use directly** for Modules 1 and 2. **Clone** for Modules 3 and 4. |
| `oci` | Do not use. Legacy provisioner, expansion disabled. |
| (none for FSS) | No file storage class exists yet, so Module 5 needs a new one. |

### Inspect the existing class

```bash
kubectl get sc oci-bv -o yaml
```

Important rules about StorageClasses:

- `parameters` are **immutable**. You cannot edit `oci-bv` to add `vpusPerGB` or `fstype`. Create a new class instead.
- Because `oci-bv` is the default, any PVC that omits `storageClassName` uses it automatically.
- `WaitForFirstConsumer` means a PVC stays `Pending` until a pod uses it. This is normal.

### Helper: clone `oci-bv` into a new class

Use this pattern in Modules 3 and 4.

```bash
# Export, strip cluster-generated fields, and rename
kubectl get sc oci-bv -o yaml \
 | sed -e '/^  uid:/d' \
       -e '/^  resourceVersion:/d' \
       -e '/^  creationTimestamp:/d' \
       -e '/storageclass.kubernetes.io\/is-default-class/d' \
       -e 's/^  name: oci-bv$/  name: NEW_NAME/' \
 > sc-clone.yaml

cat sc-clone.yaml
```

Then edit `sc-clone.yaml`: set `name`, and add your parameters under `parameters:`. Make sure the clone is **not** marked as a default class, so `oci-bv` stays the only default.

---

## 1. Provisioning PVCs on the Block Volume Service (uses `oci-bv` as is)

Minimum size is 50 GiB. Access mode is ReadWriteOnce.

**PVC**

```yaml
# pvc-bv.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: bv-claim
spec:
  storageClassName: oci-bv
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 50Gi
```

**Pod**

```yaml
# pod-bv.yaml
apiVersion: v1
kind: Pod
metadata:
  name: bv-app
spec:
  containers:
    - name: app
      image: docker.io/library/busybox:1.36
      command: ["sh", "-c", "echo 'hello from block volume' > /data/hello.txt; sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: bv-claim
```

```bash
kubectl apply -f pvc-bv.yaml
kubectl get pvc bv-claim          # Pending (expected)

kubectl apply -f pod-bv.yaml
kubectl get pvc,pv                # Bound, and a PV now exists
kubectl exec bv-app -- cat /data/hello.txt
kubectl exec bv-app -- df -h /data
```

**Persistence test**

```bash
kubectl delete pod bv-app
kubectl apply -f pod-bv.yaml
kubectl exec bv-app -- cat /data/hello.txt
```

---

## 2. Expanding a Block Volume (uses `oci-bv`, which has expansion enabled)

`ALLOWVOLUMEEXPANSION` is `true` on `oci-bv`, so no class changes are needed.

```bash
kubectl exec bv-app -- df -h /data

kubectl patch pvc bv-claim -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'

kubectl describe pvc bv-claim
kubectl get pvc bv-claim -w
kubectl exec bv-app -- df -h /data      # now ~100G
```

Rules: size can only grow, never shrink. PVCs created from the legacy `oci` class cannot be expanded.

---

## 3. Specifying Block Volume Performance (clone of `oci-bv` with `vpusPerGB`)

| `vpusPerGB` | Tier |
|---|---|
| `0` | Lower Cost |
| `10` | Balanced |
| `20` | Higher Performance |
| `30` to `120` | Ultra High Performance (needs supported shape and multipath) |

Check current OCI documentation for the exact limits.

**1. Clone and edit**

Using the helper above, set `NEW_NAME` to `oci-bv-high-perf`, then add the parameter. The resulting class should look like this:

```yaml
# sc-bv-high.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: oci-bv-high-perf
provisioner: blockvolume.csi.oraclecloud.com
parameters:
  vpusPerGB: "20"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```bash
kubectl apply -f sc-bv-high.yaml
kubectl get sc
```

**2. PVC and pod**

```yaml
# pvc-high.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: bv-high-claim
spec:
  storageClassName: oci-bv-high-perf
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 50Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: bv-high-app
spec:
  containers:
    - name: app
      image: docker.io/library/busybox:1.36
      command: ["sleep", "3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: bv-high-claim
```

**3. Verify in OCI**

```bash
kubectl apply -f pvc-high.yaml

VOL_ID=$(kubectl get pv $(kubectl get pvc bv-high-claim -o jsonpath='{.spec.volumeName}') \
  -o jsonpath='{.spec.csi.volumeHandle}')

oci bv volume get --volume-id $VOL_ID --region $REGION \
  --query 'data.{name:"display-name",vpus:"vpus-per-gb",size:"size-in-gbs"}'
```

Expected `vpus`: `20`.

---

## 4. Specifying File System Types for Block Volumes (clone of `oci-bv` with `fstype`)

Supported types are typically `ext3`, `ext4` and `xfs`.

```yaml
# sc-bv-xfs.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: oci-bv-xfs
provisioner: blockvolume.csi.oraclecloud.com
parameters:
  csi.storage.k8s.io/fstype: xfs
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```yaml
# pvc-xfs.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: bv-xfs-claim
spec:
  storageClassName: oci-bv-xfs
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 50Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: bv-xfs-app
spec:
  containers:
    - name: app
      image: docker.io/library/busybox:1.36
      command: ["sleep", "3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: bv-xfs-claim
```

```bash
kubectl apply -f sc-bv-xfs.yaml -f pvc-xfs.yaml
kubectl exec bv-xfs-app -- sh -c "mount | grep ' /data '"     # shows type xfs
```

> Tip: to combine performance and file system type, put both `vpusPerGB` and `csi.storage.k8s.io/fstype` under `parameters:` in one class.

---

## 5. Provisioning PVCs on the File Storage Service

No FSS class exists on this cluster, so this is the one place we create a class from scratch.

| | Block Volume (`oci-bv`) | File Storage |
|---|---|---|
| Access mode | RWO | **RWX** |
| Driver | `blockvolume.csi.oraclecloud.com` | `fss.csi.oraclecloud.com` |
| Binding | `WaitForFirstConsumer` | Immediate |

**Network rules** on the mount target subnet and worker subnet:

| Protocol | Ports |
|---|---|
| TCP | 2048, 2049, 2050 |
| UDP | 2048 |
| TCP and UDP | 111 |

Also confirm an IAM policy for `file-family`.

---

## 6. Demo: PVC on a New File System

```bash
export COMPARTMENT_OCID=<compartment OCID>
export SUBNET_OCID=<mount target subnet OCID>
oci iam availability-domain list --region $REGION --query 'data[].name'
export AD_NAME=<name from above>
```

```yaml
# sc-fss.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: oci-fss
provisioner: fss.csi.oraclecloud.com
parameters:
  availabilityDomain: <AD_NAME>
  mountTargetSubnetOcid: <SUBNET_OCID>
  compartmentOcid: <COMPARTMENT_OCID>
  exportPath: /FileSystem1
  exportOptions: '[{"source":"10.0.0.0/16","requirePrivilegedSourcePort":false,"access":"READ_WRITE","identitySquash":"NONE"}]'
```

Set `source` to your worker node subnet CIDR. To reuse a mount target, replace `mountTargetSubnetOcid` with `mountTargetOcid`.

```yaml
# pvc-fss.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fss-claim
spec:
  storageClassName: oci-fss
  accessModes: ["ReadWriteMany"]
  resources:
    requests:
      storage: 50Gi        # required by Kubernetes, not enforced by FSS
---
apiVersion: v1
kind: Pod
metadata:
  name: fss-writer
spec:
  containers:
    - name: app
      image: docker.io/library/busybox:1.36
      command: ["sh", "-c", "echo 'shared data' > /shared/msg.txt; sleep 3600"]
      volumeMounts:
        - name: shared
          mountPath: /shared
  volumes:
    - name: shared
      persistentVolumeClaim:
        claimName: fss-claim
---
apiVersion: v1
kind: Pod
metadata:
  name: fss-reader
spec:
  containers:
    - name: app
      image: docker.io/library/busybox:1.36
      command: ["sleep", "3600"]
      volumeMounts:
        - name: shared
          mountPath: /shared
  volumes:
    - name: shared
      persistentVolumeClaim:
        claimName: fss-claim
```

```bash
kubectl apply -f sc-fss.yaml -f pvc-fss.yaml
kubectl get pvc fss-claim
kubectl get pods -o wide
kubectl exec fss-reader -- cat /shared/msg.txt
```

---

## 7. Demo: PVC on an Existing File System

```bash
oci fs file-system list --compartment-id $COMPARTMENT_OCID --availability-domain $AD_NAME --region $REGION
oci fs mount-target list --compartment-id $COMPARTMENT_OCID --availability-domain $AD_NAME --region $REGION
oci fs export list --file-system-id <FS_OCID> --region $REGION
```

`volumeHandle` format: `<FileSystemOCID>:<MountTargetIP>:<ExportPath>`

```yaml
# fss-existing.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: fss-existing-pv
spec:
  capacity:
    storage: 50Gi
  accessModes: ["ReadWriteMany"]
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: fss.csi.oraclecloud.com
    volumeHandle: "<FS_OCID>:<MOUNT_TARGET_IP>:/FileSystem1"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fss-existing-claim
spec:
  storageClassName: ""          # prevents the default oci-bv class being applied
  volumeName: fss-existing-pv
  accessModes: ["ReadWriteMany"]
  resources:
    requests:
      storage: 50Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: fss-existing-app
spec:
  containers:
    - name: app
      image: docker.io/library/busybox:1.36
      command: ["sh", "-c", "ls -la /mnt/fs; sleep 3600"]
      volumeMounts:
        - name: fs
          mountPath: /mnt/fs
  volumes:
    - name: fs
      persistentVolumeClaim:
        claimName: fss-existing-claim
```

```bash
kubectl apply -f fss-existing.yaml
kubectl get pv,pvc
kubectl logs fss-existing-app
```

---

## 8. Cleanup (restores the original two storage classes)

```bash
kubectl delete pod bv-app bv-high-app bv-xfs-app fss-writer fss-reader fss-existing-app --ignore-not-found
kubectl delete pvc bv-claim bv-high-claim bv-xfs-claim fss-claim fss-existing-claim --ignore-not-found
kubectl delete pv fss-existing-pv --ignore-not-found
kubectl delete sc oci-bv-high-perf oci-bv-xfs oci-fss --ignore-not-found

kubectl get sc      # back to: oci, oci-bv (default)
kubectl get pv      # No resources found
```

The original `oci` and `oci-bv` classes are never modified.

---

## Quick Reference

| Task | Class | Change needed |
|---|---|---|
| Basic PVC | `oci-bv` | None |
| Expand | `oci-bv` | `kubectl patch pvc` |
| Performance | clone of `oci-bv` | add `vpusPerGB` |
| File system type | clone of `oci-bv` | add `csi.storage.k8s.io/fstype` |
| Shared RWX storage | new `oci-fss` | create class |
| Existing file system | none | static PV and PVC with `storageClassName: ""` |
