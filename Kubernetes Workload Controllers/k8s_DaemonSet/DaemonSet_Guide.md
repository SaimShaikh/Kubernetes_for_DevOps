# Kubernetes DaemonSet
---

## Real-Time Problem Scenario

### The Problem: Invisible Cluster Issues

**Scenario:** You are managing a 10-node Kubernetes cluster in production. Suddenly:

- One node experiences high memory usage, but you don't know about it until users report slowness
- Security vulnerabilities go undetected on multiple nodes
- Logs from critical pods are not being collected, making debugging impossible
- Network plugins fail on a newly added node, breaking connectivity
- You cannot monitor CPU/memory metrics across all nodes uniformly

**Root Cause:** There's no **consistent mechanism** to run monitoring, logging, and security tools on **every single node** in your cluster.

### How DaemonSet Solves This

A DaemonSet automatically ensures that a pod runs on **every node** (or selected nodes) in your cluster. When you deploy a DaemonSet for:

- **Logging**: Fluentd or Logstash collects logs from every node
- **Monitoring**: Prometheus Node Exporter exposes metrics from every node
- **Security**: Security agents scan every node for vulnerabilities
- **Networking**: CNI plugins establish connectivity on every node

This eliminates blind spots and ensures **cluster-wide observability, security, and infrastructure management**.

---

## What is DaemonSet?

A **DaemonSet** is a Kubernetes workload resource that ensures a **specific pod runs on all (or selected) nodes** in a cluster.

### Key Characteristics:

- **One pod per node**: Unlike Deployments (which can have multiple replicas on a single node), DaemonSets ensure **exactly one pod runs per node**
- **Automatic scaling**: When new nodes join the cluster, DaemonSet automatically schedules a pod on them
- **Automatic cleanup**: When nodes leave the cluster, DaemonSet removes the associated pods
- **Node-level daemons**: Designed for background services that need cluster-wide presence
- **Special tolerations**: DaemonSet pods can tolerate node taints that would normally prevent scheduling

### DaemonSet Architecture:

```
Kubernetes Cluster
├── Node 1 → [DaemonSet Pod 1]
├── Node 2 → [DaemonSet Pod 2]
├── Node 3 → [DaemonSet Pod 3]
├── Node 4 → [DaemonSet Pod 4]
└── Node 5 → [DaemonSet Pod 5]
```

Each node gets exactly one copy of the pod managed by the DaemonSet.

---

## Why DaemonSet?

### The Traditional Problem (Without DaemonSet):

1. **Manual pod management**: You'd need to create and maintain a pod on each node manually
2. **No auto-scaling**: Adding new nodes requires manual pod creation
3. **Inconsistency**: Different nodes might have different versions or configurations
4. **Operational overhead**: Deleting nodes requires manual pod cleanup

### The DaemonSet Solution:

| Problem | Solution |
|---------|----------|
| Manual node-by-node pod creation | DaemonSet automatically schedules pods on all/selected nodes |
| New nodes join without monitoring/logging | DaemonSet automatically adds pod to new nodes |
| Inconsistent configurations across nodes | All pods use the same template specification |
| Manual pod deletion on node removal | DaemonSet automatically removes pods when nodes are deleted |
| No visibility into cluster state | Logging/monitoring agents run everywhere, providing full observability |

---

## How DaemonSet Works?

### Step-by-Step Execution Flow:

#### 1. **Define the DaemonSet**
You write a YAML manifest specifying:
- Pod template (containers, resources, volumes)
- Node selectors or affinity rules (optional, for selective nodes)
- Update strategy (RollingUpdate or OnDelete)

#### 2. **Apply to Cluster**
```bash
kubectl apply -f daemonset.yaml
```

#### 3. **DaemonSet Controller Action**
The DaemonSet controller:
- Scans all nodes in the cluster
- Checks if a pod from this DaemonSet exists on each node
- Creates missing pods on nodes that don't have one
- Schedules pods to nodes that meet selector criteria

#### 4. **Pod Scheduling**
- DaemonSet pods bypass normal scheduling constraints
- They have automatic tolerations for node taints
- They can schedule even if nodes have resource pressure

