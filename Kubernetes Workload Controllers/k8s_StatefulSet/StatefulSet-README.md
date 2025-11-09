# StatefulSet in Kubernetes

## Table of Contents
- [What is a StatefulSet?](#what-is-a-statefulset)
- [Why Use StatefulSet?](#why-use-statefulset)
- [How StatefulSet Works](#how-statefulset-works)
- [Core Concepts](#core-concepts)
- [StatefulSet vs Other Workloads](#statefulset-vs-other-workloads)
- [Real-World Problem Scenarios](#real-world-problem-scenarios)
- [StatefulSet Manifest Structure](#statefulset-manifest-structure)
- [Practical Examples](#practical-examples)
- [Important Operations](#important-operations)
- [Limitations & Considerations](#limitations--considerations)
- [Best Practices](#best-practices)

---

## What is a StatefulSet?

A **StatefulSet** is a Kubernetes workload API object used to manage **stateful applications** that require stable network identities, persistent storage, and ordered deployment/scaling. Unlike Deployments (which create interchangeable pods), StatefulSets maintain a **sticky identity** for each pod throughout its lifecycle.

Each pod in a StatefulSet gets:
- A unique, persistent identifier (e.g., `mysql-0`, `mysql-1`, `mysql-2`)
- A stable hostname and DNS record
- Dedicated persistent storage that follows the pod across restarts
- Predictable ordering during creation and deletion

---

## Why Use StatefulSet?

### Problem Statement: Why Not Just Use Deployments?

Imagine you're deploying a **MySQL database cluster** with Deployments:

```yaml
# ❌ Using Deployment (Wrong Approach)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 3
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
```

**Problems that occur:**
1. **Pod names are random** → `mysql-abc123def`, `mysql-xyz789uvw` (makes identification difficult)
2. **Pods restart with different hostnames** → Database replication breaks because it can't find expected nodes
3. **All replicas start at once** → Primary database and replicas start simultaneously without proper synchronization
4. **Storage gets lost** → When a pod crashes, a new pod is created but may not get the same storage, losing data
5. **Random scheduling** → No guarantee which pod serves as primary vs. replica

### Solution: Use StatefulSet

```yaml
# ✅ Using StatefulSet (Correct Approach)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-svc
  replicas: 3
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
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 10Gi
```

**Benefits:**
1. **Predictable pod names** → `mysql-0`, `mysql-1`, `mysql-2` (always the same)
2. **Stable hostnames** → Pods always get DNS names like `mysql-0.mysql-svc.default.svc.cluster.local`
3. **Ordered startup** → Primary starts first, replicas follow and can replicate data properly
4. **Persistent storage** → Each pod always reconnects to its original storage
5. **Reliable discovery** → Other services can always find and connect to specific replicas

---

## How StatefulSet Works

### Pod Lifecycle & Ordering

StatefulSets guarantee **ordered creation and deletion**:

#### 🟢 **Scaling Up (Creating Pods)**
```
Desired: 3 replicas
Step 1: Create mysql-0 → Wait for Running & Ready ✓
Step 2: Create mysql-1 → Wait for Running & Ready ✓
Step 3: Create mysql-2 → Wait for Running & Ready ✓
```

**Why?** Ensures primary database is ready before replicas attempt to replicate data.

#### 🔴 **Scaling Down (Deleting Pods)**
```
Current: 3 replicas → Scale to 1 replica
Step 1: Delete mysql-2 first → Wait for termination ✓
Step 2: Delete mysql-1 → Wait for termination ✓
Step 3: mysql-0 remains (primary)
```

**Why?** Graceful shutdown in reverse order prevents data corruption.

### Pod Identity Components

Each pod in a StatefulSet has three identity components:

| Component | Example | Purpose |
|-----------|---------|---------|
| **Ordinal** | 0, 1, 2 | Unique index within the StatefulSet |
| **Hostname** | mysql-0 | Stable DNS name for the pod |
| **DNS FQDN** | mysql-0.mysql-svc.default.svc.cluster.local | Complete resolvable address |

---

## Core Concepts

### 1. Headless Service (Required)

StatefulSets **require** a Headless Service to manage network identity. A headless service has no cluster IP and returns individual pod IPs instead.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-svc
spec:
  clusterIP: None  # ← Makes it a headless service
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
    name: mysql
```

**What's the difference?**

| Aspect | Regular Service | Headless Service |
|--------|-----------------|------------------|
| **Cluster IP** | Gets a stable IP | None (clusterIP: None) |
| **DNS Resolution** | Returns service IP | Returns all pod IPs |
| **Use Case** | Load balancing | Pod discovery & direct access |
| **StatefulSet** | ❌ Not suitable | ✅ Required |

### 2. Persistent Storage with VolumeClaimTemplates

StatefulSets use `volumeClaimTemplates` to automatically create unique PersistentVolumeClaims (PVCs) for each pod.

```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: ["ReadWriteOnce"]
    storageClassName: fast-ssd
    resources:
      requests:
        storage: 10Gi
```

**Behavior:**
- For each pod created (e.g., `mysql-0`, `mysql-1`), a PVC is automatically created (e.g., `data-mysql-0`, `data-mysql-1`)
- Each PVC binds to its own PersistentVolume
- Pod always reconnects to its original storage, even after restarts
- **⚠️ Important:** Deleting the StatefulSet does NOT delete the PVCs (prevents data loss)

### 3. Update Strategies

StatefulSets support two update strategies:

#### **RollingUpdate (Default)**
Automatically updates pods one at a time, in reverse ordinal order.

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2  # Only update pods with ordinal >= 2
```

**Example:**
```
Current image: mysql:5.7
New image: mysql:8.0

Update order:
Step 1: Delete mysql-2 → Create mysql-2 with new image ✓
Step 2: Delete mysql-1 → Create mysql-1 with new image ✓
Step 3: Delete mysql-0 → Create mysql-0 with new image ✓
```

**Partition:** Update only specific pod ordinals (useful for canary deployments)

#### **OnDelete**
Pods are updated only when manually deleted.

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

---

## StatefulSet vs Other Workloads

| Feature | Deployment | DaemonSet | StatefulSet |
|---------|-----------|-----------|-------------|
| **Pod Identity** | Random, interchangeable | Random | Stable, unique (mysql-0, mysql-1) |
| **Scaling** | All pods start together | One per node | Sequential, ordered |
| **Storage** | Shared or no persistent storage | Shared or no persistent storage | Each pod gets unique PVC |
| **Hostname** | Random each restart | Random each restart | Consistent hostname |
| **DNS Discovery** | Via cluster IP | Via cluster IP | Direct pod DNS names |
| **Use Case** | Stateless apps (web servers) | Node-level services (logging, monitoring) | Stateful apps (databases, queues) |
| **Headless Service** | Not required | Not required | ✅ Required |

---

## Real-World Problem Scenarios

### Scenario 1: Deploying a MySQL Replication Cluster

**Problem:** You need MySQL primary-replica replication, but Deployment creates unpredictable pod names and doesn't guarantee ordered startup.

**Solution with StatefulSet:**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: default
spec:
  serviceName: mysql-svc      # Must reference headless service
  replicas: 3                 # 1 primary + 2 replicas
  podManagementPolicy: OrderedReady  # Sequential pod creation
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping", "-uroot", "-p$(MYSQL_ROOT_PASSWORD)"]
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command: ["mysqladmin", "ping", "-uroot", "-p$(MYSQL_ROOT_PASSWORD)"]
          initialDelaySeconds: 5
          periodSeconds: 10
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: mysql-storage
      resources:
        requests:
          storage: 20Gi
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-svc
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
    name: mysql
```

**Result:**
- `mysql-0` → Primary (starts first, accepts writes)
- `mysql-1` → Replica (waits for mysql-0 to be ready, replicates from mysql-0)
- `mysql-2` → Replica (waits for mysql-1 to be ready, replicates from mysql-0)

---

### Scenario 2: Deploying RabbitMQ Cluster

**Problem:** RabbitMQ nodes need to discover and join each other in a specific order. Random pod names cause node discovery to fail.

**Solution with StatefulSet:**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
spec:
  serviceName: rabbitmq-svc
  replicas: 3
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      containers:
      - name: rabbitmq
        image: rabbitmq:3-management
        env:
        - name: RABBITMQ_NODENAME
          value: rabbit@$(HOSTNAME).rabbitmq-svc.default.svc.cluster.local
        - name: RABBITMQ_ERLANG_COOKIE
          value: "secret-cookie"
        ports:
        - containerPort: 5672
          name: amqp
        - containerPort: 15672
          name: management
        volumeMounts:
        - name: data
          mountPath: /var/lib/rabbitmq
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
---
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-svc
spec:
  clusterIP: None
  selector:
    app: rabbitmq
  ports:
  - port: 5672
    name: amqp
  - port: 15672
    name: management
```

**Pod Names & Hostnames:**
- `rabbitmq-0.rabbitmq-svc.default.svc.cluster.local`
- `rabbitmq-1.rabbitmq-svc.default.svc.cluster.local`
- `rabbitmq-2.rabbitmq-svc.default.svc.cluster.local`

RabbitMQ nodes can reliably discover and join each other using these stable hostnames.

---

### Scenario 3: Deploying etcd Cluster (Consensus-Based)

**Problem:** etcd requires consistent identity and ordered startup to properly elect a leader.

**Solution with StatefulSet:**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: etcd
spec:
  serviceName: etcd-svc
  replicas: 3
  selector:
    matchLabels:
      app: etcd
  template:
    metadata:
      labels:
        app: etcd
    spec:
      containers:
      - name: etcd
        image: quay.io/coreos/etcd:v3.5.0
        env:
        - name: ETCD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: ETCD_INITIAL_CLUSTER
          value: "etcd-0=http://etcd-0.etcd-svc:2380,etcd-1=http://etcd-1.etcd-svc:2380,etcd-2=http://etcd-2.etcd-svc:2380"
        ports:
        - containerPort: 2379
          name: client
        - containerPort: 2380
          name: peer
        volumeMounts:
        - name: data
          mountPath: /etcd-data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: etcd-svc
spec:
  clusterIP: None
  selector:
    app: etcd
  ports:
  - port: 2379
    name: client
  - port: 2380
    name: peer
```

---

## StatefulSet Manifest Structure

### Complete Manifest Breakdown

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: myapp                    # StatefulSet name (used in pod names)
  namespace: default
spec:
  # Service Configuration
  serviceName: myapp-svc         # ⚠️ REQUIRED: Must match headless service name
  
  # Replica Configuration
  replicas: 3                    # Number of pod replicas
  
  # Pod Management
  podManagementPolicy: OrderedReady  # Options: OrderedReady (default), Parallel
  
  # Pod Selection
  selector:
    matchLabels:
      app: myapp                 # Must match template labels
  
  # Pod Template
  template:
    metadata:
      labels:
        app: myapp               # Must match selector
    spec:
      terminationGracePeriodSeconds: 30  # Allow graceful shutdown
      containers:
      - name: myapp-container
        image: myapp:1.0
        ports:
        - containerPort: 8080
          name: http
        volumeMounts:
        - name: data
          mountPath: /data
  
  # Update Strategy
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0               # Update all pods (0 means all)
  
  # Volume Claim Templates (for persistent storage)
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 1Gi
  
  # PVC Retention Policy (when to delete PVCs)
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain            # Keep PVCs when StatefulSet deleted
    whenScaled: Retain             # Keep PVCs when scaled down
```

### Key Fields Explained

| Field | Purpose | Example |
|-------|---------|---------|
| `serviceName` | Headless service for network identity | Must match service metadata.name |
| `replicas` | Number of pods | 3 for mysql-0, mysql-1, mysql-2 |
| `podManagementPolicy` | Creation order (OrderedReady = sequential, Parallel = all at once) | OrderedReady (default) |
| `selector.matchLabels` | Labels to identify pods | app: mysql |
| `template.labels` | Labels for pods (must match selector) | app: mysql |
| `volumeClaimTemplates` | Auto-create PVCs for each pod | Defines storage per pod |
| `updateStrategy.type` | How to roll out updates | RollingUpdate (default) or OnDelete |
| `persistentVolumeClaimRetentionPolicy` | PVC cleanup behavior | Retain (safe, default) or Delete |

---

## Practical Examples

### Example 1: Simple Stateless-like StatefulSet (Learning Purpose)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  clusterIP: None
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx
spec:
  serviceName: nginx-svc
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

**Result:**
```
$ kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
nginx-0   1/1     Running   0          10s
nginx-1   1/1     Running   0          5s
nginx-2   1/1     Running   0          1s
```

---

### Example 2: StatefulSet with Init Container (Database Initialization)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-svc
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      # Init container runs before main container
      initContainers:
      - name: init-setup
        image: postgres:14
        command: ['sh', '-c', 'if [ "$HOSTNAME" = "postgres-0" ]; then echo "Setting up primary..."; fi']
      
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
```

---

## Important Operations

### Create a StatefulSet

```bash
# Apply manifest
kubectl apply -f statefulset.yaml

# Watch pods being created sequentially
kubectl get pods -w
```

### Check StatefulSet Status

```bash
# List StatefulSets
kubectl get statefulsets
kubectl get sts

# Detailed information
kubectl describe sts mysql

# Watch events
kubectl describe sts mysql --show-events
```

### Scale a StatefulSet

```bash
# Scale up (adds new pods)
kubectl scale sts mysql --replicas=5

# Scale down (removes pods in reverse order)
kubectl scale sts mysql --replicas=2

# Watch the process
kubectl get pods -w
```

### Update a StatefulSet

```bash
# Option 1: Apply new manifest
kubectl apply -f updated-statefulset.yaml

# Option 2: Edit directly
kubectl edit sts mysql

# Option 3: Patch specific field
kubectl patch sts mysql -p '{"spec":{"template":{"spec":{"containers":[{"name":"mysql","image":"mysql:8.0"}]}}}}'

# Check update progress
kubectl get pods -w
```

### Perform Rolling Update with Partition

```bash
# Update only pods with ordinal >= 2 (test before full rollout)
kubectl patch sts mysql -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":2}}}}'

# Update template
kubectl patch sts mysql -p '{"spec":{"template":{"spec":{"containers":[{"name":"mysql","image":"mysql:8.1"}]}}}}'

# Later, complete the update
kubectl patch sts mysql -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":0}}}}'
```

### Delete a StatefulSet

```bash
# Delete StatefulSet but keep pods
kubectl delete sts mysql --cascade=false

# Delete StatefulSet with all pods
kubectl delete sts mysql

# Delete StatefulSet and wait for graceful termination
kubectl delete sts mysql --grace-period=120
```

### Check Persistent Volume Claims

```bash
# List PVCs created by StatefulSet
kubectl get pvc

# Describe a specific PVC
kubectl describe pvc data-mysql-0

# PVCs are NOT deleted when StatefulSet is deleted (data safety)
```

### Access Individual Pods

```bash
# Connect to specific pod
kubectl exec -it mysql-0 -- mysql -u root -p

# Check pod hostname inside container
kubectl exec mysql-0 -- hostname

# Check DNS resolution
kubectl exec mysql-0 -- nslookup mysql-0.mysql-svc.default.svc.cluster.local
```

---

## Limitations & Considerations

### 1. Requires Headless Service
StatefulSets require a Headless Service (clusterIP: None) for network identity. Without it, StatefulSet creation fails.

```yaml
# ❌ This won't work (has clusterIP)
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: 10.0.0.5       # ← Problem: not headless
  selector:
    app: mysql
```

### 2. Sequential Pod Creation is Slow
Pods are created one at a time. For large StatefulSets, startup is slower than Deployments.

**Workaround:** Use `podManagementPolicy: Parallel` (but loses ordering guarantees)

```yaml
spec:
  podManagementPolicy: Parallel  # Creates all pods simultaneously
```

### 3. PVCs Are Not Automatically Deleted
Deleting or scaling down a StatefulSet does NOT delete associated PVCs (intentional, for data safety).

```bash
# StatefulSet deleted but PVCs remain
$ kubectl delete sts mysql
$ kubectl get pvc
# Output: data-mysql-0, data-mysql-1, data-mysql-2 still exist!

# You must manually delete PVCs if needed
kubectl delete pvc data-mysql-0 data-mysql-1 data-mysql-2
```

**Control PVC deletion behavior:**

```yaml
persistentVolumeClaimRetentionPolicy:
  whenDeleted: Delete         # Delete PVCs when StatefulSet deleted
  whenScaled: Delete          # Delete PVCs when pods scaled down
```

### 4. No Guaranteed Ordered Pod Termination
StatefulSet does NOT guarantee ordered termination when the StatefulSet itself is deleted. To ensure graceful shutdown:

```bash
# Scale to 0 before deletion (forces ordered termination)
kubectl scale sts mysql --replicas=0
# Wait for termination
kubectl delete sts mysql
```

### 5. Broken Update States Require Manual Intervention
If a pod fails to reach Ready state during a rolling update, the StatefulSet stops and waits forever. You must manually fix the pod.

```bash
# Check pod status
kubectl get pods mysql-0

# If stuck in update, check logs
kubectl logs mysql-0

# After fixing, delete the pod to restart
kubectl delete pod mysql-0
```

### 6. Storage Class Must Support Required Access Mode
StatefulSet uses `accessModes: ["ReadWriteOnce"]` by default, meaning each pod must have exclusive storage.

```yaml
# ❌ Won't work for most storage classes
accessModes: ["ReadWriteMany"]  # Multiple pods can't share with ReadWriteOnce
```

### 7. Requires Manual Persistence Configuration
StatefulSet does NOT handle data backup or high availability automatically. You must implement:
- Database replication setup
- Backup strategies
- Failover logic

---

## Best Practices

### 1. Always Use a Headless Service

```yaml
# ✅ Correct
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  clusterIP: None              # Headless service
  selector:
    app: myapp
  ports:
  - port: 8080
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: myapp
spec:
  serviceName: myapp-svc       # Reference the headless service
```

### 2. Set Appropriate Termination Grace Period

```yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 30  # Allow graceful shutdown
      # Recommended values:
      # - 10-30 seconds for light apps
      # - 30-60 seconds for databases
      # - 60+ seconds for distributed systems
```

### 3. Implement Health Checks

```yaml
containers:
- name: myapp
  livenessProbe:
    httpGet:
      path: /health
      port: 8080
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 5
  readinessProbe:
    httpGet:
      path: /ready
      port: 8080
    initialDelaySeconds: 10
    periodSeconds: 5
```

### 4. Manage PVC Retention Properly

```yaml
persistentVolumeClaimRetentionPolicy:
  whenDeleted: Retain       # ✅ Keep data when StatefulSet deleted
  whenScaled: Retain        # Keep data when scaled down
```

### 5. Use Partition for Canary Updates

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2           # Update only mysql-2 first
                            # Test new version
                            # If good, set partition: 0 to complete update
```

### 6. Set Resource Requests and Limits

```yaml
containers:
- name: myapp
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
```

### 7. Use StorageClass for Automatic PV Provisioning

```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    storageClassName: fast-ssd  # ✅ Use StorageClass
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 10Gi
```

### 8. Label Your Pods Appropriately

```yaml
template:
  metadata:
    labels:
      app: myapp
      version: v1
      tier: database           # Use additional labels for queries
```

### 9. Document Pod Initialization Order

```yaml
# Include comments explaining startup sequence
spec:
  # Pod creation order (strictly sequential):
  # 1. postgres-0 (primary) - initializes database
  # 2. postgres-1 (replica) - waits for primary, starts replication
  # 3. postgres-2 (replica) - waits for postgres-1, starts replication
```

### 10. Scale Down Safely

```bash
# ❌ Never immediately delete
kubectl delete sts myapp

# ✅ Do this instead
# 1. Scale to 0 for graceful termination
kubectl scale sts myapp --replicas=0
# 2. Wait for pods to terminate
kubectl get pods -w
# 3. Then delete StatefulSet
kubectl delete sts myapp
# 4. Manually delete PVCs if no longer needed
kubectl delete pvc -l app=myapp
```

---

## Common Troubleshooting

### Problem: Pod Stuck in Pending

```bash
kubectl describe pod mysql-0
# Check for:
# - PVC not bound (storage class issue)
# - Node selector constraints
# - Insufficient resources

# Solution:
kubectl describe pvc data-mysql-0
```

### Problem: Pod Stuck in CrashLoopBackOff

```bash
# Check logs
kubectl logs mysql-0

# Check previous logs if available
kubectl logs mysql-0 --previous

# Delete and let StatefulSet recreate
kubectl delete pod mysql-0
```

### Problem: DNS Name Not Resolving

```bash
# Test DNS from within cluster
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup mysql-0.mysql-svc.default.svc.cluster.local

# Verify headless service is correctly configured
kubectl get svc mysql-svc -o yaml
# Ensure: clusterIP: None
```

### Problem: PVC Not Deleted After StatefulSet Deletion

```bash
# Expected behavior - PVCs are retained for data safety
kubectl get pvc

# Manual cleanup if needed
kubectl delete pvc data-mysql-0
```

---

## Summary

StatefulSets are essential for running **stateful applications** in Kubernetes. They provide:
- **Stable network identities** for reliable pod discovery
- **Persistent storage** that follows pods across restarts
- **Ordered deployment** ensuring proper initialization
- **Graceful management** of database clusters and distributed systems

Use StatefulSet when your application requires:
1. Unique, persistent pod identities
2. Stable storage per pod
3. Ordered startup/shutdown
4. Direct pod-to-pod communication

Avoid StatefulSet for:
1. Stateless web applications (use Deployment)
2. Node-level services (use DaemonSet)
3. One-off jobs (use Job)

**Remember:** StatefulSet is powerful but adds complexity. Always use it intentionally, with proper understanding of your application's state management requirements.
