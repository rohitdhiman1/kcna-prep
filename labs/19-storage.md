# Lab 19: Persistent Storage in Kubernetes

## Objective

Create a PersistentVolume and PersistentVolumeClaim, mount storage into a pod, verify data persistence across pod restarts, then set up a StorageClass for dynamic provisioning.

## Prerequisites

- A running Kubernetes cluster (kind, minikube, or cloud).
- `kubectl` installed and configured.
- Read the concept file: [Storage](../concepts/storage.md)

---

## Step 1: Understand the Storage Model

Kubernetes storage uses three main resources:

| Resource | Purpose | Created By |
|----------|---------|-----------|
| **PersistentVolume (PV)** | A piece of storage provisioned in the cluster | Admin (or dynamically by StorageClass) |
| **PersistentVolumeClaim (PVC)** | A request for storage by a pod | Developer |
| **StorageClass** | Defines how to dynamically provision PVs | Admin |

The flow is: Pod references PVC, PVC binds to PV, PV maps to actual storage.

---

## Step 2: Create a PersistentVolume (Static Provisioning)

Create a PV using `hostPath` (local directory on the node). This is suitable for single-node clusters (kind, minikube) and testing only.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: lab-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/lab-pv-data
    type: DirectoryOrCreate
EOF
```

### Verify the PV

```bash
kubectl get pv lab-pv
```

Expected output:

```
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   AGE
lab-pv   1Gi        RWO            Retain           Available                          10s
```

Key fields:
- **STATUS: Available** — the PV is ready to be claimed.
- **ACCESS MODES: RWO** (ReadWriteOnce) — can be mounted as read-write by a single node.
- **RECLAIM POLICY: Retain** — data is kept when the PVC is deleted.

### Describe the PV for full details

```bash
kubectl describe pv lab-pv
```

---

## Step 3: Create a PersistentVolumeClaim

A PVC is how a pod requests storage:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lab-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
EOF
```

### Verify the PVC is bound

```bash
kubectl get pvc lab-pvc
```

Expected output:

```
NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
lab-pvc   Bound    lab-pv   1Gi        RWO                           5s
```

The PVC has been **Bound** to `lab-pv`. Kubernetes matched the PVC's requirements (500Mi, RWO) against available PVs and found `lab-pv` (1Gi, RWO).

### Verify the PV status changed

```bash
kubectl get pv lab-pv
```

The STATUS should now show `Bound` and the CLAIM column shows `default/lab-pvc`.

---

## Step 4: Mount the PVC in a Pod

Create a pod that uses the PVC:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: storage-writer
spec:
  containers:
  - name: writer
    image: busybox:1.36
    command: ["sh", "-c", "echo 'Data written at '$(date) > /data/testfile.txt && cat /data/testfile.txt && sleep 3600"]
    volumeMounts:
    - name: my-storage
      mountPath: /data
  volumes:
  - name: my-storage
    persistentVolumeClaim:
      claimName: lab-pvc
EOF
```

### Wait for the pod to be running

```bash
kubectl get pod storage-writer --watch
```

Press `Ctrl+C` once it shows `Running`.

### Verify the file was written

```bash
kubectl exec storage-writer -- cat /data/testfile.txt
```

Expected output:

```
Data written at Sat Mar 14 12:00:00 UTC 2026
```

### Write additional data

```bash
kubectl exec storage-writer -- sh -c "echo 'Additional line of data' >> /data/testfile.txt"
kubectl exec storage-writer -- cat /data/testfile.txt
```

---

## Step 5: Delete the Pod and Verify Data Persists

### Delete the pod

```bash
kubectl delete pod storage-writer
```

### Verify the pod is gone

```bash
kubectl get pods
```

### Check the PVC is still bound

```bash
kubectl get pvc lab-pvc
```

The PVC should still show `Bound`. The PVC and PV are independent of the pod lifecycle.

### Recreate the pod and verify data

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: storage-reader
spec:
  containers:
  - name: reader
    image: busybox:1.36
    command: ["sh", "-c", "cat /data/testfile.txt && sleep 3600"]
    volumeMounts:
    - name: my-storage
      mountPath: /data
  volumes:
  - name: my-storage
    persistentVolumeClaim:
      claimName: lab-pvc
EOF
```

### Check the data

```bash
kubectl logs storage-reader
```

Expected output:

```
Data written at Sat Mar 14 12:00:00 UTC 2026
Additional line of data
```

The data survived the pod deletion and is available in the new pod. This is the purpose of persistent storage.

---

## Step 6: Understand Reclaim Policies

When a PVC is deleted, the reclaim policy determines what happens to the PV:

| Reclaim Policy | Behavior |
|---------------|----------|
| **Retain** | PV is kept with data intact; must be manually reclaimed |
| **Delete** | PV and underlying storage are deleted automatically |
| **Recycle** | Deprecated; data is scrubbed and PV is made available again |