#### 5. **Monitoring & Maintenance**
- If a DaemonSet pod dies, the controller recreates it
- If a node is cordoned/drained, the pod is terminated gracefully
- If a node recovers, a new pod is scheduled

#### 6. **Rolling Updates**
- Old pods are gradually replaced with new ones
- One pod at a time to maintain service continuity
- Controlled by `maxUnavailable` parameter

---

## DaemonSet vs Other Workload Controllers

### Comparison Table

| Aspect | DaemonSet | Deployment | StatefulSet | Job | CronJob |
|--------|-----------|------------|-------------|-----|---------|
| **Purpose** | Run one pod per node | Run N replicas anywhere | Run stateful apps | Run to completion | Run periodically |
| **Pod Replicas** | 1 per node | Configurable (0-N) | Configurable (ordered) | 1 or more (one-time) | Periodic execution |
| **Pod Identity** | Unique per node | Random/interchangeable | Persistent, ordered | Non-persistent | Non-persistent |
| **Scheduling** | One on each node | Any nodes | Specific ordering | Any available node | Any available node |
| **Use Case** | Monitoring, Logging, Networking | Web apps, APIs, services | Databases, stateful apps | Batch processing | Scheduled tasks |
| **Data Persistence** | No | Optional | Yes (required) | Optional | Optional |
| **Node Tolerance** | Built-in tolerations | Requires manual config | Requires manual config | Requires manual config | Requires manual config |
| **Pod Interchangeability** | Each pod tied to node | Pods are interchangeable | Pods are non-interchangeable | Pods are interchangeable | Pods are interchangeable |
| **Auto-recovery** | Yes | Yes | Yes | Yes (if restarted) | Yes (on schedule) |
| **Scaling** | With cluster nodes | Manual or HPA | Manual or HPA | N/A | N/A |
| **Rolling Updates** | Supported | Supported | Supported | N/A | N/A |

### Visual Comparison:
<img width="1160" height="680" alt="image" src="https://github.com/user-attachments/assets/f15db5c6-da4e-4889-acc1-3b674d3debd6" />


```
Deployment: [Pod A, Pod B, Pod C] on any nodes
DaemonSet: [Pod] on Node1, [Pod] on Node2, [Pod] on Node3

```

---
<img width="2048" height="2048" alt="image" src="https://github.com/user-attachments/assets/6ceb5ea0-0816-4238-8096-c1290194bdbb" />

## When to Use DaemonSet

### ✅ Use DaemonSet When:

1. **Logging & Observability**
   - Running Fluentd, Filebeat, or Logstash on every node
   - Centralized log aggregation from cluster
   - Example: Collecting container logs to ELK stack

2. **Monitoring & Metrics**
   - Prometheus Node Exporter on every node
   - Collecting CPU, memory, disk metrics
   - System-level performance monitoring

3. **Security & Compliance**
   - Running CIS Benchmark scanners (kube-bench) on nodes
   - Intrusion detection systems (IDS)
   - Vulnerability scanners
   - RBAC enforcement tools

4. **Networking & CNI**
   - Deploying network plugins (Calico, Flannel, Weave)
   - Load balancers at node level
   - Service mesh sidecars (Istio, Linkerd)
   - VPN clients for node connectivity

5. **Storage & Distributed Systems**
   - Running storage daemons (Ceph, GlusterFS)
   - Distributed cache layers (Redis, Memcached)
   - Volume provisioners

6. **System-Level Operations**
   - GPU drivers installation
   - Kernel module loading
   - Node configuration agents
   - Device plugin management

7. **Infrastructure Management**
   - Node labeling and annotation tools
   - Cluster auto-healing agents
   - Node resource optimization

### ❌ Do NOT Use DaemonSet When:

- You need **multiple replicas** of a pod on a single node → Use **Deployment**
- You need **stateful storage per pod** with persistent identity → Use **StatefulSet**
- You need **job-based one-time execution** → Use **Job**
- You need **scheduled periodic execution** → Use **CronJob**
- Your pod doesn't need to run on **every node** → Use **Deployment**

---

## Benefits of DaemonSet

### 1. **Universal Coverage**
   - **Guaranteed presence**: Every node gets one copy without manual intervention
   - **No blind spots**: Monitoring, logging, and security reach all nodes
   - **Consistency**: All pods use identical configuration

