# Storage

> Kubernetes uses the Container Storage Interface (CSI) to integrate with storage systems. Persistent Volumes (PV) and Persistent Volume Claims (PVC) decouple storage from pods, and StorageClasses enable dynamic provisioning.

---

## The Storage Problem

Containers are ephemeral — when a container restarts, its filesystem is reset. Kubernetes needs a way to:

1. **Persist data** beyond the life of a container or pod.
2. **Share data** between containers in the same pod.
3. **Abstract storage** so pods don't need to know the storage backend.

---

## Container Storage Interface (CSI)

CSI is the **standard API** for storage plugins in Kubernetes, similar to CRI (runtimes) and CNI (networking).

```
  CSI Architecture

  ┌──────────────────────────────────────────────────────┐
  │                   Kubernetes                          │
  │                                                       │
  │  ┌────────────┐     ┌──────────────────┐             │
  │  │  kubelet   │     │  Controller      │             │
  │  │            │     │  Manager         │             │
  │  └─────┬──────┘     └────────┬─────────┘             │
  │        │ CSI                  │ CSI                    │
  │        │ Node                │ Controller             │
  │        │ Plugin              │ Plugin                 │
  └────────┼─────────────────────┼────────────────────────┘
           │                     │
           ▼                     ▼
  ┌──────────────────────────────────────────────────────┐
  │                CSI Driver                             │
  │         (e.g., EBS CSI, NFS CSI, Ceph CSI)           │
  │                                                       │
  │  - Provisions volumes                                 │
  │  - Attaches volumes to nodes                          │
  │  - Mounts volumes into pods                           │
  └──────────────────────────────────────────────────────┘
           │
           ▼
  ┌──────────────────────────────────────────────────────┐
  │             Storage Backend                           │
  │   (AWS EBS, GCE PD, Azure Disk, NFS, Ceph, etc.)    │
  └──────────────────────────────────────────────────────┘
```

Before CSI, storage plugins were compiled into the Kubernetes binary ("in-tree" plugins). CSI made them external and pluggable.

---

## Volume Types

### Ephemeral Volumes

These exist only as long as the pod exists:

#### emptyDir

- Created when a pod is assigned to a node.
- Starts empty.
- All containers in the pod can read/write.
- **Deleted when the pod is removed** from the node.
- Stored on the node's disk (or in memory with `medium: Memory`).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-data
spec:
  containers:
  - name: writer
    image: busybox
    command: ["/bin/sh", "-c", "echo hello > /data/hello.txt; sleep 3600"]
    volumeMounts:
    - name: shared
      mountPath: /data
  - name: reader
    image: busybox
    command: ["/bin/sh", "-c", "cat /data/hello.txt; sleep 3600"]
    volumeMounts:
    - name: shared
      mountPath: /data
  volumes:
  - name: shared
    emptyDir: {}
```

#### hostPath

- Mounts a file or directory from the **host node's filesystem** into the pod.
- Data persists across pod restarts (on the same node).
- **Not recommended for production** — ties the pod to a specific node.
- Use cases: accessing Docker socket, node-level log files.

```yaml
volumes:
- name: host-data
  hostPath:
    path: /var/log/host
    type: Directory
```

```
  Ephemeral vs Persistent

  ┌─── emptyDir ────┐     ┌─── hostPath ───┐     ┌─── PV/PVC ─────┐
  │                  │     │                │     │                 │
  │ Pod lifetime     │     │ Node lifetime  │     │ Independent     │
  │                  │     │                │     │ lifetime        │
  │ Deleted with pod │     │ Tied to node   │     │                 │
  │                  │     │                │     │ Survives pod    │
  │ Good for: temp   │     │ Good for: logs │     │ and node        │
  │ data, caches     │     │ node-level data│     │ restarts        │
  │                  │     │ NOT for prod   │     │                 │
  │ Multi-container  │     │                │     │ Good for:       │
  │ data sharing     │     │                │     │ databases,      │
  │                  │     │                │     │ stateful apps   │
  └──────────────────┘     └────────────────┘     └─────────────────┘
