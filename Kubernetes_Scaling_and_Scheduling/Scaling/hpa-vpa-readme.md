# Kubernetes HPA and VPA - Autoscaling Guide

## What is Autoscaling?

**Autoscaling** means automatically adjusting resources based on your application's current demand. Instead of manually adding or removing resources, Kubernetes does it for you.

Think of it like:
- **HPA (Horizontal Pod Autoscaler):** Hiring more workers when work increases
- **VPA (Vertical Pod Autoscaler):** Giving your current workers more tools and resources

---

## What is HPA (Horizontal Pod Autoscaler)?

**HPA** automatically increases or decreases the **number of pod replicas** based on metrics like CPU usage, memory, or custom metrics.

### How HPA Works

```
Load increases → HPA detects high CPU/Memory → Creates more pod replicas
Load decreases → HPA detects low CPU/Memory → Removes pod replicas
```

**Example:**
- You have 2 pods running your web app
- Traffic spikes and CPU goes to 80%
- HPA scales to 5 pods automatically
- Traffic drops and CPU goes to 20%
- HPA scales back to 2 pods

---

## What is VPA (Vertical Pod Autoscaler)?

**VPA** automatically adjusts the **CPU and memory requests/limits** of individual pods based on actual usage over time.

### How VPA Works

```
Pod uses more resources than allocated → VPA detects pattern → Increases CPU/Memory for that pod
Pod uses less resources than allocated → VPA detects pattern → Decreases CPU/Memory for that pod
```

**Example:**
- Your pod is allocated 500Mi memory but uses 900Mi consistently
- VPA notices this pattern
- VPA increases memory allocation to 1Gi
- Pod gets restarted with new resources

---

## Why Do We Need Autoscaling?

### Problems Without Autoscaling

❌ **Manual scaling is slow:** By the time you notice high load and add pods manually, your app may have already crashed.

❌ **Resource waste:** If you over-provision (add too many pods), you waste money on unused resources.

❌ **Poor performance:** If you under-provision (too few resources), your app becomes slow or unresponsive.

### Benefits of Autoscaling

✅ **Automatic response:** Reacts to load changes in minutes without human intervention

✅ **Cost savings:** Scales down during low traffic, saving money

✅ **Better performance:** Ensures applications always have the resources they need

✅ **High availability:** Prevents crashes during traffic spikes

✅ **Efficient resource usage:** No over-provisioning or under-provisioning

---

## HPA Benefits

| Benefit | Description |
|---------|-------------|
| **Fast response to traffic spikes** | Adds pods quickly when load increases (2-4 minutes typical) |
| **Non-disruptive scaling** | New pods are added without affecting existing ones |
| **Handles variable workloads** | Perfect for apps with unpredictable traffic patterns |
| **Easy to configure** | Simple YAML configuration with CPU/memory targets |
| **Cost-effective** | Scales down automatically when traffic decreases |

---

## VPA Benefits

| Benefit | Description |
|---------|-------------|
| **Right-sizing pods** | Automatically finds the optimal CPU/memory for each pod |
| **Reduces resource waste** | Prevents over-provisioning by adjusting to actual usage |
| **Prevents throttling** | Ensures pods have enough resources to avoid slowdowns |
| **Continuous optimization** | Monitors usage patterns over time and adjusts accordingly |
| **Good for unpredictable workloads** | Helps when you don't know resource requirements upfront |

---

## HPA vs VPA - Complete Comparison

| Feature | HPA (Horizontal Pod Autoscaler) | VPA (Vertical Pod Autoscaler) |
|---------|--------------------------------|-------------------------------|
| **What it scales** | Number of pod replicas | CPU and memory per pod |
| **Scaling direction** | Horizontal (more/fewer pods) | Vertical (bigger/smaller pods) |
| **Response time** | Fast (2-4 minutes) | Slow (hours to days for recommendations) |
| **Disruption** | Non-disruptive (adds new pods) | Disruptive (restarts pods with new resources) |
| **Best for** | Traffic spikes, variable workload | Steady workloads with unclear sizing |
| **Use case** | Web apps, APIs, microservices | Batch jobs, data processing, databases |
| **Metrics** | CPU, memory, custom metrics | Historical resource usage patterns |
| **Availability impact** | No downtime (gradual scaling) | Brief interruption (pod restart) |
| **Configuration complexity** | Simple (set min/max replicas + target) | Moderate (set update mode + policies) |
| **When to use** | Load changes frequently | Resource needs change over time |
| **Installation** | Built into Kubernetes | Requires separate installation |
| **Stateless apps** | ✅ Excellent | ⚠️ Use with caution |
| **Stateful apps** | ⚠️ Use with caution | ✅ Good fit |
| **Cost optimization** | Saves money during low traffic | Prevents over-provisioning |

