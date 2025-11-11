# Kubernetes Taints and Tolerations

## What Are Taints and Tolerations?

**Taints** are marks placed on nodes that repel (push away) pods. Think of them as "locks" on nodes.

**Tolerations** are permissions added to pods that allow them to run on tainted nodes. Think of them as "keys" to unlock those locks.

Together, they give you fine-grained control over which pods can run on which nodes.

---

## The Problem They Solve

Imagine you have:
- **Node A:** Expensive GPU hardware (only for ML workloads)
- **Node B:** General purpose node (for any workload)

Without taints and tolerations, any pod could be scheduled on Node A, wasting expensive GPU resources.

**Solution:** Add a taint to Node A saying "GPU only" and add a toleration to ML pods saying "I can run on GPU nodes."

---

## How Taints and Tolerations Work

```
Node A (Tainted: gpu=true:NoSchedule)
    ↓
Pod without toleration → ❌ REJECTED (won't schedule)
Pod with matching toleration → ✅ ACCEPTED (can schedule)
```

---

## Taint Structure

A taint has three components:

```
key=value:effect
```

| Component | What It Is | Example |
|-----------|-----------|---------|
| **key** | The taint identifier | `gpu`, `environment`, `workload` |
| **value** | The taint value | `true`, `nvidia`, `production` |
| **effect** | What happens to non-tolerating pods | `NoSchedule`, `PreferNoSchedule`, `NoExecute` |

---

## Three Taint Effects

### 1. **NoSchedule** (Strict)

**What it does:** New pods without matching toleration will NOT be scheduled on this node.

**What happens to existing pods:** Nothing (they stay running).

**When to use:** When you want to strictly reserve nodes for specific workloads.

**Example:** Reserve nodes for GPU workloads only.

```bash
kubectl taint nodes gpu-node gpu=true:NoSchedule
```

---

### 2. **PreferNoSchedule** (Soft/Preference)

**What it does:** Kubernetes will TRY to avoid scheduling pods without matching toleration, but will schedule them if necessary (e.g., cluster is full).

**What happens to existing pods:** Nothing.

**When to use:** When you prefer certain pods on certain nodes but it's not critical.

**Example:** Prefer to use spot instances but can fall back to regular instances if needed.

```bash
kubectl taint nodes spot-node spot=true:PreferNoSchedule
```

---

### 3. **NoExecute** (Aggressive)

**What it does:** New pods without toleration won't be scheduled AND existing pods without toleration will be evicted immediately.

**What happens to existing pods:** They get kicked off the node.

**When to use:** For maintenance, node removal, or when a node is degraded.

**Example:** Drain a node for maintenance or when it's running out of memory.

```bash
kubectl taint nodes maintenance-node maintenance=true:NoExecute
```

---

## Quick Comparison Table

| Effect | New Pods | Existing Pods | Strictness |
|--------|----------|---------------|-----------|
| **NoSchedule** | Blocked | Unaffected | Strict |
| **PreferNoSchedule** | Avoided (soft) | Unaffected | Loose |
| **NoExecute** | Blocked | Evicted | Very Strict |

---

## Practical Commands

### Adding Taints

#### Add a single taint to a node

```bash
kubectl taint nodes <node-name> <key>=<value>:<effect>
```

Example:

```bash
# Reserve a node for GPU workloads
kubectl taint nodes worker-gpu-1 gpu=true:NoSchedule

# Mark a node for production only
kubectl taint nodes prod-node environment=production:NoSchedule

# Mark a spot instance node
kubectl taint nodes spot-1 spot=true:PreferNoSchedule

# Mark node for maintenance
kubectl taint nodes maintenance-node maintenance=true:NoExecute
```

#### Add multiple taints to the same node

```bash
kubectl taint nodes worker-gpu-1 gpu=true:NoSchedule
kubectl taint nodes worker-gpu-1 team=ml:NoSchedule
```

Now this node requires pods to tolerate BOTH taints.

---

### Removing Taints

Add a `-` at the end of the effect to remove a taint:

```bash
kubectl taint nodes <node-name> <key>=<value>:<effect>-
```

Example:

```bash
# Remove the GPU taint
kubectl taint nodes worker-gpu-1 gpu=true:NoSchedule-

# Remove the maintenance taint
kubectl taint nodes maintenance-node maintenance=true:NoExecute-

# Remove all taints from a node
kubectl taint nodes worker-gpu-1 gpu-
kubectl taint nodes worker-gpu-1 team-
```