### 2. **Automatic Scaling with Cluster**
   - **New nodes**: Pods automatically scheduled when nodes join
   - **Node removal**: Pods automatically cleaned up when nodes leave
   - **Zero operational overhead**: No manual pod management needed

### 3. **Built-in Resilience**
   - **Auto-recovery**: Failed pods are automatically recreated
   - **Graceful termination**: Pods evicted cleanly when nodes are drained
   - **Resource awareness**: Can tolerate node resource pressure

### 4. **Special Tolerations**
   DaemonSet pods automatically tolerate:
   - `node.kubernetes.io/not-ready:NoExecute` (unreachable nodes)
   - `node.kubernetes.io/disk-pressure:NoSchedule`
   - `node.kubernetes.io/memory-pressure:NoSchedule`
   - `node.kubernetes.io/unschedulable:NoSchedule`
   - `node.kubernetes.io/network-unavailable:NoSchedule`

   This ensures critical infrastructure services keep running despite node conditions.

### 5. **Cluster-Wide Observability**
   - **Complete visibility**: Logging and monitoring across entire cluster
   - **No gaps**: Every node is monitored and logged
   - **Quick debugging**: Centralized access to all node data

### 6. **Simplified Operations**
   - **One definition**: Single YAML file manages all pod instances
   - **Version control**: Easy to track changes
   - **Easy rollouts**: Rolling updates across all nodes in controlled manner

### 7. **Resource Efficiency**
   - **Predictable resource usage**: You know exactly N pods running (N = number of nodes)
   - **Better planning**: Easy to calculate total resource consumption
   - **Resource isolation**: Each node gets one copy (no oversubscription)

---

## Practical Implementation

### 1. Basic DaemonSet Example (Logging Agent)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-logging
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      # Tolerations to run on all nodes including control plane
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: NoSchedule
      
      # Container specification
      containers:
        - name: fluentd
          image: fluent/fluentd:v1.14-1
          resources:
            limits:
              memory: "200Mi"
              cpu: "100m"
            requests:
              memory: "100Mi"
              cpu: "50m"
          
          # Mount node logs
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: docker-containers
              mountPath: /var/lib/docker/containers
              readOnly: true
      
      # Graceful termination
      terminationGracePeriodSeconds: 30
      
      # Volumes for accessing node logs
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: docker-containers
          hostPath:
            path: /var/lib/docker/containers
```

### 2. DaemonSet with Node Selector (Worker Nodes Only)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-agent
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      # Run only on worker nodes
      nodeSelector:
        node-role.kubernetes.io/worker: "true"
      
      containers:
        - name: node-exporter
          image: prom/node-exporter:latest
          ports:
            - containerPort: 9100
              name: metrics
          resources:
            requests:
              cpu: "100m"
              memory: "50Mi"
            limits:
              cpu: "200m"
              memory: "100Mi"
```

### 3. DaemonSet with Advanced Affinity

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: gpu-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: gpu-monitor
  template:
    metadata:
      labels:
        app: gpu-monitor
    spec:
      # Run only on nodes with GPU
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: accelerator
                    operator: In
                    values: ["gpu"]
      
      containers:
        - name: gpu-monitor
          image: nvidia/gpu-monitoring:latest
          resources:
            requests:
              nvidia.com/gpu: "1"
            limits:
              nvidia.com/gpu: "1"
```

### 4. Creating DaemonSet via kubectl

```bash
# Apply DaemonSet from file
kubectl apply -f daemonset.yaml

# Create DaemonSet from scratch (inline YAML)
kubectl create daemonset my-daemon --image=nginx --dry-run=client -o yaml

# Generate and apply
kubectl create daemonset my-daemon --image=nginx --dry-run=client -o yaml | kubectl apply -f -
```

---

## Advanced Concepts

### 1. Update Strategies

#### RollingUpdate (Default)

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1          # Max pods unavailable during update
      maxSurge: 0                # DaemonSet doesn't use maxSurge
```

**Behavior:**
- Terminates one pod at a time
- Creates new pod with updated image
- Waits for pod to become ready before updating next pod
- Ensures service continuity