---

## When to Use HPA vs VPA

### Use HPA When:

✅ You have **unpredictable traffic** (e.g., e-commerce during sales, news sites during breaking news)

✅ Your application is **stateless** (web servers, REST APIs)

✅ You need **fast response** to load changes

✅ You can handle **multiple replicas** of the same pod

✅ **Examples:** Web applications, API gateways, microservices, queue processors

---

### Use VPA When:

✅ Your application has **steady traffic** but resource needs vary

✅ You don't know the **optimal CPU/memory** settings upfront

✅ Your application is **stateful** (databases, caches)

✅ You want to **optimize resource usage** over time

✅ **Examples:** Databases, batch processing, data pipelines, ML training jobs

---

## HPA Configuration and Commands

### Prerequisites

Before using HPA, you need the **Metrics Server** installed:

```bash
# Install Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Check if Metrics Server is running
kubectl get deployment metrics-server -n kube-system

# Verify metrics are available
kubectl top nodes
kubectl top pods
```

---

### Creating HPA - Method 1: Using kubectl autoscale

**Basic HPA based on CPU:**

```bash
kubectl autoscale deployment <deployment-name> \
  --cpu-percent=50 \
  --min=2 \
  --max=10
```

**Example:**

```bash
# Autoscale nginx deployment
kubectl autoscale deployment nginx-deployment \
  --cpu-percent=50 \
  --min=2 \
  --max=10
```

This means:
- Keep minimum **2 pods** running
- Scale up to **10 pods** maximum
- Scale when CPU usage crosses **50%**

---

### Creating HPA - Method 2: Using YAML Manifest

**Simple HPA YAML (CPU only):**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Apply it:

```bash
kubectl apply -f nginx-hpa.yaml
```

---

**HPA with CPU and Memory:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 3
  maxReplicas: 15
  metrics:
  # Scale based on CPU
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  # Scale based on Memory
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 500Mi
```

This scales when:
- CPU usage exceeds **70%** OR
- Memory usage exceeds **500Mi** (picks the largest scale)

---

### Viewing HPA Status

```bash
# List all HPAs
kubectl get hpa

# Get detailed HPA info
kubectl get hpa <hpa-name>

# Describe HPA (shows scaling events)
kubectl describe hpa <hpa-name>

# Watch HPA in real-time
kubectl get hpa <hpa-name> --watch
```

**Example output:**

```
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx-hpa    Deployment/nginx-deployment   45%/50%   2         10        3          5m
```

This means:
- Current CPU usage: **45%**
- Target CPU: **50%**
- Currently running **3 replicas**

---

### Testing HPA

**Step 1: Create a test deployment**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
```

```bash
kubectl apply -f php-apache-deployment.yaml
```

**Step 2: Create HPA**

```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

**Step 3: Generate load**

Open a new terminal and run:

```bash
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

**Step 4: Watch HPA scale**

In another terminal:

```bash
kubectl get hpa php-apache --watch
```

You'll see replicas increase as CPU goes up!

**Step 5: Stop load**

Press `Ctrl+C` in the load-generator terminal. Watch replicas scale back down.

---

### Deleting HPA

```bash
# Delete specific HPA
kubectl delete hpa <hpa-name>

# Example
kubectl delete hpa nginx-hpa
```

---

## VPA Configuration and Commands

### Prerequisites

VPA is **not installed by default**. You need to install it first:

```bash
# Clone VPA repository
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler

# Install VPA
./hack/vpa-up.sh

# Verify VPA components are running
kubectl get pods -n kube-system | grep vpa
```

