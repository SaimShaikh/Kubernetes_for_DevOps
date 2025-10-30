# Kubernetes apiVersion 
## What is apiVersion?

Think of `apiVersion` as a **label that tells Kubernetes which part of its brain should handle your object**.

Every Kubernetes object (Pod, Service, Deployment, etc.) needs an `apiVersion` in its YAML file. It's like saying "Hey Kubernetes, this is a Deployment, so use the apps/v1 controller to manage it."

---

## The Three Main Categories

### 1️⃣ Core API → `v1`

These are the **basic building blocks** of Kubernetes. They're so fundamental that they don't need a special group name.

| What It Is | apiVersion | What It Does |
|-----------|-----------|------------|
| **Pod** | `v1` | Single container or group of containers running together |
| **Service** | `v1` | Exposes Pods so other services can talk to them |
| **ConfigMap** | `v1` | Stores settings and configuration data |
| **Secret** | `v1` | Stores passwords, tokens, and sensitive stuff |
| **Namespace** | `v1` | Divides your cluster into separate spaces |
| **PersistentVolume** | `v1` | Storage space in your cluster |
| **PersistentVolumeClaim** | `v1` | "I need storage" - requests from Pods |

**Quick Rule:** If it's a core, basic resource → use `v1`

---

### 2️⃣ Workload Controllers → `apps/v1`

These are the **smart managers** that control Pods. They handle scaling, updates, and making sure the right number of Pods are always running.

| What It Is | apiVersion | What It Does |
|-----------|-----------|------------|
| **Deployment** | `apps/v1` | Manages regular apps (most common) |
| **ReplicaSet** | `apps/v1` | Keeps a fixed number of Pods running |
| **StatefulSet** | `apps/v1` | Manages apps that need identity (like databases) |
| **DaemonSet** | `apps/v1` | Runs one Pod on every single Node |

**Quick Rule:** If it controls or manages Pods → use `apps/v1`

---

### 3️⃣ Special Purpose Groups

Different Kubernetes features live in different API groups:

| What It Is | apiVersion | What It Does |
|-----------|-----------|------------|
| **Ingress** | `networking.k8s.io/v1` | Routes traffic from outside into your cluster |
| **NetworkPolicy** | `networking.k8s.io/v1` | Controls which Pods can talk to which Pods |
| **Job** | `batch/v1` | Runs one-time tasks (finish and stop) |
| **CronJob** | `batch/v1` | Runs tasks on a schedule (like a timer) |
| **Role** | `rbac.authorization.k8s.io/v1` | Defines who can do what (permissions) |
| **ClusterRole** | `rbac.authorization.k8s.io/v1` | Permissions across the whole cluster |

---

## Why Does This Matter?

When you run this command:

```bash
kubectl apply -f myapp.yaml
```

Kubernetes reads the `apiVersion` in your file and asks:
- "Which controller should handle this?"
- "What fields are allowed?"
- "Which rules apply?" (scaling, updates, etc.)

If you use the **wrong apiVersion**, Kubernetes won't understand what you want.

### Example of Wrong vs Right

❌ **WRONG:**
```yaml
apiVersion: v1
kind: Deployment
```

You get this error:
```
error: unable to recognize "file.yaml": no matches for kind "Deployment" in version "v1"
```

✅ **CORRECT:**
```yaml
apiVersion: apps/v1
kind: Deployment
```

---

## Simple Examples

### Example 1: Creating a Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: web
    image: nginx:latest
```

### Example 2: Creating a Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:latest
```

### Example 3: Creating a Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: web
```

### Example 4: Creating an Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: my-service
            port:
              number: 80
```

---

## The Quick Cheat Sheet

| When You Need... | Use apiVersion |
|------------------|----------------|
| Pod, Service, ConfigMap, Secret, Namespace | `v1` |
| Deployment, ReplicaSet, StatefulSet, DaemonSet | `apps/v1` |
| Ingress, NetworkPolicy | `networking.k8s.io/v1` |
| Job, CronJob | `batch/v1` |
| Role, ClusterRole | `rbac.authorization.k8s.io/v1` |

---

## Key Takeaways for DevOps

1. **Core stuff = v1** → Pods, Services, ConfigMaps, Secrets
2. **Workload controllers = apps/v1** → Deployments, ReplicaSets, StatefulSets, DaemonSets
3. **Special features = custom groups** → Networking, Batch jobs, RBAC
4. **Always check your apiVersion** → Wrong version = error
5. **apiVersion tells Kubernetes which brain to use** → Different rules apply based on the apiVersion

---

## Common Mistakes to Avoid

❌ Using `v1` for Deployment (it's `apps/v1`)  
❌ Using `v1` for Ingress (it's `networking.k8s.io/v1`)  
❌ Forgetting to include apiVersion at all  
❌ Mixing up Job (batch/v1) with Deployment (apps/v1)

---

## Resources to Learn More

- Check which apiVersion you need by running: `kubectl api-resources`
- Official Kubernetes API docs: https://kubernetes.io/docs/reference/
- Always copy templates from your cluster's API to be sure

---

**Remember:** Every YAML file starts with `apiVersion`. Get this right, and Kubernetes knows exactly what to do. Get it wrong, and you'll see errors. 🎯
