# Kubernetes Resource Quota - Complete Guide

## Real-World Problem Scenarios

### Scenario 1: The Resource Hog Team
**Problem:** Your company has multiple teams sharing a single Kubernetes cluster. The DevOps team's project suddenly deploys 100 pods consuming massive amounts of CPU and memory. Within hours, no other team can deploy anything because the cluster is completely saturated. The Data Science team can't run their critical models, and the Frontend team's deployments fail indefinitely.

**Impact:** Cluster paralysis, blocked deployments, service degradation for other teams, emergency incident response required, loss of productivity.

**Solution:** Kubernetes Resource Quota enforces namespace-level limits, automatically preventing teams from exceeding their allocated share of cluster resources.

---

### Scenario 2: Unexpected Cloud Bills
**Problem:** Without resource controls, developers deploy multiple database replicas, test environments with huge memory allocations, and forgot-about batch jobs. Your cloud bill skyrockets unexpectedly. An engineer created 50 PersistentVolumeClaims accidentally, and now you're paying for storage you're not using.

**Impact:** Budget overruns, unexpected costs, financial penalties, loss of control over infrastructure spending.

**Solution:** Resource Quota limits the number of objects (pods, services, PVCs) and total storage, directly controlling infrastructure costs and preventing accidental resource explosion.

---

### Scenario 3: Single Pod Monopoly Crisis
**Problem:** A developer creates one Pod with no resource limits. This Pod greedily consumes all available memory on the node, causing other Pods to be evicted or unable to schedule. Critical applications start failing randomly because resources are being stolen by this uncontrolled Pod.

**Impact:** Unpredictable application behavior, cascading failures, difficult-to-debug issues, reduced cluster stability.

**Solution:** Resource Quota combined with LimitRange enforces that EVERY Pod must specify resource requests and limits, preventing any single workload from monopolizing cluster resources.

---

## What is a Kubernetes Resource Quota?

A **Kubernetes ResourceQuota** is a namespace-level policy that enforces constraints on the aggregate resource consumption and object creation within that namespace. It acts as a "budget" for a namespace, preventing teams from exceeding their fair share of cluster resources.

### Key Characteristics

- **Namespace-scoped:** Each namespace has its own independent quota
- **Admission control:** Enforced at object creation time by the admission controller
- **Aggregate limits:** Limits apply to the total of ALL pods/containers in the namespace
- **Multiple resource types:** Can limit CPU, memory, storage, GPU, and object counts simultaneously
- **Hard constraints:** Requests exceeding quota are rejected immediately
- **Automatic tracking:** Kubernetes automatically tracks usage in real-time

---

## Why Use Kubernetes Resource Quota?

### Multi-Tenant Cluster Fairness
In shared clusters, without quotas, one team's runaway process could starve all other teams of resources. Quotas ensure each team gets guaranteed access to their allocated resources.

### Cost Control
Quotas prevent accidental resource waste—no more forgotten batch jobs, accidentally duplicated deployments, or test environments consuming production resources. Direct cost predictability.

### Cluster Stability
By limiting resource consumption, quotas prevent one namespace from destabilizing the entire cluster and impacting other critical workloads.

### Forced Resource Planning
With quotas in place, developers must think about resource requirements upfront. This naturally leads to better resource estimation and more efficient application design.

### Organizational Boundaries
Quotas reinforce team and project isolation. Each business unit or project gets its own namespace with clearly defined resource limits that match their needs.

### Compliance and Governance
Many enterprise environments require resource constraints and cost controls for compliance, auditing, and governance purposes.

---

## Resource Quota Types Comparison