```bash
# Trigger rolling update by changing image
kubectl set image daemonset/fluentd-logging fluentd=fluent/fluentd:v1.15 -n kube-system
```

#### OnDelete

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

**Behavior:**
- Does not automatically update pods
- Old pods continue running even after spec changes
- Only creates new pods when old ones are manually deleted
- Useful for critical services where disruption is unacceptable

```bash
# Manual pod deletion triggers update
kubectl delete pod <pod-name> -n <namespace>
```

### 2. Taints and Tolerations

#### DaemonSet Automatic Tolerations

DaemonSet automatically adds these tolerations:

```yaml
tolerations:
  - key: node.kubernetes.io/not-ready
    operator: Exists
    effect: NoExecute
  - key: node.kubernetes.io/unreachable
    operator: Exists
    effect: NoExecute
  - key: node.kubernetes.io/disk-pressure
    operator: Exists
    effect: NoSchedule
  - key: node.kubernetes.io/memory-pressure
    operator: Exists
    effect: NoSchedule
  - key: node.kubernetes.io/pid-pressure
    operator: Exists
    effect: NoSchedule
  - key: node.kubernetes.io/unschedulable
    operator: Exists
    effect: NoSchedule
  - key: node.kubernetes.io/network-unavailable
    operator: Exists
    effect: NoSchedule
```

#### Custom Tolerations

```yaml
spec:
  template:
    spec:
      tolerations:
        # Tolerate node not ready for 600 seconds
        - key: node.kubernetes.io/not-ready
          operator: Exists
          effect: NoExecute
          tolerationSeconds: 600
        
        # Tolerate custom taint
        - key: dedicated
          operator: Equal
          value: logging
          effect: NoSchedule
```

#### Taint Nodes for Selective DaemonSet Deployment

```bash
# Taint node
kubectl taint nodes node-1 dedicated=logging:NoSchedule

# Add toleration to DaemonSet to run only on tainted nodes
spec:
  tolerations:
    - key: dedicated
      operator: Equal
      value: logging
      effect: NoSchedule
```

### 3. Node Selectors and Affinity

#### Simple Node Selector

```yaml
spec:
  template:
    spec:
      nodeSelector:
        disk: ssd          # Run only on nodes labeled with disk=ssd
```

#### Node Affinity

```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          # Hard requirement
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/os
                    operator: In
                    values: ["linux"]
          
          # Soft preference (best effort)
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: node-type
                    operator: In
                    values: ["worker"]
```

---

## Troubleshooting & Debugging

### 1. Check DaemonSet Status

```bash
# Get DaemonSet overview
kubectl get daemonset -n <namespace>
kubectl get daemonset -A                    # All namespaces

# Get detailed information
kubectl describe daemonset <name> -n <namespace>

# Get YAML configuration
kubectl get daemonset <name> -n <namespace> -o yaml

# Watch DaemonSet in real-time
kubectl get daemonset -n <namespace> -w
```

### 2. Check DaemonSet Pods

```bash
# List all pods from DaemonSet
kubectl get pods -l app=<label> -n <namespace> -o wide

# Get pod details
kubectl describe pod <pod-name> -n <namespace>

# Check pod logs
kubectl logs <pod-name> -n <namespace>
kubectl logs -f <pod-name> -n <namespace>        # Follow logs
kubectl logs <pod-name> -n <namespace> --tail=50 # Last 50 lines
kubectl logs -p <pod-name> -n <namespace>        # Previous pod logs

# Check pod events
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

### 3. Common Issues and Solutions

#### Issue 1: Pod Not Running on Some Nodes

**Check:**
```bash
# See which nodes have the pod
kubectl get pods -o wide -l app=<label> -n <namespace>

# Check node taints
kubectl describe node <node-name> | grep -A 5 Taints

# Check if node is cordoned
kubectl get nodes -o wide
```

**Solutions:**
- Add tolerations to DaemonSet if node has taints
- Remove cordoning: `kubectl uncordon <node-name>`
- Ensure node selector matches node labels

#### Issue 2: DaemonSet Pod CrashLoopBackOff

**Check:**
```bash
# Get pod logs
kubectl logs <pod-name> -n <namespace>

