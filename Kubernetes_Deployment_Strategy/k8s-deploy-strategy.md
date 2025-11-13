# Kubernetes Deployment Strategies

Welcome to the **Kubernetes Deployment Strategies** guide! This document explains different ways to release new versions of your applications in Kubernetes in simple, easy-to-understand language.

---

## 📚 What You'll Learn

By the end of this guide, you'll understand:

- **What** are deployment strategies and why they matter
- **Why** different strategies exist for different situations
- **When** to use each strategy
- **How** each strategy works with real-world examples
- **Benefits and drawbacks** of each approach
- **Best practices** to follow

---

## 🎯 What is a Deployment Strategy?

A **deployment strategy** is a plan for how you release new versions of your application to users. It defines:

- How old pods (running your old version) are replaced with new pods (running your new version)
- How traffic is switched between versions
- How quickly or gradually the update happens
- What happens if something goes wrong

**Simple analogy:** It's like replacing all the light bulbs in a building. You can replace them all at once (fast but risky), one room at a time (slow but safe), or test one first before replacing all (safe and smart).

---

## ❓ Why Do We Need Different Strategies?

Different applications have different needs:

- **Banking app:** Needs to be available 24/7, can't go down. Must switch back instantly if something breaks.
- **Social media feature:** Can tolerate some downtime. Want to test with real users first.
- **Internal tool:** Less critical, can handle temporary downtime during updates.

Each strategy balances three things differently:

| Aspect | What It Means |
|--------|---------------|
| **Speed** | How fast the deployment completes |
| **Safety** | How well it protects against errors |
| **Cost** | How many resources (servers) you need |
| **Risk** | How many users are affected if something breaks |

---

## 🚀 Deployment Strategies Explained

### 1. **Recreate Strategy** (The Simple Way)

#### What It Does
Stops all old pods immediately and starts all new pods at once.

**Timeline:**
```
Old pods: Running Running Running STOP
New pods: STOP    STOP    STOP    Running Running Running
Users:    Getting service...     [DOWNTIME]    Getting service again
```

#### When to Use
- Development environments (testing code locally)
- Non-critical applications
- When downtime is acceptable (scheduled maintenance)

#### Pros ✅
- Very simple to understand
- Fast deployment
- Low cost (no duplicate infrastructure)