| Quota Type | Purpose | Measured Unit | Default Unit | Common Values |
|-----------|---------|---------------|--------------|---------------|
| **Compute - CPU Requests** | Minimum CPU guaranteed per namespace | cores | 1 = 1 core | 100m, 500m, 2, 4 |
| **Compute - Memory Requests** | Minimum RAM guaranteed per namespace | bytes | 1Mi = 1 Megabyte | 256Mi, 1Gi, 4Gi |
| **Compute - CPU Limits** | Maximum total CPU usage per namespace | cores | 1 = 1 core | 500m, 2, 4, 8 |
| **Compute - Memory Limits** | Maximum total memory usage per namespace | bytes | 1Mi = 1 Megabyte | 512Mi, 2Gi, 8Gi |
| **Storage - Requests** | Maximum total storage per namespace | bytes | 1Gi = 1 Gigabyte | 50Gi, 100Gi, 500Gi |
| **Storage - PVC Count** | Maximum number of PersistentVolumeClaims | count | 1 | 5, 10, 50 |
| **Object Count - Pods** | Maximum total pods per namespace | count | 1 | 10, 50, 100 |
| **Object Count - Services** | Maximum number of Services | count | 1 | 5, 10, 20 |
| **Object Count - Deployments** | Maximum number of Deployments | count | 1 | 10, 20, 50 |
| **Object Count - ConfigMaps** | Maximum number of ConfigMaps | count | 1 | 25, 50, 100 |
| **Extended Resources** | Custom resources like GPU | count | 1 | 1, 2, 4 |

---

## How Kubernetes Resource Quota Works

### The Admission Control Flow

```
Developer creates Pod
        ↓
API Server receives request
        ↓
Admission Controller intercepts
        ↓
LimitRange applies defaults (if Pod has no requests/limits)
        ↓
ResourceQuota checks namespace usage
        ↓
Does new Pod exceed quota hard limits?
        ↓
    YES → Reject request (HTTP 403 Forbidden)
    NO → Continue
        ↓
Object created and quota usage updated
```

### Usage Tracking

- Kubernetes maintains a **status** field in each ResourceQuota object
- Shows both **hard** limits (defined by admin) and **used** (current consumption)
- Updated automatically as pods are created, deleted, or modified
- Reconciliation runs periodically (every 5 minutes by default) to catch discrepancies

### Mandatory Resource Specifications

**Critical Rule:** If ResourceQuota enforces `requests.cpu` or `requests.memory`, EVERY new Pod in that namespace MUST explicitly specify resource requests/limits for that resource. Pods without these specifications will be rejected.

---

## ResourceQuota vs LimitRange

Both control resources, but at different levels:

| Aspect | ResourceQuota | LimitRange |
|--------|---------------|-----------|
| **Scope** | Entire namespace (aggregate) | Individual Pods/Containers |
| **What it controls** | Total requests/limits for all pods combined | Requests/limits for single pod/container |
| **Enforces minimums** | No | Yes (enforces min resources) |
| **Enforces maximums** | Yes (prevents exceeding quota) | Yes (prevents single pod monopoly) |
| **Provides defaults** | No | Yes (auto-injects defaults) |
| **Purpose** | Budget/fairness across teams | Quality of service per workload |
| **Example limit** | Max 10Gi memory total in namespace | Max 2Gi memory per pod, Min 100Mi |

**Both Should Be Used Together:** ResourceQuota sets the budget, LimitRange prevents individual workloads from breaking that budget.

---

## Quota Scopes

Quota scopes allow targeting quotas to specific Pod types:

| Scope | Applies To | Use Case |
|-------|-----------|----------|
| **BestEffort** | Pods with no requests/limits | Development/testing environments |
| **NotBestEffort** | Pods with at least one request/limit | Production workloads with resource specs |
| **Terminating** | Pods with activeDeadlineSeconds set | Batch jobs, short-lived tasks |
| **NotTerminating** | Pods without activeDeadlineSeconds | Long-running services |
| **PriorityClass** | Pods matching specific priority class | Resource allocation by priority level |

---

## Usage Examples

### Example 1: Basic Compute Resource Quota
**Scenario:** Limit a development team's namespace to fair resource allocation

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-dev-quota
  namespace: team-dev
spec:
  hard:
    requests.cpu: "4"         # Team can request max 4 CPU cores
    requests.memory: 8Gi      # Team can request max 8GB memory
    limits.cpu: "8"           # Total CPU limit for all pods: 8 cores
    limits.memory: 16Gi       # Total memory limit for all pods: 16GB
    pods: "20"                # Maximum 20 pods in namespace
    services: "5"             # Maximum 5 services