```

---

## Persistent Volumes (PV) and Persistent Volume Claims (PVC)

PV and PVC separate storage **provisioning** from storage **consumption**:

```
  PV / PVC / Pod Relationship

  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │   Cluster Admin                    Application Developer    │
  │   (or dynamic provisioner)         (pod creator)            │
  │                                                             │
  │   Creates PV                       Creates PVC              │
  │   ┌──────────────────┐            ┌──────────────────┐     │
  │   │ PersistentVolume │            │ PersistentVolume │     │
  │   │                  │◄──(bind)──►│ Claim            │     │
  │   │ Capacity: 10Gi   │            │                  │     │
  │   │ AccessMode: RWO  │            │ Request: 5Gi     │     │
  │   │ StorageClass: ssd│            │ AccessMode: RWO  │     │
  │   │                  │            │ StorageClass: ssd│     │
  │   │ (cluster-scoped) │            │ (namespace-      │     │
  │   │                  │            │  scoped)         │     │
  │   └──────────────────┘            └────────┬─────────┘     │
  │                                            │               │
  │                                            │ referenced by │
  │                                            ▼               │
  │                                   ┌──────────────────┐     │
  │                                   │      Pod         │     │
  │                                   │                  │     │
  │                                   │ volumes:         │     │
  │                                   │ - persistentVol- │     │
  │                                   │   umeClaim:      │     │
  │                                   │   claimName: pvc │     │
  │                                   └──────────────────┘     │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

### PersistentVolume (PV)

A **cluster-level resource** representing a piece of storage. Created by the admin or dynamically by a StorageClass.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:                       # For local testing only
    path: /mnt/data
```

### PersistentVolumeClaim (PVC)

A **namespace-scoped resource** that represents a user's request for storage. Kubernetes finds a matching PV and binds them.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

### Using a PVC in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

---

## Access Modes

Access modes define how a volume can be mounted:

| Mode | Abbreviation | Description |
|------|-------------|-------------|
| **ReadWriteOnce** | RWO | Can be mounted as read-write by a **single node** |
| **ReadOnlyMany** | ROX | Can be mounted as read-only by **many nodes** |
| **ReadWriteMany** | RWX | Can be mounted as read-write by **many nodes** |
| **ReadWriteOncePod** | RWOP | Can be mounted as read-write by a **single pod** (K8s 1.22+) |

```
  Access Modes

  RWO (ReadWriteOnce)           ROX (ReadOnlyMany)
  ┌──────┐                      ┌──────┐  ┌──────┐  ┌──────┐
  │Node 1│                      │Node 1│  │Node 2│  │Node 3│
  │ R/W  │                      │  R   │  │  R   │  │  R   │
  └──┬───┘                      └──┬───┘  └──┬───┘  └──┬───┘
     │                              │         │         │
     ▼                              └────┬────┘─────────┘
  ┌──────┐                           ┌───▼───┐
  │  PV  │                           │  PV   │
  └──────┘                           └───────┘
  Only one node                      Many nodes, read-only

  RWX (ReadWriteMany)
  ┌──────┐  ┌──────┐  ┌──────┐
  │Node 1│  │Node 2│  │Node 3│
  │ R/W  │  │ R/W  │  │ R/W  │
  └──┬───┘  └──┬───┘  └──┬───┘
     │         │         │
     └────┬────┘─────────┘
       ┌──▼───┐
       │  PV  │ (e.g., NFS, CephFS)
       └──────┘
  Many nodes, read-write
```

Not all storage backends support all access modes. For example:
- AWS EBS: RWO only
- NFS: RWO, ROX, RWX
- Azure Files: RWO, ROX, RWX

---

## StorageClasses and Dynamic Provisioning

Without StorageClasses, an admin must pre-create PVs. **StorageClasses enable dynamic provisioning** — Kubernetes automatically creates PVs when a PVC is submitted.

```
  Static vs Dynamic Provisioning

  Static Provisioning:
  Admin creates PV ──► User creates PVC ──► Kubernetes binds them

  Dynamic Provisioning:
  Admin creates StorageClass ──► User creates PVC ──► Kubernetes
  (one-time setup)                (references SC)       auto-creates PV
                                                        and binds it
```

### StorageClass Definition

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs    # CSI driver name
parameters:
  type: gp3                           # Storage-specific parameters
  iopsPerGB: "10"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### PVC Referencing a StorageClass

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fast-storage
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast-ssd    # References the StorageClass
```