#### Cons ❌
- **Service is down** during deployment (users can't access your app)
- Risky for production
- If new version has bugs, everyone is affected

#### Example Command
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  strategy:
    type: Recreate
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-app:v2  # New version
```

---

### 2. **Rolling Update Strategy** (The Safe Default)

#### What It Does
Gradually replaces old pods with new pods, one (or a few) at a time. Always keeps some pods running.

**Timeline:**
```
Old pods: Running Running Running Running Running
          Updating... Updating... Updating...
New pods: STOP    Ready   Ready    Ready    Ready Ready
Users:    Continuous service availability
```

#### When to Use
- Production environments (recommended default)
- Applications that need zero downtime
- When you want a gradual, safe rollout

#### Pros ✅
- **Zero downtime** - service always available
- Safe - if issues happen, only some users affected
- Can rollback quickly if needed
- No extra infrastructure cost

#### Cons ❌
- Takes longer than Recreate
- Two versions running simultaneously (can confuse debugging)
- More complex setup

#### How It Works
Kubernetes replaces pods gradually using these settings:

| Setting | What It Does | Example |
|---------|--------------|---------|
| **maxSurge** | Extra pods allowed during update | If maxSurge=1 and you have 3 pods, temporarily 4 pods run |
| **maxUnavailable** | Maximum pods that can be down | If maxUnavailable=1, at least 2 out of 3 pods always run |

#### Example Command
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Can add 1 extra pod during update
      maxUnavailable: 0    # Must keep all pods running
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-app:v2
```

#### Real-World Example
You have a web app with 3 pods running version 1.0:
1. Start 1 new pod with version 2.0 (now 4 pods running)
2. Stop 1 old pod with version 1.0 (now 3 pods: 2 old + 1 new)
3. Start another new pod (now 4 pods: 1 old + 2 new)
4. Stop another old pod (now 3 pods: 0 old + 3 new)
5. Done! All pods are now version 2.0

---

### 3. **Blue-Green Strategy** (The Instant Switch)

#### What It Does
Run two complete, identical environments (Blue = old version, Green = new version). Test the green environment completely, then instantly switch all traffic to it.

**Timeline:**
```
BLUE Environment:  v1.0 v1.0 v1.0 (All traffic)   (No traffic)
GREEN Environment: (No traffic)     (All traffic) v2.0 v2.0 v2.0

User Traffic: ──→ BLUE ───→ SWITCH ───→ GREEN ──→
```

#### When to Use
- Critical applications (banking, payment systems)
- When instant rollback is important
- When you need complete environment testing
- When downtime is not acceptable

#### Pros ✅
- **Instant switch** - no gradual migration
- **Instant rollback** - switch back to blue immediately if issues
- Complete environment testing before switch
- No traffic during deployment

#### Cons ❌
- **Double infrastructure cost** - pay for 2 full environments
- Takes more resources and time
- Complex setup

#### How It Works

**Step 1: Setup**
```
Blue (Old):  Running version 1.0, getting all traffic
Green (New): Not running yet
```

**Step 2: Deploy**
```
Blue (Old):  Running version 1.0, getting all traffic
Green (New): Deployed with version 2.0, not getting traffic yet
```

**Step 3: Test**
```
Blue (Old):  Running version 1.0, getting all traffic
Green (New): Running version 2.0, testing in progress
```

**Step 4: Switch**
```
Blue (Old):  Running version 1.0, NOT getting traffic anymore
Green (New): Running version 2.0, getting ALL traffic now
```

**Step 5: Cleanup (after confirming new version is stable)**
```
Blue (Old):  Stopped and removed
Green (New): Running version 2.0, getting all traffic
```

#### Example Setup
```yaml
# Blue Deployment (Old Version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: blue
  template:
    metadata:
      labels:
        app: my-app
        version: blue
    spec:
      containers:
      - name: my-app
        image: my-app:v1.0

---

# Green Deployment (New Version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: green
  template:
    metadata:
      labels:
        app: my-app
        version: green
    spec:
      containers:
      - name: my-app
        image: my-app:v2.0

---

# Service - Routes traffic to Blue initially
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
    version: blue  # Traffic goes to blue
  ports:
  - port: 80
    targetPort: 8080
```

**To switch traffic to Green:**
```bash
kubectl patch service my-app-service -p '{"spec":{"selector":{"version":"green"}}}'
```

---

### 4. **Canary Strategy** (The Smart Test)

#### What It Does
Route a small percentage of traffic to the new version while most users still use the old version. Gradually increase traffic to the new version as it proves stable.

**Timeline:**
```
Old Version (v1): 95% traffic ──→ 80% traffic ──→ 50% traffic ──→ 0%
New Version (v2): 5% traffic  ──→ 20% traffic ──→ 50% traffic ──→ 100%

Monitoring: Checking errors, latency, and performance constantly
```

#### When to Use
- High-visibility features (important to users)
- When you want to test with real users gradually
- Critical applications (testing in production safely)
- When you want to catch issues early

#### Pros ✅
- **Catches issues early** - only small group affected if problem
- **Real user testing** - see how real users interact with new version
- **Gradual rollout** - minimize risk
- **Easy rollback** - if issues appear, switch back

#### Cons ❌
- **Most complex** to setup and manage
- Need good monitoring to detect issues
- Takes longer than other strategies
- Need tools to manage traffic splitting

#### How It Works

**Phase 1: 5% to New Version**
```
New version gets 5% of traffic (1 in 20 users)
Monitoring watches: errors, latency, crashes
If issues found → rollback immediately
If stable → continue to phase 2
```

**Phase 2: 25% to New Version**
```
New version gets 25% of traffic (1 in 4 users)
More users testing the new version
Monitoring continues
```

**Phase 3: 50% to New Version**
```
New version gets 50% of traffic (1 in 2 users)
Half users on new version, half on old
Testing with larger group
```

**Phase 4: 100% to New Version**
```
All traffic goes to new version
Old version can be removed
```

#### Example Setup
```yaml
# Old Deployment (v1)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-v1
spec:
  replicas: 9  # 90% of traffic
  selector:
    matchLabels:
      app: my-app
      version: v1
  template:
    metadata:
      labels:
        app: my-app
        version: v1
    spec:
      containers:
      - name: my-app
        image: my-app:v1.0

---

# New Deployment (v2) - Canary
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-v2
spec:
  replicas: 1  # 10% of traffic
  selector:
    matchLabels:
      app: my-app
      version: v2
  template:
    metadata:
      labels:
        app: my-app
        version: v2
    spec:
      containers:
      - name: my-app
        image: my-app:v2.0

---

# Service - Routes to both versions
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app  # Both v1 and v2 match this
  ports:
  - port: 80
    targetPort: 8080
```

**To gradually increase traffic to v2:**
```bash
# Phase 1: 10% traffic to v2 (already done above)

# Phase 2: 25% traffic to v2 (scale v1 to 3, v2 to 1)
kubectl scale deployment my-app-v1 --replicas=3
kubectl scale deployment my-app-v2 --replicas=1

# Phase 3: 50% traffic to v2
kubectl scale deployment my-app-v1 --replicas=2
kubectl scale deployment my-app-v2 --replicas=2

# Phase 4: 100% traffic to v2
kubectl scale deployment my-app-v1 --replicas=0
kubectl scale deployment my-app-v2 --replicas=4
```

---

## 🎯 Quick Comparison Table

| Strategy | Speed | Safety | Cost | Downtime | Use Case |
|----------|-------|--------|------|----------|----------|
| **Recreate** | ⚡⚡⚡ Fastest | ❌ Low | 💰 Low | ⛔ Yes (full) | Dev/Testing |
| **Rolling** | ⚡⚡ Medium | ✅ High | 💰 Low | ✅ No | Production (default) |
| **Blue-Green** | ⚡ Slow | ✅ Very High | 💰💰 High | ✅ No | Critical apps |
| **Canary** | 🐢 Slowest | ✅✅ Highest | 💰 Low | ✅ No | High-visibility features |

---

## 🛡️ Best Practices

### 1. **Always Use Health Checks**
```yaml
spec:
  template:
    spec:
      containers:
      - name: my-app
        livenessProbe:           # Is the app alive?
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:          # Is the app ready for traffic?
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

### 2. **Set Resource Requests and Limits**
```yaml
spec:
  template:
    spec:
      containers:
      - name: my-app
        resources:
          requests:
            cpu: "100m"          # Minimum resources needed
            memory: "128Mi"
          limits:
            cpu: "500m"          # Maximum resources allowed
            memory: "512Mi"
```

### 3. **Monitor Your Deployments**
- Watch error rates
- Check latency and response times
- Monitor CPU and memory usage
- Have alerting in place

### 4. **Test Before Production**
- Test in staging environment first
- Use same deployment strategy in staging as production
- Verify health checks work correctly

### 5. **Plan Your Rollback**
- Know how to quickly rollback to previous version
- Test rollback procedures
- Keep previous version running if possible

---

## 📋 Decision Tree: Which Strategy to Use?

```
START: Need to deploy new version?
  ↓
Is downtime acceptable?
  ├─ YES
  │   ↓
  │   Is this development environment?
  │   ├─ YES → Use RECREATE (simple, fast)
  │   └─ NO
  │       ↓
  │       Is infrastructure cost a concern?
  │       ├─ YES → Use ROLLING (safe, cost-effective)
  │       └─ NO → Use BLUE-GREEN (instant rollback, safe)
  │
  └─ NO (Zero downtime required)
      ↓
      Is this a critical/high-visibility feature?
      ├─ YES → Use CANARY (test with real users safely)
      └─ NO → Use ROLLING (standard safe approach)
```

---

## 🔧 Useful Commands

### View current deployment strategy
```bash
kubectl get deployment my-app -o yaml | grep -A 10 strategy
```

### Update deployment image
```bash
kubectl set image deployment/my-app my-app=my-app:v2.0
```

### Watch deployment progress
```bash
kubectl rollout status deployment/my-app
```

### Rollback to previous version
```bash
kubectl rollout undo deployment/my-app
```

### View rollout history
```bash
kubectl rollout history deployment/my-app
```

### Check pod status
```bash
kubectl get pods -o wide
```

---

## 📚 Learning Resources

| Topic | Resource |
|-------|----------|
| Rolling Deployments | [Kubernetes Official Docs](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/) |
| Blue-Green Pattern | [Martin Fowler - Blue Green Deployment](https://martinfowler.com/bliki/BlueGreenDeployment.html) |
| Canary Releases | [Flagger - Automated Canary Releases](https://flagger.app/) |
| Practice Labs | Deploy to local cluster with `minikube` |

---

## 🎓 Summary

| Strategy | Best For | Remember |
|----------|----------|----------|
| **Recreate** | Simple environments | Causes downtime |
| **Rolling** | Most production apps | Default Kubernetes strategy |
| **Blue-Green** | Mission-critical apps | Costs 2x infrastructure |
| **Canary** | Important features | Most sophisticated, catch bugs early |

Choose the strategy based on your app's requirements for **availability, risk tolerance, and resources**.

---

## 💡 Key Takeaways

1. **Rolling deployment is the safe default** for most production applications
2. **Canary deployment is the smartest** if you want to test with real users
3. **Blue-Green is best** when you need instant rollback
4. **Recreate is only for development** because it causes downtime
5. **Always monitor** your deployments to catch issues early
6. **Health checks are essential** for all strategies

---

## ❓ FAQ

**Q: Can I use multiple strategies?**
A: Yes! Many teams use canary first, then blue-green for critical switches.

**Q: What's the default strategy in Kubernetes?**
A: Rolling updates are the default.

**Q: How do I test a canary deployment locally?**
A: Use `minikube` or `kind` to create a local Kubernetes cluster and practice.

**Q: What if my application needs persistent data?**
A: Use StatefulSets and be careful with rolling updates. Test thoroughly!

**Q: Can I automate this?**
A: Yes! Tools like Helm, Flux, and ArgoCD can automate deployments.

---

## 🚀 Next Steps

1. **Practice** each strategy in a local Kubernetes cluster
2. **Choose** the right strategy for your application
3. **Implement** health checks and monitoring
4. **Test** your rollback procedures
5. **Document** your deployment process
6. **Automate** with CI/CD tools like GitLab CI, GitHub Actions, or Jenkins

---

**Happy Deploying! 🎉**