```

**Application Impact:**
- If pods in this namespace collectively request 4 cores, no new pod can request CPU
- Memory limits shared: if one pod limits to 10Gi, only 6Gi available for others
- Creating 21st pod will be rejected

---

### Example 2: Storage-Focused Quota
**Scenario:** Limit database team's persistent storage usage

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: database-team-quota
  namespace: databases
spec:
  hard:
    requests.storage: 500Gi           # Total storage allocation
    persistentvolumeclaims: "20"      # Max 20 PVC objects
    gold.storageclass.storage.k8s.io/requests.storage: 300Gi    # SSD tier limit
    bronze.storageclass.storage.k8s.io/requests.storage: 200Gi  # HDD tier limit
```

**How it works:**
- Total storage across all PVCs cannot exceed 500Gi
- SSD (gold) PVCs limited to 300Gi
- HDD (bronze) PVCs limited to 200Gi
- Can have maximum 20 PVC objects total

---

### Example 3: Object Count Quota
**Scenario:** Prevent teams from creating excessive Kubernetes objects

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-quota
  namespace: production
spec:
  hard:
    pods: "100"
    services: "10"
    replicationcontrollers: "20"
    deployments: "50"
    statefulsets: "10"
    configmaps: "100"
    secrets: "50"
    persistentvolumeclaims: "25"
    serviceaccounts: "10"
    networkpolicies: "5"
```

**What this prevents:**
- Creating more than 100 pods (deployment fails beyond this)
- More than 10 Services (LoadBalancer/NodePort creation rejected)
- Runaway ConfigMap creation consuming API resources

---

### Example 4: GPU Resource Quota
**Scenario:** Control expensive GPU resource allocation

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: gpu-quota
  namespace: ml-team
spec:
  hard:
    limits.nvidia.com/gpu: "4"        # Max 4 GPUs total
    pods: "10"                        # Max 10 pods
    requests.cpu: "8"
    requests.memory: 32Gi
```

**Usage:**
```yaml
# Pod requesting GPU
apiVersion: v1
kind: Pod
metadata:
  name: gpu-training-job
spec:
  containers:
  - name: training
    image: ml-training:latest
    resources:
      limits:
        nvidia.com/gpu: "2"  # Uses 2 of the 4 allocated GPUs
```

---

### Example 5: Priority-Based Quota Scope
**Scenario:** Different resource limits for critical vs best-effort workloads

```yaml
# First, create PriorityClasses
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-priority
value: 1000
globalDefault: false
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: false
---
# Quota for critical workloads - generous
apiVersion: v1
kind: ResourceQuota
metadata:
  name: critical-quota
  namespace: production
spec:
  hard:
    pods: "50"
    requests.cpu: "20"
    requests.memory: 40Gi
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["critical-priority"]
---
# Quota for low-priority workloads - strict
apiVersion: v1
kind: ResourceQuota
metadata:
  name: best-effort-quota
  namespace: production
spec:
  hard:
    pods: "10"
    requests.cpu: "2"
    requests.memory: 4Gi
  scopes:
  - BestEffort
```

**Effect:**
- Critical workloads: up to 50 pods, 20 CPU cores, 40GB memory
- Best-effort (no requests/limits): max 10 pods, 2 cores, 4GB memory

---

### Example 6: Combined with LimitRange
**Scenario:** Quota + LimitRange for comprehensive resource governance

```yaml
# First: LimitRange sets per-pod/container defaults and limits
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
  namespace: staging
spec:
  limits:
  # Container-level constraints
  - max:
      cpu: "2"
      memory: "2Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
    default:                  # Auto-applied if not specified
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:          # Auto-applied if not specified
      cpu: "100m"
      memory: "128Mi"
    type: Container
  # Pod-level constraints
  - max:
      cpu: "4"
      memory: "4Gi"
    min:
      cpu: "200m"
      memory: "256Mi"
    type: Pod
---
# Second: ResourceQuota sets namespace budget
apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-quota
  namespace: staging
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "30"
```