### Test the Retain policy

```bash
# Delete the pod first
kubectl delete pod storage-reader

# Delete the PVC
kubectl delete pvc lab-pvc

# Check the PV status
kubectl get pv lab-pv
```

Expected output:

```
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM             STORAGECLASS   AGE
lab-pv   1Gi        RWO            Retain           Released   default/lab-pvc                  5m
```

The PV status is now `Released`. The data is preserved, but the PV cannot be claimed by a new PVC until an admin manually reclaims it.

### Clean up the static PV

```bash
kubectl delete pv lab-pv
```

---

## Step 7: Create a StorageClass

A StorageClass enables dynamic provisioning -- PVs are created automatically when a PVC is made.

### Check existing StorageClasses

```bash
kubectl get storageclass
```

Most clusters come with a default StorageClass:

```
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                 10d
```

### Create a custom StorageClass

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: lab-storage
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
EOF
```

**Note:** The provisioner depends on your cluster:
- **kind**: `rancher.io/local-path`
- **minikube**: `k8s.io/minikube-hostpath`
- **AWS EKS**: `ebs.csi.aws.com`
- **GKE**: `pd.csi.storage.gke.io`

Adjust the provisioner to match your cluster.

### Verify

```bash
kubectl get storageclass lab-storage
```

---

## Step 8: Test Dynamic Provisioning

Create a PVC that references the StorageClass. No PV is needed -- it will be created automatically.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  storageClassName: lab-storage
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 256Mi
EOF
```

### Check the PVC status

```bash
kubectl get pvc dynamic-pvc
```

With `WaitForFirstConsumer`, the PVC stays `Pending` until a pod uses it:

```
NAME          STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
dynamic-pvc   Pending                                      lab-storage    10s
```

### Create a pod that uses the dynamic PVC

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: dynamic-pod
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "echo 'Dynamic storage works!' > /data/dynamic.txt && sleep 3600"]
    volumeMounts:
    - name: dynamic-vol
      mountPath: /data
  volumes:
  - name: dynamic-vol
    persistentVolumeClaim:
      claimName: dynamic-pvc
EOF
```

### Verify the PVC is now Bound

```bash
kubectl get pvc dynamic-pvc
```

Expected output:

```
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
dynamic-pvc   Bound    pvc-a1b2c3d4-e5f6-7890-abcd-ef1234567890   256Mi      RWO            lab-storage    30s
```

A PV was automatically created and bound. Check it:

```bash
kubectl get pv
```

You should see a dynamically provisioned PV with an auto-generated name.

### Verify the data

```bash
kubectl exec dynamic-pod -- cat /data/dynamic.txt
```

Expected output:

```
Dynamic storage works!
```

---

## Step 9: Access Modes and Volume Types (Reference)

### Access Modes

| Mode | Abbreviation | Description |
|------|-------------|-------------|
| ReadWriteOnce | RWO | Read-write by a single node |
| ReadOnlyMany | ROX | Read-only by many nodes |
| ReadWriteMany | RWX | Read-write by many nodes |
| ReadWriteOncePod | RWOP | Read-write by a single pod (Kubernetes 1.27+) |

### Common Volume Types

| Type | Use Case |
|------|---------|
| `hostPath` | Single-node testing only |
| `nfs` | Shared storage across nodes |
| `awsElasticBlockStore` | AWS EBS volumes |
| `gcePersistentDisk` | GCP persistent disks |
| `csi` | Any CSI-compatible storage driver |
| `emptyDir` | Temporary per-pod storage (not persistent) |

---

## Cleanup

```bash
kubectl delete pod dynamic-pod
kubectl delete pvc dynamic-pvc
kubectl delete storageclass lab-storage
```

---

## Key Takeaways

1. **PersistentVolumes (PVs)** represent physical storage; **PersistentVolumeClaims (PVCs)** are requests for that storage.
2. Data in a PVC **survives pod restarts and deletions** -- that is the whole point of persistent storage.
3. **Static provisioning** requires an admin to create PVs manually before PVCs can bind.
4. **Dynamic provisioning** with a StorageClass automatically creates PVs when PVCs are submitted.
5. The **reclaim policy** (Retain vs Delete) controls what happens to data when a PVC is released.
6. `hostPath` volumes are for single-node testing only; production clusters use CSI drivers.
7. `WaitForFirstConsumer` delays PV creation until a pod is scheduled, ensuring node-local storage works correctly.

---

## Next Steps

- Read about [Autoscaling](../concepts/autoscaling.md) to learn how Kubernetes scales workloads.
- Try [Lab 20: Autoscaling](20-autoscaling.md) to configure Horizontal Pod Autoscalers.
