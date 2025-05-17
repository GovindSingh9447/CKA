# CKA Exam Detailed Notes – Storage Topics

---

## 1. Implement Storage Classes and Dynamic Volume Provisioning

### What is a StorageClass?
- **StorageClass** is a Kubernetes resource that defines different types of storage (e.g., SSD, HDD, network storage) and how they are provisioned.
- Each StorageClass uses a **provisioner** (plugin/driver) to dynamically create PersistentVolumes (PVs) when requested by a PersistentVolumeClaim (PVC).
- Common provisioners: 
  - In-tree (legacy, deprecated): `kubernetes.io/aws-ebs`, `kubernetes.io/gce-pd`, etc.
  - **CSI (Container Storage Interface)** drivers: Modern, recommended way (e.g., `ebs.csi.aws.com`, `pd.csi.storage.gke.io`).

### StorageClass YAML Example
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```
- **provisioner**: Name of the driver.
- **parameters**: Storage-specific settings (e.g., disk type).
- **reclaimPolicy**: What happens to the PV after the PVC is deleted (`Delete` or `Retain`).
- **volumeBindingMode**: When volume binding and provisioning happens; `WaitForFirstConsumer` is recommended for multi-zone clusters.

### Dynamic Provisioning Process
1. User creates a PVC that references a StorageClass.
2. The StorageClass’s provisioner dynamically creates a PV that matches the request.
3. The new PV is bound to the PVC.

---

## 2. Configure Volume Types, Access Modes, and Reclaim Policies

### Volume Types (Supported Backends)
- **hostPath**: Local path on the node (for development/test only)
- **nfs**: Network File System
- **awsElasticBlockStore**, **gcePersistentDisk**, **azureDisk**, etc. (Cloud provider managed)
- **csi**: Generic interface for external storage plugins
- **cephfs**, **glusterfs**, **iscsi**, etc.

### Access Modes
- **ReadWriteOnce (RWO)**: Mounted as read-write by a single node.
- **ReadOnlyMany (ROX)**: Mounted as read-only by many nodes.
- **ReadWriteMany (RWX)**: Mounted as read-write by many nodes.
- **ReadWriteOncePod (RWOP)**: (K8s v1.22+) Mounted as read-write by a single pod.

> Not all volume types support all access modes. For example, AWS EBS only supports RWO.

### Reclaim Policies
- **Retain**: PV is not deleted when PVC is deleted. Manual intervention required to clean up.
- **Delete**: PV and underlying storage are deleted with PVC. Used for dynamically provisioned cloud volumes.
- **Recycle**: Deprecated. Wipes data with a basic scrub and makes PV available again.

### Example PV with Access Modes and Reclaim Policy
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  nfs:
    path: /exported/path
    server: nfs.example.com
```

---

## 3. Manage Persistent Volumes (PV) and Persistent Volume Claims (PVC)

### PersistentVolume (PV)
- Cluster-wide resource that represents a piece of storage in the cluster.
- Created statically (by admin) or dynamically (by StorageClass/provisioner).

### PersistentVolumeClaim (PVC)
- A request for storage by a user/pod.
- Specifies size, access modes, and optionally StorageClass.

### Binding Process (How PVC finds a PV)
- Kubernetes finds an available PV that satisfies the PVC’s requirements.
- If no PV exists but StorageClass is specified, dynamic provisioning is triggered.
- Once PV is bound to PVC, it cannot be bound to another PVC.

### Example: Static PV and PVC
```yaml
# PersistentVolume
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-static
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
  persistentVolumeReclaimPolicy: Retain

# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-static
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```
- The above PVC will bind to the above PV if the size and access modes match.

### Example: Dynamic Provisioning with StorageClass
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dynamic
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: standard
```
- This PVC will trigger dynamic provisioning using the `standard` StorageClass.

### Key Commands
```sh
kubectl get storageclass
kubectl describe storageclass <name>
kubectl get pv
kubectl describe pv <name>
kubectl get pvc
kubectl describe pvc <name>
kubectl delete pv <name>
kubectl delete pvc <name>
```

### Mounting PVC in a Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: html
  volumes:
    - name: html
      persistentVolumeClaim:
        claimName: pvc-dynamic
```

---

## Troubleshooting Tips

- **PVC stuck in Pending**: No matching PV exists or provisioner failed.
- **PV not released**: Check the reclaim policy (`Retain` requires manual cleanup).
- **Pod fails to start**: Check events/logs for mount errors, permissions, or StorageClass issues.
- **Access mode mismatch**: Verify PV and PVC access modes are compatible.

---

## Best Practices

- Prefer **dynamic provisioning** with CSI drivers for most environments.
- Use **Retain** policy for critical data where manual cleanup is preferred; use **Delete** for ephemeral data.
- Monitor storage usage and PV/PVC status regularly.
- For multi-zone clusters, use `WaitForFirstConsumer` for `volumeBindingMode` to ensure volume is provisioned in the correct zone.

---

## References

- [Kubernetes Storage Concepts](https://kubernetes.io/docs/concepts/storage/)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [CSI Drivers](https://kubernetes.io/docs/concepts/storage/volumes/#csi)