**How they work together:**
- LimitRange ensures EVERY pod/container has requests/limits
- ResourceQuota ensures all pods combined don't exceed namespace limits
- Pod without explicit resources gets LimitRange defaults automatically
- Pod trying to exceed LimitRange max is rejected
- If quota is consumed, even valid pods are rejected

---

## Operational Best Practices

### 1. Start with Monitoring, Then Enforcement
❌ **Don't:** Create strict quotas immediately without understanding actual usage  
✅ **Do:** Monitor namespace resource usage for 2-4 weeks, then set quotas at 1.5x peak observed usage

**Implementation:**
```bash
# Monitor CPU/Memory usage over time
kubectl top pods -n <namespace> --containers

# Use Prometheus/Grafana for historical tracking
```

### 2. Always Pair ResourceQuota with LimitRange
Without LimitRange, pods without resource specs can cause quota consumption to succeed but resource allocation to fail. Always ensure every Pod has defaults.

### 3. Implement Tiered Quotas by Environment

| Environment | CPU Request | Memory | Pods | SSD Storage | Purpose |
|------------|-------------|--------|------|------------|---------|
| Production | 20 cores | 40Gi | 100 | 500Gi | Critical workloads |
| Staging | 5 cores | 10Gi | 30 | 100Gi | Pre-production testing |
| Dev | 2 cores | 4Gi | 15 | 50Gi | Active development |
| Test | 1 core | 2Gi | 10 | 20Gi | Experimental features |

### 4. Document Quota Rationale
Add annotations explaining why specific limits were chosen:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: backend-api-quota
  namespace: backend-api
  annotations:
    description: "Backend API team namespace"
    business-unit: "Platform"
    cost-center: "ENG-001"
    created-by: "platform-team"
    last-reviewed: "2025-11-09"
    next-review: "2026-02-09"
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
```

### 5. Build Exception Request Process
Some teams legitimately need temporary quota increases:

- Create documented exception request process
- Require business justification and approval
- Use temporary quota increases (not permanent changes)
- Automatically expire temporary increases
- Track and analyze exceptions for capacity planning

### 6. Monitor Quota Utilization
Regular monitoring prevents quota exhaustion surprises:

```bash
# Check current usage vs limits
kubectl describe resourcequota -n <namespace>

# View usage percentage
kubectl get resourcequota -n <namespace> -o wide

# Alert when approaching limits (80-90%)
```

### 7. Size for Peak, Not Average
Set quotas based on peak usage patterns, not average:
- Add 20% buffer for spike handling
- Account for autoscaling behavior
- Consider batch jobs and periodic tasks
- Regular review and adjustment quarterly

### 8. Use Quota for Cost Allocation
Quotas directly tie to costs—use them for chargeback and cost allocation:

```yaml
annotations:
  cost-center: "marketing"
  budget-per-month: "$5000"
  estimated-usage: "$4200"
```

---

## Troubleshooting Common Issues

### Issue 1: Pod Creation Rejected with Quota Error

**Error Message:**
```
Error from server (Forbidden): error when creating "pod.yaml": 
pods "nginx" is forbidden: exceeded quota: team-quota, requested: 
requests.memory=512Mi, used: requests.memory=8Gi, limited: requests.memory=8Gi
```

**Diagnosis:**
```bash
# Check quota status
kubectl describe resourcequota -n <namespace>

# See what's consuming quota
kubectl top pods -n <namespace>

# List all pods and their resource requests
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].resources.requests.memory}{"\n"}{end}'
```

**Solutions:**
1. Delete unused pods to free up quota
2. Resize existing pods to use less memory
3. Request temporary quota increase
4. Request permanent quota increase if workload legitimately grew

---

### Issue 2: Quota Consumed But Pods Not Created

**Symptom:** `kubectl describe resourcequota` shows resources are used, but Pod creation fails

**Cause:** Failed Pod validations consumed quota before rejection. This is a known Kubernetes behavior that reconciles every 5 minutes.

**Solution:**
```bash
# Wait 5 minutes for reconciliation
# Or manually trigger reconciliation
kubectl delete pod <pod-name> -n <namespace>