# Check previous logs (before crash)
kubectl logs -p <pod-name> -n <namespace>

# Describe pod for error messages
kubectl describe pod <pod-name> -n <namespace>
```

**Common Causes:**
- Image not found or pull error
- Configuration errors in pod spec
- Resource limits exceeded
- Volume mount issues

**Solutions:**
```bash
# Fix image and redeploy
kubectl set image daemonset/<name> <container>=<new-image> -n <namespace>

# Increase resource limits
kubectl patch daemonset <name> -n <namespace> --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits/memory", "value":"500Mi"}]'

# Check resource availability on nodes
kubectl describe node <node-name>
```

#### Issue 3: Insufficient Resources

**Check:**
```bash
# Check node resource allocation
kubectl describe node <node-name>

# Check pod resource requests
kubectl describe pod <pod-name> -n <namespace>

# Check actual resource usage
kubectl top pod <pod-name> -n <namespace>
kubectl top node <node-name>
```

**Solutions:**
- Reduce resource requests/limits in DaemonSet
- Move other workloads off nodes
- Upgrade nodes with more resources
- Use node selectors to avoid resource-constrained nodes

#### Issue 4: Pod Evicted

**Check:**
```bash
# Get pod status
kubectl describe pod <pod-name> -n <namespace>

# Look for "Evicted" status in Events
```

**Common Causes:**
- Node memory/disk pressure
- Pod resource limits exceeded
- Node resource exhaustion

**Solutions:**
```bash
# Clean node disk space
ssh <node>
sudo journalctl --vacuum-time=3d  # Clean logs older than 3 days
sudo docker system prune -a        # Clean unused Docker images

# Delete evicted pods
kubectl get pods -n <namespace> | grep Evicted | awk '{print $1}' | xargs kubectl delete pod -n <namespace>

# Increase node resources or reduce DaemonSet resource requests
```

#### Issue 5: Update Not Progressing

**Check:**
```bash
# Check update strategy
kubectl get daemonset <name> -n <namespace> -o yaml | grep -A 5 updateStrategy

# Check rollout status
kubectl rollout status daemonset/<name> -n <namespace>

# Get update history
kubectl rollout history daemonset/<name> -n <namespace>
```

**Solutions:**
```bash
# Rollback to previous version
kubectl rollout undo daemonset/<name> -n <namespace>

# Increase maxUnavailable for faster updates
kubectl patch daemonset <name> -n <namespace> --type='json' \
  -p='[{"op": "replace", "path": "/spec/updateStrategy/rollingUpdate/maxUnavailable", "value":2}]'
```

---

## Best Practices for DevOps

### 1. Resource Management

```yaml
spec:
  template:
    spec:
      containers:
        - name: app
          resources:
            # Always set requests and limits
            requests:
              cpu: "50m"           # Guaranteed minimum
              memory: "64Mi"
            limits:
              cpu: "200m"          # Maximum allowed
              memory: "200Mi"
```

**Why:** Prevents resource starvation and OOMKill scenarios

### 2. Health Checks

```yaml
spec:
  template:
    spec:
      containers:
        - name: app
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 30
          
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
```

**Why:** Automatically restarts unhealthy pods and prevents traffic to unready pods

### 3. Graceful Shutdown

```yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: app
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 15"]
```

**Why:** Allows pods time to finish requests before termination

### 4. Logging and Monitoring

```yaml
spec:
  template:
    metadata:
      labels:
        monitoring: "true"
    spec:
      containers:
        - name: app
          ports:
            - name: metrics
              containerPort: 9090
```

Add ServiceMonitor or PodMonitor for Prometheus:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: daemonset-metrics
spec:
  selector:
    matchLabels:
      monitoring: "true"
  podMetricsEndpoints:
    - port: metrics
      interval: 30s
```

### 5. Security Considerations

```yaml
spec:
  template:
    spec:
      serviceAccountName: daemonset-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
        - name: app
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
```

**Why:** Implements defense-in-depth and least privilege principles

### 6. Version Control and GitOps

```bash
# Store all DaemonSets in Git
git add daemonsets/
git commit -m "Update Fluentd logging agent to v1.15"
git push

# Use ArgoCD or Flux for auto-sync
kubectl apply -k overlays/production/
```