---

### Viewing Taints

#### See taints on a specific node

```bash
kubectl describe node <node-name>
```

This shows a lot of info. Look for the **Taints** section.

Example output:

```
Name:               worker-gpu-1
...
Taints:             gpu=true:NoSchedule
                    team=ml:NoSchedule
```

#### See taints on all nodes

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

Output:

```
NAME              TAINTS
master-1          <none>
worker-gpu-1      [{"key":"gpu","value":"true","effect":"NoSchedule"}]
worker-2          <none>
```

#### Get detailed taint info in JSON

```bash
kubectl get node <node-name> -o json | grep -A 10 taints
```

---

## Adding Tolerations to Pods

You add tolerations to the pod spec. They act as permission to run on tainted nodes.

### Basic Toleration Syntax

```yaml
tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

### Toleration Operators

| Operator | What It Means | Example |
|----------|-------------|---------|
| **Equal** | Toleration value must exactly match taint value | `value: "true"` |
| **Exists** | Toleration matches any taint value (ignores value) | No `value` field needed |

---

## Real-World Example: GPU Node

### Step 1: Taint the GPU node

```bash
kubectl taint nodes gpu-worker gpu=true:NoSchedule
```

### Step 2: Create a pod with matching toleration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-training-pod
spec:
  containers:
  - name: ml-app
    image: my-ml-app:latest
    resources:
      limits:
        nvidia.com/gpu: 1
  
  # Add toleration to allow running on GPU nodes
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

### Step 3: Deploy the pod

```bash
kubectl apply -f ml-training-pod.yaml
```

✅ This pod can now run on the GPU node!

### Step 4: Try deploying a pod without toleration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: regular-pod
spec:
  containers:
  - name: app
    image: nginx:latest
```

```bash
kubectl apply -f regular-pod.yaml
```

❌ This pod will NOT be scheduled on the GPU node (no toleration).

---

## Example 1: Dedicated Nodes for Teams

**Scenario:** You want Node A only for the ML team and Node B only for the Frontend team.

### Setup

```bash
# Taint ML node
kubectl taint nodes ml-node team=ml:NoSchedule

# Taint Frontend node
kubectl taint nodes frontend-node team=frontend:NoSchedule
```

### ML Team Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-job
spec:
  containers:
  - name: ml-container
    image: ml-app:latest
  tolerations:
  - key: "team"
    operator: "Equal"
    value: "ml"
    effect: "NoSchedule"
```

### Frontend Team Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend-app
spec:
  containers:
  - name: web-app
    image: frontend:latest
  tolerations:
  - key: "team"
    operator: "Equal"
    value: "frontend"
    effect: "NoSchedule"
```

---

## Example 2: Node Maintenance with NoExecute

**Scenario:** You need to drain a node for maintenance but want to give apps time to gracefully shut down.

### Step 1: Add NoExecute taint

```bash
kubectl taint nodes worker-1 maintenance=true:NoExecute
```

This immediately evicts all pods without toleration.

### Step 2: For apps that need graceful shutdown

Add toleration with `tolerationSeconds`:

```yaml
tolerations:
- key: "maintenance"
  operator: "Equal"
  value: "true"
  effect: "NoExecute"
  tolerationSeconds: 300  # Give 300 seconds to shut down gracefully
```

This pod will get 5 minutes to finish before being evicted.

### Step 3: After maintenance, remove the taint

```bash
kubectl taint nodes worker-1 maintenance=true:NoExecute-
```

---

## Example 3: Using Exists Operator

**Scenario:** You want ANY pod with a toleration to match, regardless of the value.

### Taint the node

```bash
kubectl taint nodes special-node workload-type=any:NoSchedule
```

### Pod with Exists operator (matches ANY value)

```yaml
tolerations:
- key: "workload-type"
  operator: "Exists"
  effect: "NoSchedule"
```

✅ This pod tolerates the taint regardless of the value.

### Pod without value in toleration

```yaml
tolerations:
- key: "workload-type"
  operator: "Equal"
  value: "specific"
  effect: "NoSchedule"
```

❌ This pod only tolerates if value is "specific".

---

## Multiple Taints and Tolerations

**Important Rule:** If a node has multiple taints and a pod doesn't tolerate even ONE of them, the pod won't be scheduled.

### Node with multiple taints

