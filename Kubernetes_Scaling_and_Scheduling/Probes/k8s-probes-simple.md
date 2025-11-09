# Kubernetes Probes - Simple Guide

## What are Probes?

**Probes** are health checks that Kubernetes runs on containers to make sure they're working properly. Think of them as automatic checks that monitor your application and take action if something goes wrong.

Kubernetes has **three types of probes**:

1. **Startup Probe** - Checks if the app started
2. **Readiness Probe** - Checks if the app is ready for traffic
3. **Liveness Probe** - Checks if the app is still working

---

## The Three Probes Explained

### 1. Startup Probe - "Did it start?"

**What it does:** Checks if your application has started successfully

**When to use:** Only for apps that take a long time to start (like Java apps)

**What happens if it fails:** Kubernetes keeps trying. If it fails too many times, the container restarts.

**Once it passes:** Other probes take over

**Example:** A Spring Boot app takes 30 seconds to start

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 0
  periodSeconds: 10
  failureThreshold: 30  # Allow 30 failures (30 * 10 = 300 seconds = 5 minutes)
```

---

### 2. Readiness Probe - "Is it ready for traffic?"

**What it does:** Checks if your app is ready to receive requests

**When to use:** ALWAYS add this for web apps and APIs

**What happens if it fails:** Kubernetes removes the pod from traffic. No requests go to it.

**What happens when it passes:** Kubernetes sends traffic to the pod

**Example:** Database connection test

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 2  # After 2 failures, stop sending traffic
```

---

### 3. Liveness Probe - "Is it still working?"

**What it does:** Checks if your app is still alive and healthy

**When to use:** For apps that can freeze or deadlock

**What happens if it fails:** Kubernetes **restarts the container**

**How often:** Periodically during the lifetime of the container

**Example:** Checking if the app process is responsive

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
  failureThreshold: 3  # After 3 failures, restart
```

---

## Probe Types - How to Check Health

Kubernetes can check health in three ways:

### 1. HTTP Probe (Most Common)

Sends an HTTP GET request to your app and checks the response code.

```yaml
readinessProbe:
  httpGet:
    path: /health           # URL path to check
    port: 8080              # Container port
    scheme: HTTP            # HTTP or HTTPS
  initialDelaySeconds: 5
  periodSeconds: 10
```

**When to use:** Web apps, APIs, microservices

---

### 2. TCP Probe

Tries to open a TCP connection to a port. If successful, the app is healthy.

```yaml
readinessProbe:
  tcpSocket:
    port: 3306              # Database port
  initialDelaySeconds: 5
  periodSeconds: 10
```

**When to use:** Databases, cache servers (Redis, Memcached)

---

### 3. Exec Probe

Runs a command inside the container. If exit code is 0, it's healthy.

```yaml
readinessProbe:
  exec:
    command:
    - /bin/sh
    - -c
    - mysql -h 127.0.0.1 -e 'SELECT 1'  # MySQL check
  initialDelaySeconds: 5
  periodSeconds: 10
```

**When to use:** Complex health checks, custom scripts

---

## Probe Parameters Explained

| Parameter | What it means | Default | Example |
|-----------|--------------|---------|---------|
| **initialDelaySeconds** | Wait this many seconds before first check | 0 | 5 (wait 5 seconds) |
| **periodSeconds** | Check every this many seconds | 10 | 10 (check every 10s) |
| **timeoutSeconds** | Wait this long for response | 1 | 3 (wait max 3 seconds) |
| **failureThreshold** | Restart after this many failures | 3 | 3 (fail 3 times then restart) |
| **successThreshold** | Pass after this many successes | 1 | 1 (pass on first success) |

---

## Practical Examples

### Example 1: Simple Web App (Nginx)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
    
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 15
      periodSeconds: 20
```

---

### Example 2: Node.js App

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodejs-pod
spec:
  containers:
  - name: app
    image: my-node-app:latest
    ports:
    - containerPort: 3000
    
    startupProbe:
      httpGet:
        path: /health
        port: 3000
      failureThreshold: 30
      periodSeconds: 10
    
    readinessProbe:
      httpGet:
        path: /ready
        port: 3000
      initialDelaySeconds: 5
      periodSeconds: 5
    
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 15
      periodSeconds: 30
```

---

### Example 3: Database (MySQL with TCP)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
spec:
  containers:
  - name: mysql
    image: mysql:8
    ports:
    - containerPort: 3306
    
    readinessProbe:
      tcpSocket:
        port: 3306
      initialDelaySeconds: 10
      periodSeconds: 5
    
    livenessProbe:
      tcpSocket:
        port: 3306
      initialDelaySeconds: 20
      periodSeconds: 10
```

---

### Example 4: App with Custom Script

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-app
spec:
  containers:
  - name: app
    image: custom-app:latest
    
    readinessProbe:
      exec:
        command:
        - /bin/sh
        - -c
        - test -f /tmp/ready || exit 1
      initialDelaySeconds: 5
      periodSeconds: 10
```

---

## Common Mistakes to Avoid

❌ **Mistake 1:** No probes at all  
✅ **Fix:** Always add readiness probe for web apps

---

❌ **Mistake 2:** initialDelaySeconds too small  
✅ **Fix:** Set it higher than app startup time

---

❌ **Mistake 3:** Liveness probe checks external services  
Example: ❌ Checking if database is up  
✅ **Do:** Only check app's own health

---

❌ **Mistake 4:** Same probe for startup and liveness  
✅ **Fix:** Use different endpoints or thresholds

---

❌ **Mistake 5:** No timeout set  
✅ **Fix:** Always set timeoutSeconds

---

## When to Use Each Probe

### Use Startup Probe when:
- App takes more than 10 seconds to start
- App is slow at initialization (Spring Boot, .NET)

### Use Readiness Probe when:
- Web app or API (YES, always!)
- App loads data on startup
- App depends on external services

### Use Liveness Probe when:
- App can deadlock or freeze
- App needs automatic restart on crash
- Long-running services

---

## Checking Probe Status

### View probe status
```bash
kubectl describe pod <pod-name>
```

Look for "Readiness:" and "Liveness:" in output

### Check if pod is ready
```bash
kubectl get pods
# Look at READY column
# Example: 1/1 = ready, 0/1 = not ready
```

### View probe events
```bash
kubectl describe pod <pod-name>
# Look at "Events" section
```

---

## Quick Decision Guide

**Does your app take time to start?**
- Yes → Add Startup Probe
- No → Skip Startup Probe

**Is it a web app or API?**
- Yes → Add Readiness Probe
- No → Maybe Readiness Probe for databases

**Can your app freeze or need restart?**
- Yes → Add Liveness Probe
- No → Optional

---

## Important Notes

1. **Readiness is more important than Liveness** - Readiness prevents broken requests
2. **Keep probes simple** - Simple checks are more reliable
3. **Test your health endpoints** - Make sure `/health` actually works
4. **Don't probe external services** - Only check your app's health
5. **Use Deployment, not Pod** - Probes work better with Deployments

---

## Example Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
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
      - name: app
        image: my-web-app:latest
        ports:
        - containerPort: 8080
        
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
```

---

## Key Takeaways

✅ **Always use Readiness Probe** for web apps  
✅ **Use initialDelaySeconds** to allow app startup time  
✅ **Keep probes simple** - don't check external services  
✅ **Check app status** with `kubectl describe pod`  
✅ **Use Startup Probe** for slow-starting apps  

---

**Version:** 1.0  
**Best for:** Kubernetes beginners  
**Last Updated:** November 2025