**Why:** Full audit trail, easy rollbacks, and reproducible deployments

### 7. Namespace Isolation

```bash
# Deploy in specific namespaces
kubectl apply -f daemonset.yaml -n kube-system    # System daemons
kubectl apply -f daemonset.yaml -n monitoring      # Monitoring agents
kubectl apply -f daemonset.yaml -n security        # Security agents
```

**Why:** Better organization and RBAC control

### 8. Testing Before Production

```bash
# Test on subset of nodes first
kubectl label node <test-node> test-daemonset=true

# Deploy DaemonSet with nodeSelector
nodeSelector:
  test-daemonset: "true"

# Monitor and verify

# Remove selector to rollout to all nodes
kubectl patch daemonset <name> --type json -p='[{"op":"remove","path":"/spec/template/spec/nodeSelector"}]'
```

---

## Real-World Use Cases

### 1. Log Aggregation with Fluentd

**Problem:** Logs scattered across multiple nodes, no centralized logging.

**Solution:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      
      containers:
        - name: fluentd
          image: fluent/fluentd-kubernetes-daemonset:v1-debian-elasticsearch
          env:
            - name: FLUENT_ELASTICSEARCH_HOST
              value: "elasticsearch.logging.svc.cluster.local"
            - name: FLUENT_ELASTICSEARCH_PORT
              value: "9200"
            - name: FLUENT_ELASTICSEARCH_SCHEME
              value: "http"
          resources:
            limits:
              memory: "512Mi"
            requests:
              memory: "256Mi"
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: docker-containers
              mountPath: /var/lib/docker/containers
              readOnly: true
      
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: docker-containers
          hostPath:
            path: /var/lib/docker/containers
```

**Benefits:**
- All pod logs collected to ELK stack
- Centralized search and analysis
- Historical log retention

### 2. Cluster Monitoring with Prometheus Node Exporter

**Problem:** No visibility into node-level metrics (CPU, memory, disk, network).

**Solution:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9100"
    spec:
      hostNetwork: true
      hostPID: true
      containers:
        - name: node-exporter
          image: prom/node-exporter:latest
          args:
            - --path.procfs=/host/proc
            - --path.sysfs=/host/sys
          ports:
            - containerPort: 9100
              hostPort: 9100
          volumeMounts:
            - name: proc
              mountPath: /host/proc
            - name: sys
              mountPath: /host/sys
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
```

**Benefits:**
- Real-time metrics from every node
- Prometheus scrapes all nodes automatically
- Enables alerting on node conditions

### 3. Security Scanning with Kube-Bench

**Problem:** Compliance requirements not met, security misconfigurations undetected.

**Solution:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-bench-scanner
  namespace: security
spec:
  selector:
    matchLabels:
      app: kube-bench
  template:
    metadata:
      labels:
        app: kube-bench
    spec:
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      
      containers:
        - name: kube-bench
          image: aquasec/kube-bench:latest
          args:
            - node
            - --json
            - --outputfile=/results/node-benchmark.json
          volumeMounts:
            - name: results
              mountPath: /results
            - name: etc
              mountPath: /etc
              readOnly: true
            - name: lib
              mountPath: /lib
              readOnly: true
          securityContext:
            privileged: true
      
      volumes:
        - name: results
          hostPath:
            path: /var/log/kube-bench
        - name: etc
          hostPath:
            path: /etc
        - name: lib
          hostPath:
            path: /lib
      
      restartPolicy: OnFailure
```

**Benefits:**
- CIS Benchmark compliance checks on every node
- Automated security scanning
- Audit trail for compliance

### 4. Network Plugin Deployment (Calico)

**Problem:** Pods cannot communicate, network policies not enforced.

**Solution:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: calico-node
  namespace: calico-system
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  template:
    metadata:
      labels:
        k8s-app: calico-node
    spec:
      hostNetwork: true
      priorityClassName: system-node-critical
      containers:
        - name: calico-node
          image: calico/node:latest
          env:
            - name: DATASTORE_TYPE
              value: kubernetes
            - name: NODENAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          securityContext:
            privileged: true
          volumeMounts:
            - name: lib-modules
              mountPath: /lib/modules
            - name: var-run-calico
              mountPath: /var/run/calico
      volumes:
        - name: lib-modules
          hostPath:
            path: /lib/modules
        - name: var-run-calico
          hostPath:
            path: /var/run/calico
```