# Check if quota was reconciled
kubectl describe resourcequota -n <namespace>
```

---

### Issue 3: Pods Without Requests/Limits Not Rejected

**Symptom:** Pods created without resource specifications despite ResourceQuota

**Cause:** ResourceQuota for CPU/memory set, but LimitRange not configured (only required for cpu/memory quotas)

**Solution:**
```yaml
# Add LimitRange to enforce defaults
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: <namespace>
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    type: Container
```

---

### Issue 4: Storage Quota Not Working for All Storage Classes

**Symptom:** PVC created despite approaching storage quota limit

**Cause:** Not all storage classes covered in quota, or PVC using different storage class than anticipated

**Solution:**
```bash
# Check PVC storage class
kubectl get pvc -n <namespace> -o wide

# Add storage class-specific quota
kubectl get resourcequota -n <namespace> -o yaml
```

Update quota to explicitly cover all storage classes used in namespace.

---

### Issue 5: Extended Resources (GPU) Quota Not Enforced

**Symptom:** Pods requesting GPUs created beyond quota limit

**Cause:** Device plugin not installed, or GPU quota syntax incorrect

**Solution:**
```bash
# Verify GPU resource available on nodes
kubectl describe nodes | grep nvidia.com/gpu

# Correct quota syntax for extended resources
limits.nvidia.com/gpu: "4"  # Correct
requests.nvidia.com/gpu: "4"  # Also correct

# Note: Extended resources use limits and requests differently
```

---

### Issue 6: Can't Delete Namespace Due to Quota

**Symptom:** Namespace stuck in "Terminating" state with ResourceQuota

**Solution:** Rarely, finalizers on ResourceQuota objects cause issues:

```bash
# Check namespace status
kubectl get namespace <namespace> -o yaml

# If stuck, manually remove quota finalizers
kubectl patch resourcequota <quota-name> -n <namespace> \
  -p '{"metadata":{"finalizers":[]}}' --type merge
```

---

## Important Operational Notes

### Quota Hard Enforcement
- **Hard limits** are absolute—requests exceeding them are rejected immediately
- **No soft/warning levels**—it's all or nothing
- **Retroactive deletion** doesn't happen—quota remains until objects are deleted

### Enforcement Timing
- Checked at **resource creation time only**
- Not retroactively enforced on existing resources
- Changes to quota don't affect already-running pods
- Quota controller reconciles every 5 minutes to catch discrepancies

### Resource Request Requirement
When ResourceQuota sets limits for CPU or memory:
- **Every Pod in that namespace must explicitly specify requests or limits** for that resource
- Without requests/limits, Pods are rejected even if quota isn't exceeded
- Use LimitRange to provide defaults for Pods without explicit values

### Multi-Container Pods
Quotas measure **total requests/limits across ALL containers in Pod**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
  - name: app
    resources:
      requests:
        cpu: 200m
        memory: 256Mi
  - name: sidecar
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
  # Total quota consumed: 300m CPU, 384Mi memory
```

### Storage Quota Considerations
- **Requests quota:** Enforces minimum storage allocation at PVC creation
- **Count quota:** Limits number of PVC objects only (not actual usage)
- **Actual used storage:** Not tracked by ResourceQuota, only by storage systems
- **Consider storage class separately:** Can have different quotas per storage class

### Zero-Quota Restrictions
Setting a quota to "0" completely prevents that resource:

```yaml
spec:
  hard:
    pods: "0"  # Prevents ANY pods from being created in namespace
```

Use case: Temporarily disable a namespace without deleting it.

---

## Quick Command Reference