```bash
kubectl taint nodes worker-1 gpu=true:NoSchedule
kubectl taint nodes worker-1 team=ml:NoSchedule
kubectl taint nodes worker-1 memory=high:NoSchedule
```

### Pod that tolerates all three

```yaml
tolerations:
- key: "gpu"
  operator: "Equal"
  value: "true"
  effect: "NoSchedule"
- key: "team"
  operator: "Equal"
  value: "ml"
  effect: "NoSchedule"
- key: "memory"
  operator: "Equal"
  value: "high"
  effect: "NoSchedule"
```

✅ This pod can run on worker-1.

### Pod that tolerates only two

```yaml
tolerations:
- key: "gpu"
  operator: "Equal"
  value: "true"
  effect: "NoSchedule"
- key: "team"
  operator: "Equal"
  value: "ml"
  effect: "NoSchedule"
```

❌ Missing toleration for `memory=high:NoSchedule` → won't schedule.

---

## Automatic Taints (System Taints)

Kubernetes automatically adds these taints when node conditions occur:

| Taint | Meaning | Effect |
|-------|---------|--------|
| `node.kubernetes.io/not-ready:NoExecute` | Node is not ready | Existing pods evicted |
| `node.kubernetes.io/unreachable:NoExecute` | Node is unreachable | Existing pods evicted |
| `node.kubernetes.io/memory-pressure:NoSchedule` | Node is low on memory | New pods blocked |
| `node.kubernetes.io/disk-pressure:NoSchedule` | Node is low on disk | New pods blocked |

Most pods have automatic tolerations for these, so you don't usually see evictions.

---

## Taints vs Node Affinity

| Feature | Taints | Node Affinity |
|---------|--------|--------------|
| Direction | Node repels pods (push away) | Pod attracts to nodes (pull towards) |
| Use Case | Reserve nodes or prevent scheduling | Attract pods to specific nodes |
| Strictness | Can be strict (NoSchedule) or soft (PreferNoSchedule) | Can be required or preferred |

**Best Practice:** Use BOTH together for maximum control.

---

## Troubleshooting

### Pod won't schedule on a node

**Check 1:** Does the node have taints?

```bash
kubectl describe node <node-name> | grep -A 5 Taints
```

**Check 2:** Does the pod have matching tolerations?

```bash
kubectl describe pod <pod-name> | grep -A 10 Tolerations
```

**Check 3:** Check if there's a resource constraint

```bash
kubectl describe node <node-name> | grep -A 5 "Allocated resources"
```

### Can't remove pods from a node

If a taint won't evict pods, they might have a toleration with `tolerationSeconds`:

```bash
kubectl describe pod <pod-name> | grep -A 10 tolerationSeconds
```

Wait for the time to elapse or delete the pod manually:

```bash
kubectl delete pod <pod-name> --grace-period=0 --force
```

---

## Best Practices

✅ **DO:**
- Use taints to reserve expensive resources (GPUs, high-memory nodes)
- Name taints descriptively (`gpu=true`, `environment=production`)
- Document your taint strategy in your team
- Test toleration configurations before deploying to production
- Use `PreferNoSchedule` for loose constraints
- Use `NoSchedule` for strict constraints
- Combine with Node Affinity for complex scheduling

❌ **DON'T:**
- Overuse taints (causes scheduling complexity)
- Create overly specific taint-toleration combinations
- Forget to add tolerations when tainting nodes
- Use `NoExecute` carelessly (can disrupt running apps)
- Ignore automatic system taints

---

## Quick Command Reference

```bash
# Add taint
kubectl taint nodes <node> <key>=<value>:<effect>

# Remove taint
kubectl taint nodes <node> <key>=<value>:<effect>-

# View taints on a node
kubectl describe node <node>

# View taints on all nodes
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# Check pod tolerations
kubectl describe pod <pod>

# Get detailed JSON info
kubectl get node <node> -o json
```

---

## Summary

- **Taints** mark nodes with restrictions
- **Tolerations** give pods permission to run on tainted nodes
- **Effects**: `NoSchedule` (block), `PreferNoSchedule` (avoid), `NoExecute` (evict)
- **Operators**: `Equal` (exact match) or `Exists` (any value)
- Combine with Node Affinity for powerful scheduling control
- Use for reserving expensive hardware, dedicating nodes, or maintenance

---

## References

- [Kubernetes Official Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Taints and Tolerations Best Practices](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