You should see three VPA components:
- `vpa-recommender`
- `vpa-updater`
- `vpa-admission-controller`

---

### Creating VPA - YAML Manifest

**Basic VPA YAML:**

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  updatePolicy:
    updateMode: "Auto"
```

**Update Modes:**

| Mode | What It Does |
|------|-------------|
| **Off** | Only provides recommendations (no action) |
| **Initial** | Sets resources only when pod is created |
| **Recreate** | Restarts pods when resources need updating |
| **Auto** | Automatically updates pods (evicts and recreates) |

---

**VPA with Resource Policies:**

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: webapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: webapp-container
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2
        memory: 2Gi
      controlledResources: ["cpu", "memory"]
```

This ensures:
- CPU stays between **100m and 2 cores**
- Memory stays between **128Mi and 2Gi**

Apply it:

```bash
kubectl apply -f webapp-vpa.yaml
```

---

### Viewing VPA Recommendations

```bash
# List all VPAs
kubectl get vpa

# Get detailed VPA info
kubectl describe vpa <vpa-name>

# Example
kubectl describe vpa nginx-vpa
```

**Example output:**

```
Name:         nginx-vpa
Namespace:    default
...
Status:
  Recommendation:
    Container Recommendations:
      Container Name:  nginx
      Lower Bound:
        Cpu:     100m
        Memory:  128Mi
      Target:
        Cpu:     250m
        Memory:  256Mi
      Upper Bound:
        Cpu:     500m
        Memory:  512Mi
```

**What this means:**
- **Target:** VPA recommends **250m CPU** and **256Mi memory**
- **Lower Bound:** Minimum resources pod should have
- **Upper Bound:** Maximum resources pod might need

---

### Testing VPA

**Step 1: Create a test deployment**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hamster
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hamster
  template:
    metadata:
      labels:
        app: hamster
    spec:
      containers:
      - name: hamster
        image: registry.k8s.io/ubuntu-slim:0.1
        resources:
          requests:
            cpu: 100m
            memory: 50Mi
        command: ["/bin/sh"]
        args:
          - "-c"
          - "while true; do timeout 0.5s yes >/dev/null; sleep 0.5s; done"
```

```bash
kubectl apply -f hamster-deployment.yaml
```

**Step 2: Create VPA**

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: hamster-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hamster
  updatePolicy:
    updateMode: "Auto"
```

```bash
kubectl apply -f hamster-vpa.yaml
```

**Step 3: Watch VPA recommendations**

```bash
kubectl describe vpa hamster-vpa
```

Wait a few minutes. VPA will analyze usage and provide recommendations.

**Step 4: Check if pod was recreated with new resources**

```bash
kubectl get pods
kubectl describe pod <hamster-pod-name>
```

Look at the `Requests` section to see updated CPU/memory values.

---

### VPA Recommendation-Only Mode

If you want VPA to **only suggest** resources without automatically applying them:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  updatePolicy:
    updateMode: "Off"  # Only recommend, don't apply
```

View recommendations:

```bash
kubectl describe vpa nginx-vpa
```

Then manually update your deployment with recommended values.

---

### Deleting VPA

```bash
# Delete specific VPA
kubectl delete vpa <vpa-name>

# Example
kubectl delete vpa nginx-vpa

# Uninstall VPA completely
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-down.sh
```

---

## Can You Use HPA and VPA Together?

⚠️ **Generally NOT recommended**, but possible with caution.

### Why It's Tricky

- **HPA** scales based on CPU/Memory **percentage**
- **VPA** changes the CPU/Memory **requests**
- If VPA increases requests, HPA might think CPU% is lower and scale down pods (conflict!)

### If You Must Use Both

Use this combination:
- **HPA:** Scale based on CPU
- **VPA:** Scale based on Memory only

Example VPA that only controls memory:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: webapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      controlledResources: ["memory"]  # Only memory, not CPU
```

---

## Real-World Examples

### Example 1: E-commerce Website (Use HPA)

**Scenario:** Online store with traffic spikes during sales.

**Solution:** Use HPA to add more pods during high traffic.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ecommerce-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ecommerce-app
  minReplicas: 5
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