```bash
# Create namespace
kubectl create namespace <namespace>

# Apply ResourceQuota
kubectl apply -f quota.yaml -n <namespace>

# View all ResourceQuotas in namespace
kubectl get resourcequota -n <namespace>

# Get detailed quota information
kubectl describe resourcequota <quota-name> -n <namespace>

# View quota in YAML format
kubectl get resourcequota <quota-name> -n <namespace> -o yaml

# Check quota usage as JSON
kubectl get resourcequota <quota-name> -n <namespace> \
  -o jsonpath='{.status.used}' | jq .

# Delete ResourceQuota
kubectl delete resourcequota <quota-name> -n <namespace>

# Edit existing quota
kubectl edit resourcequota <quota-name> -n <namespace>

# Check what's consuming resources
kubectl top pods -n <namespace> --containers

# Get pods and their resource requests
kubectl get pods -n <namespace> \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].resources.requests.cpu}{"\n"}{end}'

# Apply LimitRange
kubectl apply -f limitrange.yaml -n <namespace>

# View LimitRange
kubectl get limitrange -n <namespace>

# Describe LimitRange
kubectl describe limitrange <range-name> -n <namespace>
```

---

## ResourceQuota Lifecycle

### 1. Planning Phase
- Analyze namespace requirements and workloads
- Calculate peak resource usage
- Plan tiered quotas by environment
- Design exception process

### 2. Implementation Phase
```bash
# Create namespace
kubectl create namespace team-backend

# Create LimitRange first
kubectl apply -f limitrange.yaml -n team-backend

# Create ResourceQuota
kubectl apply -f resourcequota.yaml -n team-backend

# Verify application
kubectl describe resourcequota -n team-backend
```

### 3. Monitoring Phase
- Track quota usage over time
- Alert at 70%, 80%, 90% thresholds
- Monitor quota rejection events
- Collect metrics for capacity planning

### 4. Adjustment Phase
- Quarterly quota reviews
- Adjust based on actual vs. estimated usage
- Handle growth with planned increases
- Document changes and reasons

---

## Common ResourceQuota Patterns

### Development Team Pattern
```yaml
# Low limits, permissive for experimentation
requests.cpu: "2"
requests.memory: 4Gi
pods: "20"
configmaps: "50"
```

### Production Pattern
```yaml
# High limits, tight constraints on object count
requests.cpu: "20"
requests.memory: 40Gi
pods: "100"
services: "10"  # Limit to critical services only
```

### Database Namespace Pattern
```yaml
# Focus on storage and pod count
requests.storage: 1000Gi
persistentvolumeclaims: "50"
pods: "20"
```

### CI/CD Namespace Pattern
```yaml
# High pod count, moderate resources
pods: "200"  # Many pipeline pods
requests.cpu: "10"
requests.memory: 20Gi
```

---

## Key Takeaways

✅ **Use ResourceQuota for fairness** - Prevent any single team from monopolizing cluster resources  
✅ **Always pair with LimitRange** - Ensures pods have resource specifications  
✅ **Start with monitoring** - Understand actual usage before setting quotas  
✅ **Use tiered quotas** - Different limits for dev/staging/production  
✅ **Enable cost tracking** - Link quotas to cost centers and budgets  
✅ **Build exception process** - Allow temporary increases with approval  
✅ **Regular reviews** - Adjust quotas quarterly based on actual usage  
✅ **Alert at thresholds** - Notify teams at 70-90% quota consumption  
✅ **Document rationale** - Record why specific limits were chosen  
✅ **Test before enforcement** - Deploy with high limits first, then tighten gradually  

---

## Integration with Other Kubernetes Features

- **Namespaces** - Quotas apply per namespace
- **RBAC** - Restrict who can create/modify ResourceQuotas
- **PriorityClass** - Different quotas for different priority levels
- **Pod Disruption Budget** - Combine with quota for stable deployments
- **Network Policies** - Orthogonal to resource quotas
- **Monitoring/Metrics** - Track quota usage with Prometheus/Grafana

---

## Additional Resources

- [Official Kubernetes ResourceQuota Documentation](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Kubernetes LimitRange Documentation](https://kubernetes.io/docs/concepts/policy/limit-range/)
- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes Best Practices - Resource Quotas](https://kubernetes.io/docs/concepts/configuration/overview/)

---

**Version:** 1.0  
**Last Updated:** November 2025  
**Maintained by:** DevOps Team