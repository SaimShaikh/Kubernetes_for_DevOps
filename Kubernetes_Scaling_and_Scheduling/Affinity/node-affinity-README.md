# Kubernetes Node Affinity

## What is Node Affinity?

**Node affinity** is a simple way to tell Kubernetes which nodes your pods should run on. It uses **labels** on nodes to decide where to place your pods.

Think of it like this: You have multiple worker machines, and you want your app to run only on specific machines based on certain characteristics (like hardware type or location).

---

## Why Use Node Affinity?

Imagine you have:
- Some nodes with **GPU** (graphics cards)
- Some nodes with **regular CPU only**
- Nodes in **different data centers**

Node affinity helps you say: *"Hey Kubernetes, please run my AI app on GPU nodes and my web app on CPU nodes."*

**Common use cases:**
- Run memory-heavy apps on nodes with more memory
- Run GPU applications on nodes with GPUs
- Keep pods in specific zones or regions
- Separate workloads on different types of hardware

---

## Two Types of Node Affinity

### 1. Required (Hard Rule)
**"Please schedule my pod ONLY if you find a matching node"**

If no matching nodes exist, the pod **will NOT be scheduled** (it stays in pending state).

```yaml
requiredDuringSchedulingIgnoredDuringExecution
```

### 2. Preferred (Soft Rule)
**"Try to find a matching node, but if you can't, schedule it anywhere"**

The scheduler tries to find the best match, but if it can't, it will schedule the pod on any available node.

```yaml
preferredDuringSchedulingIgnoredDuringExecution
```

---

## How to Use Node Affinity

### Step 1: Label Your Nodes

First, add labels to the nodes where you want pods to run.

```bash
# Label a node as "frontend"
kubectl label nodes worker-1 role=frontend

# Label another node as "backend"
kubectl label nodes worker-2 role=backend

# View all node labels
kubectl get nodes --show-labels
```

### Step 2: Create a Pod with Node Affinity

Add an `affinity` section in your pod specification.

#### Example 1: Simple Required Affinity

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: nginx:latest
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: role
            operator: In
            values:
            - frontend
```

**What this means:** "Schedule this pod ONLY on nodes labeled with `role=frontend`"

#### Example 2: Simple Preferred Affinity

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: nginx:latest
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        nodeSelectorTerms:
        - matchExpressions:
          - key: role
            operator: In
            values:
            - backend
```

**What this means:** "Try to schedule on nodes with `role=backend`, but if none available, schedule anywhere. The weight (100) shows priority."

---

## Operators: How to Match Labels

Node affinity supports different operators for matching:

| Operator | Meaning |
|----------|---------|
| `In` | Label value is in the list |
| `NotIn` | Label value is NOT in the list |
| `Exists` | Label key exists (any value) |
| `DoesNotExist` | Label key does NOT exist |
| `Gt` | Label value is greater than (for numbers) |
| `Lt` | Label value is less than (for numbers) |

### Simple Examples:

```yaml
# Match: role = frontend OR role = backend
- key: role
  operator: In
  values:
  - frontend
  - backend

# Match: role != testing
- key: role
  operator: NotIn
  values:
  - testing

# Match: label "gpu" exists (any value)
- key: gpu
  operator: Exists
```

---

## Real-World Example: Deployment with Node Affinity

Here's a complete deployment example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
      
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: environment
                operator: In
                values:
                - production
              - key: zone
                operator: In
                values:
                - zone-a
                - zone-b
```

**This deployment will:**
- Run only on nodes labeled with `environment=production`
- Run only in `zone-a` OR `zone-b`
- If no such nodes exist, pods will stay in pending state

---

## Step-by-Step Tutorial

### Step 1: Check Your Nodes

```bash
kubectl get nodes
# Output:
# NAME       STATUS   ROLES    AGE   VERSION
# worker-1   Ready    <none>   5d    v1.25.0
# worker-2   Ready    <none>   5d    v1.25.0
```

### Step 2: Label Your Nodes

```bash
kubectl label nodes worker-1 workload=compute
kubectl label nodes worker-2 workload=memory
```

### Step 3: Verify Labels

```bash
kubectl get nodes --show-labels
```

### Step 4: Create a Deployment YAML File

Save this as `my-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: compute-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: compute
  template:
    metadata:
      labels:
        app: compute
    spec:
      containers:
      - name: app
        image: busybox
        command: ['sh', '-c', 'echo "Running on compute node" && sleep 3600']
      
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: workload
                operator: In
                values:
                - compute
```

### Step 5: Apply the Deployment

```bash
kubectl apply -f my-deployment.yaml
```

### Step 6: Check Where Pods Are Running

```bash
kubectl get pods -o wide
# Output shows which node each pod is running on
```

All pods should be on `worker-1` (the node with `workload=compute` label).

---

## Node Affinity vs NodeSelector: Simple Comparison

| Feature | NodeSelector | Node Affinity |
|---------|--------------|---------------|
| **Ease** | Simpler | Slightly more complex |
| **Flexibility** | Basic label matching | Advanced operators (In, NotIn, Gt, Lt, Exists) |
| **Required/Preferred** | Only required | Both available |
| **Best For** | Simple needs | Complex scheduling needs |

**Simple rule:** Use **NodeSelector** for basic cases, use **Node Affinity** when you need more control.

---

## Common Mistakes to Avoid

❌ **Mistake 1:** Forgetting to label nodes first
```bash
# Label nodes BEFORE creating pods with affinity
kubectl label nodes worker-1 role=web
```

❌ **Mistake 2:** Using required affinity without enough nodes
```yaml
# If no nodes have this label, pods will stay pending!
requiredDuringSchedulingIgnoredDuringExecution
```

❌ **Mistake 3:** Wrong operator
```yaml
# Check your operator spelling (In, NotIn, Exists, etc.)
operator: In  # ✓ Correct
operator: in  # ✗ Wrong (case-sensitive)
```

---

## Key Takeaways

1. **Node affinity** uses node labels to control pod placement
2. **Required** affinity = pod MUST match (hard rule)
3. **Preferred** affinity = pod SHOULD match (soft rule)
4. Always **label nodes first**, then use affinity in your pods
5. Use operators like `In`, `NotIn`, `Exists` to match labels
6. Check pod status with `kubectl get pods -o wide` to see where pods are scheduled

---

## Useful Commands

```bash
# Label a node
kubectl label nodes <node-name> <key>=<value>

# View node labels
kubectl get nodes --show-labels

# Remove a label from a node
kubectl label nodes <node-name> <key>-

# View pod placement
kubectl get pods -o wide

# Debug why pod isn't scheduled
kubectl describe pod <pod-name>
```

---

## Next Steps

- Try creating multiple deployments with different affinity rules
- Experiment with `requiredDuringSchedulingIgnoredDuringExecution` vs `preferredDuringSchedulingIgnoredDuringExecution`
- Learn about **Pod Affinity** (scheduling pods together)
- Learn about **Taints and Tolerations** (opposite approach - nodes control which pods run)

---

**Happy Scheduling!** 🚀