---

### Example 2: Data Processing Job (Use VPA)

**Scenario:** Batch job that processes different-sized datasets.

**Solution:** Use VPA to adjust memory based on dataset size.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: data-processor-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: data-processor
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: processor
      minAllowed:
        memory: 512Mi
      maxAllowed:
        memory: 8Gi
```

---

### Example 3: REST API with Custom Metrics (Advanced HPA)

**Scenario:** Scale based on request rate (custom metric).

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: rest-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
```

---

## Troubleshooting

### HPA Issues

**Problem:** HPA shows `<unknown>` for metrics

```bash
kubectl get hpa
# NAME         REFERENCE               TARGETS         MINPODS   MAXPODS
# nginx-hpa    Deployment/nginx        <unknown>/50%   2         10
```

**Solution:**
- Metrics Server is not installed or not working
- Check: `kubectl get pods -n kube-system | grep metrics-server`
- Ensure pods have resource requests defined

---

**Problem:** HPA not scaling

**Solution:**
- Check if deployment has `resources.requests` defined
- View HPA events: `kubectl describe hpa <hpa-name>`
- Check current metrics: `kubectl top pods`

---

### VPA Issues

**Problem:** VPA not updating pods

**Solution:**
- Check VPA components are running: `kubectl get pods -n kube-system | grep vpa`
- Verify `updateMode` is set to `Auto` or `Recreate`
- Check VPA logs: `kubectl logs -n kube-system <vpa-recommender-pod>`

---

**Problem:** Pods keep restarting

**Solution:**
- VPA is evicting pods to update resources
- Use `updateMode: "Off"` to only get recommendations
- Set `minAllowed` and `maxAllowed` to limit resource changes

---

## Best Practices

### HPA Best Practices

✅ Always define `resources.requests` in your pod specs (HPA needs this!)

✅ Set realistic `minReplicas` and `maxReplicas` (don't set max too high)

✅ Use `targetAverageUtilization` between **60-80%** (not too low or high)

✅ Monitor HPA behavior with `kubectl get hpa --watch`

✅ Test scaling behavior before production

✅ Combine with Cluster Autoscaler for node-level scaling

---

### VPA Best Practices

✅ Start with `updateMode: "Off"` to see recommendations first

✅ Set `minAllowed` and `maxAllowed` to prevent extreme resource changes

✅ Use VPA for **stateful apps** and **batch jobs**

✅ Don't use VPA with HPA on the same resource (unless separating CPU/memory)

✅ Monitor VPA recommendations regularly: `kubectl describe vpa`

✅ Be aware that VPA restarts pods (plan for brief downtime)

---

## Quick Command Reference

### HPA Commands

```bash
# Create HPA
kubectl autoscale deployment <name> --cpu-percent=50 --min=2 --max=10

# List HPAs
kubectl get hpa

# Describe HPA
kubectl describe hpa <hpa-name>

# Watch HPA
kubectl get hpa <hpa-name> --watch

# Delete HPA
kubectl delete hpa <hpa-name>

# Check metrics
kubectl top pods
kubectl top nodes
```

---

### VPA Commands

```bash
# Install VPA
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh

# List VPAs
kubectl get vpa

# Describe VPA (see recommendations)
kubectl describe vpa <vpa-name>

# Delete VPA
kubectl delete vpa <vpa-name>

# Uninstall VPA
./hack/vpa-down.sh

# Check VPA components
kubectl get pods -n kube-system | grep vpa
```

---

## Summary

| Feature | HPA | VPA |
|---------|-----|-----|
| **Scaling Type** | Horizontal (more pods) | Vertical (bigger pods) |
| **Best For** | Traffic spikes | Resource optimization |
| **Speed** | Fast (minutes) | Slow (hours/days) |
| **Disruption** | None | Pod restarts |
| **Use Case** | Web apps, APIs | Databases, batch jobs |

**Key Takeaway:** Use **HPA for traffic**, **VPA for sizing**, and avoid using both together unless you know what you're doing!

---

## References

- [Kubernetes Official HPA Documentation](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Kubernetes Official VPA Documentation](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [HPA Walkthrough Guide](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
