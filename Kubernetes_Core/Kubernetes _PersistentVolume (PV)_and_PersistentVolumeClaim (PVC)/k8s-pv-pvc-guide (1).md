# Kubernetes Persistent Volumes (PV) and Persistent Volume Claims (PVC)

## 📋 Table of Contents
- [Real-World Problem Scenario](#real-world-problem-scenario)
- [What is Kubernetes Persistent Storage?](#what-is-kubernetes-persistent-storage)
- [Understanding PV and PVC](#understanding-pv-and-pvc)
- [Why Do We Need PV and PVC?](#why-do-we-need-pv-and-pvc)
- [How PV and PVC Work](#how-pv-and-pvc-work)
- [Key Concepts](#key-concepts)
- [PV vs PVC - Comparison](#pv-vs-pvc---comparison)
- [Storage Provisioning](#storage-provisioning)
- [Access Modes](#access-modes)
- [Reclaim Policies](#reclaim-policies)
- [Volume Binding Modes](#volume-binding-modes)
- [Real-World Use Cases](#real-world-use-cases)
- [Practical Implementation](#practical-implementation)
- [Troubleshooting Common Issues](#troubleshooting-common-issues)
- [Best Practices](#best-practices)
- [Important Notes](#important-notes)

---

## 🚨 Real-World Problem Scenario

### The Problem
You're running a **PostgreSQL database** in a Kubernetes cluster. Everything works fine until one day:
- A pod crashes due to a node failure
- Kubernetes automatically restarts the pod on a different node
- **All your database data is gone!** 💥

### Why Did This Happen?
By default, **containers are ephemeral** - when a pod dies, all data stored inside it disappears forever. This is catastrophic for stateful applications like databases, message queues, or any application that needs to persist data.

### The Solution
**Persistent Volumes (PV) and Persistent Volume Claims (PVC)** provide durable storage that exists independently of pod lifecycles. Even if your pod crashes, moves to another node, or gets rescheduled, your data remains safe and accessible.

---

## 🔍 What is Kubernetes Persistent Storage?

Kubernetes Persistent Storage is a mechanism that allows **data to survive beyond the lifecycle of individual pods**. It abstracts the underlying storage infrastructure (cloud disks, network storage, local disks) and provides a consistent interface for applications to request and use storage.

### Core Components:
1. **PersistentVolume (PV)** - The actual storage resource
2. **PersistentVolumeClaim (PVC)** - A request for storage
3. **StorageClass** - Defines how storage is provisioned

---

## 📦 Understanding PV and PVC

### PersistentVolume (PV)

A **PersistentVolume** is a piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned using Storage Classes. It represents **actual storage resources** like:
- AWS EBS volumes
- Google Persistent Disks
- Azure Disks
- NFS shares
- Local node storage
- Ceph, GlusterFS, etc.

**Key Characteristics:**
- Cluster-level resource (not namespaced)
- Has a lifecycle independent of any individual pod
- Can be pre-provisioned (static) or dynamically created
- Contains configuration for capacity, access modes, and reclaim policy

### PersistentVolumeClaim (PVC)

A **PersistentVolumeClaim** is a **request for storage by a user/pod**. It's similar to how pods consume node resources (CPU, memory), PVCs consume PV resources.

**Key Characteristics:**
- Namespace-scoped resource
- Specifies storage requirements (size, access mode, storage class)
- Acts as a "voucher" that pods redeem for storage access
- Automatically binds to a matching PV

### The Relationship

Think of it as a **consumer-provider model**:
- **PV** = Available storage (like a parking lot with spaces)
- **PVC** = Storage request (like a parking ticket requesting a space)
- **Pod** = The application that uses the storage (like your car)

```
Pod → mounts → PVC → binds to → PV → backed by → Physical Storage
```

---

## 🤔 Why Do We Need PV and PVC?

### Without PV/PVC (Container Volumes)
- ❌ Data is lost when container crashes
- ❌ Data cannot be shared between pods on different nodes
- ❌ Manual storage management on each node
- ❌ Tight coupling between application and infrastructure

### With PV/PVC
- ✅ **Data persistence** - Survives pod restarts, failures, and rescheduling
- ✅ **Portability** - Abstract storage details from applications
- ✅ **Flexibility** - Easily switch storage backends without changing app code
- ✅ **Automation** - Dynamic provisioning based on demand
- ✅ **Separation of concerns** - Admins manage storage, developers request it

---

## ⚙️ How PV and PVC Work

### The PV-PVC Lifecycle

```
1. PROVISIONING → 2. BINDING → 3. USING → 4. RECLAIMING
```

#### 1️⃣ Provisioning Stage
Storage is made available in the cluster.

**Static Provisioning:**
- Admin manually creates PVs pointing to existing storage
- PVs exist in the cluster waiting to be claimed

**Dynamic Provisioning:**
- PV is automatically created when a PVC is submitted
- Based on StorageClass specifications
- No manual PV creation needed

#### 2️⃣ Binding Stage
- User creates a PVC with storage requirements
- Kubernetes control plane searches for a matching PV
- If found, PVC is **bound** to the PV (1:1 relationship)
- PVC remains `Pending` if no suitable PV exists

**Binding Criteria:**
- Storage capacity (PV must have at least the requested size)
- Access mode compatibility
- Storage class match
- Volume mode match

#### 3️⃣ Using Stage
- Pod references the PVC in its volume specification
- Kubernetes mounts the PV into the pod's filesystem
- Application reads/writes data to the persistent storage
- Data persists even if pod is deleted/recreated

#### 4️⃣ Reclaiming Stage
- When PVC is deleted, PV enters the "Released" state
- What happens next depends on the **Reclaim Policy**:
  - **Retain** - PV kept with data intact (manual cleanup required)
  - **Delete** - PV and underlying storage automatically deleted
  - **Recycle** - PV scrubbed and made available again (deprecated)

---

## 🔑 Key Concepts

### StorageClass

A **StorageClass** provides a way to describe different "classes" of storage. It defines:
- **Provisioner** - Which storage plugin to use (e.g., `ebs.csi.aws.com`)
- **Parameters** - Type of storage (SSD, HDD), IOPS, region, etc.
- **Reclaim Policy** - What happens when PVC is deleted
- **Volume Binding Mode** - When to bind PV to PVC

**Example StorageClass:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iopsPerGB: "50"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### Storage Capacity
- Defined in PV and requested in PVC
- PV capacity must meet or exceed PVC request
- Standard Kubernetes resource notation (Ki, Mi, Gi, Ti)

### Volume Modes
- **Filesystem** - Volume is mounted as a directory (default)
- **Block** - Volume exposed as a raw block device

---

## 📊 PV vs PVC - Comparison

| Aspect | PersistentVolume (PV) | PersistentVolumeClaim (PVC) |
|--------|------------------------|------------------------------|
| **Definition** | Actual storage resource in cluster | Request for storage by a pod |
| **Scope** | Cluster-wide (no namespace) | Namespace-scoped |
| **Created By** | Admin or StorageClass (dynamic) | User/Developer |
| **Represents** | Physical storage (EBS, NFS, etc.) | Storage requirements/needs |
| **Lifecycle** | Independent of pods | Can be tied to pod lifecycle |
| **Relationship** | 1 PV can bind to 1 PVC | 1 PVC binds to 1 PV |
| **Purpose** | Provide storage | Request storage |
| **Analogy** | Hotel room | Room reservation |
| **Configuration** | Capacity, access modes, reclaim policy | Storage request, access modes, storage class |
| **Binding** | Bound TO a PVC | Bound BY a PV |

### Regular Volumes vs Persistent Volumes

| Feature | Volume | PersistentVolume |
|---------|--------|------------------|
| **Lifecycle** | Tied to pod | Independent of pod |
| **Persistence** | Lost when pod terminates | Survives pod termination |
| **Scope** | Pod-level | Cluster-level |
| **Definition** | Defined in pod spec | Separate resource object |
| **Sharing** | Within pod containers only | Across multiple pods (with appropriate access mode) |
| **Management** | Managed with pod | Managed separately |
| **Use Case** | Temporary data, logs | Databases, user uploads, critical data |

---

## 🔄 Storage Provisioning

### Static Provisioning

**When to Use:**
- ✅ Fixed, predictable storage needs
- ✅ Pre-existing storage infrastructure
- ✅ Compliance requirements for specific storage
- ✅ Small-scale deployments

**Workflow:**
1. Admin creates physical storage (e.g., EBS volume)
2. Admin creates PV manifest pointing to that storage
3. User creates PVC requesting storage
4. Kubernetes binds PVC to matching PV

**Example:**
```yaml
# PV (created by admin)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
---
# PVC (created by user)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

### Dynamic Provisioning

**When to Use:**
- ✅ Variable storage requirements
- ✅ Large-scale deployments
- ✅ Self-service storage for developers
- ✅ Cost optimization (provision only when needed)

**Workflow:**
1. Admin creates StorageClass
2. User creates PVC referencing StorageClass
3. Kubernetes automatically provisions PV
4. PV and PVC are automatically bound

**Example:**
```yaml
# StorageClass (created by admin)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
---
# PVC (created by user)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-storage
spec:
  storageClassName: gp3
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

**Benefits:**
- Eliminates manual PV creation
- Reduces admin overhead
- Prevents storage waste
- Enables self-service for developers

---

## 🔐 Access Modes

Access modes define **how a volume can be mounted** by nodes and pods.

### Available Access Modes

| Access Mode | Short Code | Description | Use Cases |
|-------------|-----------|-------------|-----------|
| **ReadWriteOnce** | RWO | Read-write by **single node** | Single-instance databases, stateful apps |
| **ReadOnlyMany** | ROX | Read-only by **multiple nodes** | Shared configuration, static websites, read replicas |
| **ReadWriteMany** | RWX | Read-write by **multiple nodes** | Shared file systems, collaborative apps |
| **ReadWriteOncePod** | RWOP | Read-write by **single pod** (v1.22+) | Applications requiring exclusive pod access |

### Important Notes on Access Modes

⚠️ **Common Misconception:**
- `ReadWriteOnce` does NOT mean "one pod" - it means "one **node**"
- Multiple pods on the **same node** can mount a RWO volume
- If you need single-pod access, use `ReadWriteOncePod` (requires CSI driver support)

⚠️ **Storage Backend Limitations:**
- Not all storage types support all access modes
- Example: AWS EBS only supports RWO
- Example: NFS supports ROX and RWX

### Access Mode Selection Guide

**Choose ReadWriteOnce (RWO) when:**
- Running single-instance databases (MySQL, PostgreSQL)
- Using cloud block storage (EBS, Azure Disk)
- One writer at a time is required

**Choose ReadOnlyMany (ROX) when:**
- Sharing read-only configuration files
- Distributing static content across multiple pods
- Running read replicas that don't write

**Choose ReadWriteMany (RWX) when:**
- Multiple pods need concurrent write access
- Running shared file systems (CMS, shared uploads)
- Using NFS or network file storage

**Choose ReadWriteOncePod (RWOP) when:**
- Strict single-pod access is required
- Data safety guarantees need at most one writer
- Using CSI drivers that support this mode

---

## 🗑️ Reclaim Policies

Reclaim policy defines **what happens to a PV when its PVC is deleted**.

### Policy Options

| Policy | Behavior | Use Case |
|--------|----------|----------|
| **Retain** | PV remains, data kept, manual cleanup needed | Production databases, critical data |
| **Delete** | PV and backing storage automatically deleted | Development, temporary storage, cloud volumes |
| **Recycle** | PV data erased (basic scrub), made available again | ⚠️ **Deprecated** - don't use |

### Retain Policy

**How it Works:**
1. User deletes PVC
2. PV enters "Released" state (not "Available")
3. PV keeps all data intact
4. Admin must manually clean up and make PV available again

**When to Use:**
- ✅ Production databases
- ✅ Sensitive data that requires audit before deletion
- ✅ Data that might need recovery
- ✅ Compliance requirements

**Example:**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: critical-data-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain  # <-- Retain policy
  awsElasticBlockStore:
    volumeID: vol-0a1b2c3d4e5f6g7h8
```

### Delete Policy

**How it Works:**
1. User deletes PVC
2. PV is automatically deleted
3. Underlying storage (EBS volume, disk) is deleted
4. All data is permanently lost

**When to Use:**
- ✅ Development/testing environments
- ✅ Temporary storage
- ✅ Cost optimization (avoid orphaned volumes)
- ✅ Ephemeral data

**Important:** This is the **default** for dynamically provisioned volumes!

### Changing Reclaim Policy

You can change the reclaim policy of an existing PV:

```bash
# Change from Delete to Retain
kubectl patch pv my-pv -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

⚠️ **Note:** You can change policy on PV but NOT on StorageClass (field is immutable)

---

## ⏱️ Volume Binding Modes

Volume binding mode controls **when** PV binding occurs.

### Immediate Mode (Default)

**Behavior:**
- PVC binds to PV as soon as PVC is created
- Volume provisioning happens immediately
- PV is selected before pod is scheduled

**Problems:**
- ❌ PV might be in a different zone than where pod gets scheduled
- ❌ Can lead to pod scheduling failures
- ❌ Wasteful if pod never starts

**When to Use:**
- Storage is zone-agnostic (NFS, cluster-wide storage)
- Single-zone clusters

### WaitForFirstConsumer Mode (Recommended)

**Behavior:**
- PVC remains `Pending` until a pod using it is created
- Volume binding delayed until pod scheduling
- PV is provisioned in the same zone as the pod

**Benefits:**
- ✅ Ensures PV and Pod are co-located
- ✅ Prevents cross-zone mounting failures
- ✅ Better resource utilization
- ✅ Avoids orphaned volumes

**When to Use:**
- ✅ Multi-zone clusters
- ✅ Cloud storage (EBS, GCE PD, Azure Disk)
- ✅ Production deployments

**Example:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/aws-ebs
volumeBindingMode: WaitForFirstConsumer  # <-- Recommended
parameters:
  type: gp3
```

### Comparison

| Aspect | Immediate | WaitForFirstConsumer |
|--------|-----------|---------------------|
| **Binding Time** | When PVC is created | When pod is scheduled |
| **Volume Provisioning** | Immediate | Delayed until pod creation |
| **Zone Awareness** | ❌ No | ✅ Yes |
| **Use Case** | Single-zone, NFS | Multi-zone, cloud storage |
| **PVC Initial Status** | Bound (if PV available) | Pending |

---

## 💼 Real-World Use Cases

### 1. Databases (MySQL, PostgreSQL, MongoDB)

**Problem:** Database pod crashes, all data is lost

**Solution:** Use PVC with RWO access mode

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  storageClassName: gp3
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password"
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
```

### 2. Shared File Storage (CMS, WordPress)

**Problem:** Multiple web server pods need access to shared uploaded files

**Solution:** Use PVC with RWX access mode and NFS

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-shared-pvc
spec:
  storageClassName: nfs-storage
  accessModes:
    - ReadWriteMany  # Multiple pods can write
  resources:
    requests:
      storage: 100Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
spec:
  replicas: 3  # Multiple replicas sharing storage
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:latest
        volumeMounts:
        - name: wp-content
          mountPath: /var/www/html/wp-content/uploads
      volumes:
      - name: wp-content
        persistentVolumeClaim:
          claimName: wordpress-shared-pvc
```

### 3. Configuration Management

**Problem:** Multiple pods need access to shared read-only configuration

**Solution:** Use PVC with ROX access mode

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: config-pvc
spec:
  accessModes:
    - ReadOnlyMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: nfs-storage
```

### 4. Message Queues (RabbitMQ, Kafka)

**Problem:** Message queue data must persist across pod restarts

**Solution:** StatefulSet with volumeClaimTemplates

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
spec:
  serviceName: kafka
  replicas: 3
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:latest
        volumeMounts:
        - name: kafka-data
          mountPath: /var/lib/kafka/data
  volumeClaimTemplates:  # Dynamic PVC creation per replica
  - metadata:
      name: kafka-data
    spec:
      storageClassName: fast-ssd
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 100Gi
```

### 5. Log Aggregation

**Problem:** Need to collect and persist logs from multiple pods

**Solution:** Shared RWX volume for log collectors

### 6. CI/CD Build Caches

**Problem:** Build artifacts and caches lost between CI runs

**Solution:** PV for persistent build cache

---

## 🛠️ Practical Implementation

### Complete Example: WordPress with MySQL

```yaml
# MySQL PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: wordpress
spec:
  storageClassName: gp3
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
# WordPress PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
  namespace: wordpress
spec:
  storageClassName: efs-storage
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
---
# MySQL Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        - name: MYSQL_DATABASE
          value: wordpress
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-data
        persistentVolumeClaim:
          claimName: mysql-pvc
---
# WordPress Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: wordpress
spec:
  replicas: 3
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:latest
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql
        - name: WORDPRESS_DB_NAME
          value: wordpress
        - name: WORDPRESS_DB_USER
          value: root
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 80
        volumeMounts:
        - name: wordpress-data
          mountPath: /var/www/html/wp-content
      volumes:
      - name: wordpress-data
        persistentVolumeClaim:
          claimName: wordpress-pvc
```

### Useful Commands

```bash
# List all PVs in cluster
kubectl get pv

# List all PVCs in namespace
kubectl get pvc -n <namespace>

# Describe PV for details
kubectl describe pv <pv-name>

# Describe PVC for details
kubectl describe pvc <pvc-name> -n <namespace>

# Check PV status
kubectl get pv -o wide

# Check StorageClasses
kubectl get storageclass
kubectl get sc

# Check which pod is using a PVC
kubectl get pods -o=json | jq -r '.items[] | select(.spec.volumes[].persistentVolumeClaim.claimName=="<pvc-name>") | .metadata.name'

# Delete PVC (be careful!)
kubectl delete pvc <pvc-name> -n <namespace>

# Edit PV reclaim policy
kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'

# Expand PVC (if allowVolumeExpansion: true)
kubectl patch pvc <pvc-name> -n <namespace> -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'
```

---

## 🐛 Troubleshooting Common Issues

### 1. PVC Stuck in Pending State

**Symptoms:**
```bash
$ kubectl get pvc
NAME        STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-pvc      Pending                                      gp3            2m
```

**Common Causes & Solutions:**

#### Cause 1: No Matching PV (Static Provisioning)
```bash
# Check if PV exists
kubectl get pv

# Solution: Create matching PV or use dynamic provisioning
```

#### Cause 2: Storage Class Not Found
```bash
# Check if storage class exists
kubectl get sc

# Check PVC for storage class name typos
kubectl describe pvc <pvc-name>

# Solution: Fix storage class name or create it
```

#### Cause 3: WaitForFirstConsumer Mode
```bash
# Check storage class binding mode
kubectl get sc <storage-class-name> -o yaml | grep volumeBindingMode

# This is NORMAL behavior - PVC binds when pod is created
# Solution: Create the pod that uses this PVC
```

#### Cause 4: Insufficient Resources
```bash
# Check events
kubectl describe pvc <pvc-name>

# Look for: "no persistent volumes available for this claim"
# Solution: Reduce storage request or provision more storage
```

#### Cause 5: Node Affinity Mismatch
```bash
# Check PVC events
kubectl describe pvc <pvc-name>

# Solution: Ensure PV's node affinity matches available nodes
```

### 2. Pod Cannot Mount Volume

**Symptoms:**
```
FailedAttachVolume or FailedMount errors
```

**Solutions:**

```bash
# Check pod events
kubectl describe pod <pod-name>

# Common causes:
# - Volume still attached to another node (wait for detachment)
# - Access mode incompatibility
# - File system corruption
# - Storage driver not installed on node
```

### 3. PV Shows "Released" Not "Available"

**Cause:** PVC was deleted but PV has Retain policy

**Solution:**
```bash
# Option 1: Delete and recreate PV (data loss!)
kubectl delete pv <pv-name>

# Option 2: Remove claimRef to make it available again
kubectl patch pv <pv-name> -p '{"spec":{"claimRef": null}}'
```

### 4. Volume is Full

**Symptoms:**
```
"no space left on device" errors in application logs
```

**Solution:**
```bash
# Check PV capacity
kubectl get pv

# Expand PVC (if storage class allows)
kubectl patch pvc <pvc-name> -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'

# Verify expansion
kubectl get pvc <pvc-name>
```

### 5. Cannot Delete PVC

**Symptoms:**
```bash
$ kubectl delete pvc my-pvc
# PVC stuck in "Terminating" state
```

**Cause:** PVC is still being used by a pod

**Solution:**
```bash
# Find which pod is using the PVC
kubectl get pods -o=json | jq '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName=="my-pvc") | .metadata.name'

# Delete the pod first
kubectl delete pod <pod-name>

# Then delete PVC
kubectl delete pvc my-pvc

# If still stuck, remove finalizers (careful!)
kubectl patch pvc <pvc-name> -p '{"metadata":{"finalizers":null}}'
```

### 6. Access Denied / Permission Errors

**Cause:** File system permissions or SELinux

**Solution:**
```yaml
# Use securityContext in pod spec
spec:
  securityContext:
    fsGroup: 1000  # Set GID for volume
  containers:
  - name: app
    securityContext:
      runAsUser: 1000
      runAsGroup: 1000
```

---

## ✅ Best Practices

### 1. Use Dynamic Provisioning
- ✅ Prefer StorageClass and dynamic provisioning over static PVs
- ✅ Reduces administrative overhead
- ✅ Eliminates manual storage allocation

### 2. Set Appropriate Reclaim Policies
- ✅ Production: Use `Retain` for critical data
- ✅ Development: Use `Delete` to avoid storage waste
- ✅ Change policy to Retain before testing deletions

### 3. Choose Correct Access Modes
- ✅ Use `ReadWriteOnce` for single-node storage (most common)
- ✅ Use `ReadWriteMany` only when truly needed (requires compatible storage)
- ✅ Use `ReadOnlyMany` for shared read-only data

### 4. Use WaitForFirstConsumer Binding Mode
- ✅ Ensures PV is created in same zone as pod
- ✅ Prevents cross-zone mounting failures
- ✅ Required for multi-zone clusters with zonal storage

### 5. Implement Storage Quotas
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: dev
spec:
  hard:
    requests.storage: "500Gi"
    persistentvolumeclaims: "10"
```

### 6. Monitor PVC Usage
- ✅ Set up alerts for PVC capacity
- ✅ Monitor disk usage within pods
- ✅ Enable volume expansion when possible

### 7. Backup Strategy
- ✅ Use volume snapshots (VolumeSnapshot API)
- ✅ Implement backup automation (Velero, Stash)
- ✅ Test restore procedures regularly

### 8. Label and Organize PVs/PVCs
```yaml
metadata:
  name: db-pvc
  labels:
    app: mysql
    environment: production
    tier: database
```

### 9. Use StatefulSets for Stateful Apps
- ✅ Automatic PVC creation via volumeClaimTemplates
- ✅ Stable network identities
- ✅ Ordered deployment and scaling

### 10. Encrypt Data at Rest
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: encrypted-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"  # Enable encryption
```

### 11. Set Resource Limits
```yaml
spec:
  containers:
  - name: app
    resources:
      requests:
        ephemeral-storage: "2Gi"
      limits:
        ephemeral-storage: "4Gi"
```

### 12. Avoid HostPath in Production
- ❌ Tight coupling to specific node
- ❌ Data loss if node fails
- ❌ Security concerns
- ✅ Use only for development/testing

### 13. Document Storage Requirements
- ✅ Specify IOPS, throughput requirements
- ✅ Define backup/retention policies
- ✅ Document disaster recovery procedures

### 14. Test Volume Expansion
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable-storage
provisioner: ebs.csi.aws.com
allowVolumeExpansion: true  # Enable expansion
```

### 15. Use CSI Drivers
- ✅ Modern, vendor-neutral interface
- ✅ Better features (snapshots, cloning, expansion)
- ✅ Replaces deprecated in-tree volume plugins

---

## ⚠️ Important Notes

### Security Considerations
- 🔒 Use RBAC to control who can create/delete PVs and PVCs
- 🔒 Encrypt sensitive data at rest using StorageClass parameters
- 🔒 Use Pod Security Standards to restrict hostPath volumes
- 🔒 Implement network policies for storage access control

### Performance Considerations
- ⚡ SSD vs HDD - choose based on IOPS requirements
- ⚡ Provisioned IOPS for predictable performance
- ⚡ Consider throughput limits of storage backend
- ⚡ Local storage offers lowest latency but no HA

### Cost Optimization
- 💰 Delete unused PVCs promptly
- 💰 Use reclaim policy Delete for non-critical data
- 💰 Right-size storage requests
- 💰 Monitor and clean up orphaned volumes
- 💰 Use lifecycle policies for cloud storage

### High Availability
- 🔄 Use replicated storage backends (EBS, GCE PD)
- 🔄 Deploy across multiple zones
- 🔄 Implement backup and disaster recovery
- 🔄 Consider using StatefulSets with replication

### Limitations
- ⚠️ RWO volumes cannot be shared across nodes
- ⚠️ Some storage types don't support RWX
- ⚠️ Volume expansion requires storage class support
- ⚠️ Cannot change access mode after creation
- ⚠️ PV-PVC binding is 1:1 (cannot share one PV with multiple PVCs)

### Migration Notes
- 📦 Moving data between PVs requires manual copy or backup/restore
- 📦 Changing storage class requires new PVC creation
- 📦 PV cannot be moved between clusters (export/import needed)

### Deprecations
- ⛔ In-tree volume plugins are deprecated (use CSI drivers)
- ⛔ Recycle reclaim policy is deprecated
- ⛔ Some older volume types being phased out

### Cloud Provider Specifics

#### AWS EBS
- Only supports ReadWriteOnce
- Zone-specific (use WaitForFirstConsumer)
- Supports gp2, gp3, io1, io2, st1, sc1 types

#### GCE Persistent Disk
- Supports ReadWriteOnce and ReadOnlyMany
- Zone and regional options available
- pd-standard, pd-ssd, pd-balanced types

#### Azure Disk
- Only supports ReadWriteOnce
- Zone-redundant options available
- Standard_LRS, Premium_LRS types

#### NFS
- Supports all access modes
- No zonal restrictions
- Lower performance than block storage

---

## 📚 Quick Reference

### PVC Status States
- `Pending` - Waiting for binding or pod creation
- `Bound` - Successfully bound to a PV
- `Lost` - PV does not exist or is unavailable

### PV Status States
- `Available` - Ready to be claimed
- `Bound` - Bound to a PVC
- `Released` - PVC deleted, data retained
- `Failed` - Automatic reclamation failed

### Common Storage Provisioners
- `kubernetes.io/aws-ebs` - AWS Elastic Block Store
- `kubernetes.io/gce-pd` - Google Compute Engine Persistent Disk
- `kubernetes.io/azure-disk` - Azure Disk
- `kubernetes.io/azure-file` - Azure File
- `kubernetes.io/nfs` - Network File System
- `ebs.csi.aws.com` - AWS EBS CSI Driver (recommended)
- `pd.csi.storage.gke.io` - GCE PD CSI Driver (recommended)

### Storage Class Parameters (AWS EBS)
```yaml
parameters:
  type: gp3               # Volume type
  iops: "3000"           # IOPS
  throughput: "125"      # Throughput in MiB/s
  encrypted: "true"      # Encryption
  kmsKeyId: "arn:..."   # KMS key
```

---

## 🎯 Summary

**Persistent Volumes (PV) and Persistent Volume Claims (PVC)** are fundamental Kubernetes resources for managing durable storage in containerized environments. They provide:

✅ **Data persistence** beyond pod lifecycles
✅ **Abstraction** from underlying storage infrastructure  
✅ **Flexibility** to use various storage backends  
✅ **Automation** through dynamic provisioning  
✅ **Portability** across different environments

By understanding the relationship between PVs, PVCs, and StorageClasses, along with concepts like access modes, reclaim policies, and binding modes, you can effectively design and manage stateful applications in Kubernetes.

Remember: **Pods are ephemeral, but data doesn't have to be!** 🚀

---

## 📖 Additional Resources

- [Official Kubernetes Documentation - Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Official Kubernetes Documentation - Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Official Kubernetes Documentation - Dynamic Volume Provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)
- [CSI Drivers List](https://kubernetes-csi.github.io/docs/drivers.html)

---

**Created for DevOps Engineers | Kubernetes Storage Guide**