```
  StorageClass / PVC / PV Flow

  ┌───────────────┐     ┌───────────────┐     ┌───────────────┐
  │ StorageClass  │     │     PVC       │     │      PV       │
  │ "fast-ssd"    │     │               │     │ (auto-created)│
  │               │◄────│ storageClass: │     │               │
  │ provisioner:  │     │  fast-ssd     │────►│ capacity: 20Gi│
  │  aws-ebs      │     │               │     │ accessMode:   │
  │               │     │ request: 20Gi │     │  RWO          │
  │ reclaimPolicy:│     │               │     │               │
  │  Delete       │     │               │     │ Backed by:    │
  │               │     │               │     │  AWS EBS gp3  │
  └───────────────┘     └───────────────┘     └───────────────┘
```

---

## Reclaim Policies

What happens to the PV when the PVC is deleted:

| Policy | Behavior | Use Case |
|--------|----------|----------|
| **Retain** | PV is kept, data preserved. Admin must manually reclaim. | Production databases, important data |
| **Delete** | PV and underlying storage are deleted automatically. | Temporary/disposable storage |
| **Recycle** | Data is scrubbed (`rm -rf /data/*`), PV becomes available again. | **Deprecated** — use dynamic provisioning instead |

```
  Reclaim Policies

  PVC deleted:

  Retain:                Delete:               Recycle (deprecated):
  ┌──────┐              ┌──────┐               ┌──────┐
  │  PV  │ remains      │  PV  │ deleted       │  PV  │ data wiped
  │ data │ with data    │      │ along with    │      │ PV reused
  │ safe │              │  💨  │ storage       │ empty│
  └──────┘              └──────┘               └──────┘
  Admin reclaims        Automatic cleanup      rm -rf /data/*
```

---

## PV Lifecycle

```
  PV Status Lifecycle

  Available ───► Bound ───► Released ───► (Retain/Delete)
      │              │           │
      │              │           │
   PV created    PVC binds    PVC deleted
   No PVC yet    to this PV   PV status changes

  Statuses:
  - Available: PV is free, not bound to any PVC
  - Bound: PV is bound to a PVC
  - Released: PVC deleted, PV retains data (Retain policy)
  - Failed: Automatic reclamation failed
```

---

## Volume Binding Modes

StorageClasses support two binding modes:

| Mode | Behavior |
|------|----------|
| **Immediate** | PV is provisioned as soon as PVC is created |
| **WaitForFirstConsumer** | PV is provisioned only when a pod using the PVC is scheduled (allows topology-aware provisioning) |

`WaitForFirstConsumer` is preferred because it provisions storage in the same availability zone as the pod.

---

## Key Exam Points

- **CSI** (Container Storage Interface) is the standard for storage plugins, like CRI for runtimes and CNI for networking.
- **emptyDir** = ephemeral, deleted with the pod. **hostPath** = node-level, not for production.
- **PV** = cluster-scoped storage resource. **PVC** = namespace-scoped user request.
- Kubernetes **binds** a PVC to a matching PV based on capacity, access mode, and StorageClass.
- **StorageClasses** enable dynamic provisioning — no need to pre-create PVs.
- **Access modes**: RWO (one node R/W), ROX (many nodes R), RWX (many nodes R/W).
- **Reclaim policies**: Retain (keep data), Delete (delete everything), Recycle (deprecated).
- **WaitForFirstConsumer** delays provisioning until a pod is scheduled — preferred for topology awareness.

---

## What to Remember for the Exam

1. **CSI** is the storage plugin standard. Know the acronym and what it standardizes.
2. **PV vs PVC**: PV is the actual storage (cluster-scoped). PVC is the request (namespace-scoped). PVC binds to PV.
3. **StorageClass** enables dynamic provisioning. User creates PVC with a storageClassName, Kubernetes auto-creates the PV.
4. **Access modes**: RWO, ROX, RWX. Know what each abbreviation means. RWO is the most common.
5. **Reclaim policies**: Retain (safe, manual cleanup) vs Delete (automatic). Recycle is deprecated.
6. **emptyDir** = temp storage shared between containers in one pod. **hostPath** = mounts from node filesystem.
7. **Volume binding**: WaitForFirstConsumer is preferred over Immediate for topology-aware provisioning.
8. **CSI drivers are external** — they replaced in-tree storage plugins. Examples: AWS EBS CSI, GCE PD CSI, NFS CSI.