**Benefits:**
- Pod-to-pod networking on all nodes
- Network policies enforced
- CNI integration complete

### 5. Resource Cleanup Daemon

**Problem:** Orphaned volumes, old images consuming disk space.

**Solution:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: disk-cleanup
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: disk-cleanup
  template:
    metadata:
      labels:
        app: disk-cleanup
    spec:
      containers:
        - name: cleanup
          image: alpine:latest
          command:
            - sh
            - -c
            - |
              while true; do
                echo "Cleaning up old images..."
                docker image prune -a -f --filter "until=72h" || true
                echo "Cleaning up orphaned volumes..."
                docker volume prune -f || true
                sleep 86400  # Run daily
              done
          securityContext:
            privileged: true
          volumeMounts:
            - name: docker-socket
              mountPath: /var/run/docker.sock
      volumes:
        - name: docker-socket
          hostPath:
            path: /var/run/docker.sock
```

**Benefits:**
- Automatic disk space cleanup
- Prevents disk pressure issues
- Runs on every node uniformly

---

## Kubectl Commands Cheat Sheet for DaemonSet

```bash
# Creating DaemonSet
kubectl apply -f daemonset.yaml
kubectl create daemonset my-daemon --image=nginx --dry-run=client -o yaml

# Getting DaemonSet Information
kubectl get daemonset                                    # Current namespace
kubectl get daemonset -n <namespace>                     # Specific namespace
kubectl get daemonset -A                                 # All namespaces
kubectl get daemonset <name> -o yaml                     # YAML output
kubectl get daemonset <name> -o wide                     # Detailed view
kubectl describe daemonset <name>                        # Detailed description

# Watching DaemonSet
kubectl get daemonset -w                                 # Watch real-time changes
kubectl get pods -l app=<label> -w                       # Watch related pods

# Editing DaemonSet
kubectl edit daemonset <name>                            # Edit in default editor
kubectl patch daemonset <name> -p '{"spec":{"template":{"spec":{"nodeSelector":{"key":"value"}}}}}'

# Updating DaemonSet
kubectl set image daemonset/<name> <container>=<image>  # Update image
kubectl rollout status daemonset/<name>                  # Check rollout status
kubectl rollout history daemonset/<name>                 # View rollout history
kubectl rollout undo daemonset/<name>                    # Rollback to previous

# Getting Pod Information
kubectl get pods -l app=<label> -o wide                  # List pods with node info
kubectl describe pod <pod-name>                          # Pod details
kubectl logs <pod-name>                                  # Pod logs
kubectl logs -f <pod-name>                               # Follow logs
kubectl logs -p <pod-name>                               # Previous pod logs
kubectl exec -it <pod-name> -- /bin/sh                   # Execute command in pod

# Deleting DaemonSet
kubectl delete daemonset <name>                          # Delete DaemonSet
kubectl delete daemonset <name> --cascade=orphan         # Delete without deleting pods

# Advanced Debugging
kubectl get events -n <namespace> --sort-by='.lastTimestamp'  # View events
kubectl top node                                         # Node resource usage
kubectl top pod -l app=<label>                          # Pod resource usage
kubectl get daemonset <name> -o json | jq '.status'     # JSON status
```

---

## Summary: When & How to Deploy DaemonSet

| Scenario | Action | Example |
|----------|--------|---------|
| Need logs from **every node** | Deploy Fluentd DaemonSet | `fluentd-logging` |
| Monitor **node metrics** | Deploy Prometheus Node Exporter | `node-exporter` |
| Run **security agent everywhere** | Deploy security DaemonSet | `falco`, `kube-bench` |
| Deploy **network plugin** | Deploy CNI DaemonSet | `calico-node` |
| Run **GPU drivers** on GPU nodes | Deploy with affinity | `nvidia-gpu-driver` |
| Horizontal scaling **without node constraint** | Use **Deployment** instead | `web-app` |
| Stateful app with **persistent identity** | Use **StatefulSet** instead | `database` |

---

