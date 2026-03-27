
# 🚀 Kubernetes Ingress — In-Depth Guide

> **Learn by scenario. Understand by doing.**

---

## 📚 Table of Contents

1. [The Scenario — Why Do We Need Ingress?](#1-the-scenario--why-do-we-need-ingress)
2. [What Is Kubernetes Ingress?](#2-what-is-kubernetes-ingress)
3. [Benefits of Using Ingress](#3-benefits-of-using-ingress)
4. [Core Concepts](#4-core-concepts)
   - Ingress Resource
   - Ingress Controller
   - IngressClass
5. [Architecture Diagram](#5-architecture-diagram)
6. [How to Use Ingress in Kubernetes](#6-how-to-use-ingress-in-kubernetes)
   - Install an Ingress Controller (NGINX)
   - Deploy Sample Apps
   - Create an Ingress Resource
7. [Routing Rules Deep Dive](#7-routing-rules-deep-dive)
   - Path-Based Routing
   - Host-Based Routing
   - Default Backend
8. [TLS / HTTPS with Ingress](#8-tls--https-with-ingress)
9. [Annotations — Fine-Tuning Behavior](#9-annotations--fine-tuning-behavior)
10. [IngressClass — Multiple Controllers](#10-ingressclass--multiple-controllers)
11. [Common Ingress Controllers Compared](#11-common-ingress-controllers-compared)
12. [Real-World Practical Examples](#12-real-world-practical-examples)
13. [Debugging & Troubleshooting](#13-debugging--troubleshooting)
14. [Quick Reference Cheat Sheet](#14-quick-reference-cheat-sheet)

---

## 1. The Scenario — Why Do We Need Ingress?

### 🏗️ Imagine You're Building a SaaS App

You've deployed three microservices in your Kubernetes cluster:

| Service        | Purpose              |
|----------------|----------------------|
| `frontend`     | React web UI         |
| `api`          | REST API backend     |
| `auth`         | Authentication       |

**Problem:** How do users from the internet reach these services?

### ❌ Option A: NodePort for Every Service

```
User → NodePort :30001 → frontend
User → NodePort :30002 → api
User → NodePort :30003 → auth
```

- Users have to remember ugly port numbers like `myapp.com:30002`
- Not production-friendly
- No SSL termination

### ❌ Option B: LoadBalancer for Every Service

```
User → LoadBalancer-1 ($$) → frontend
User → LoadBalancer-2 ($$) → api
User → LoadBalancer-3 ($$) → auth
```

- Each LoadBalancer costs money (especially on AWS/GCP/Azure)
- 10 services = 10 Load Balancers = 💸💸💸

### ✅ Option C: Kubernetes Ingress (The Right Way)

```
User → 1 LoadBalancer → Ingress Controller → Routes to correct service
```

- **One entry point** for all traffic
- Route based on URL paths or hostnames
- **SSL/TLS in one place**
- **Cost-efficient** — one cloud LoadBalancer for everything

---

## 2. What Is Kubernetes Ingress?

> **Ingress** is a Kubernetes API object that manages **external HTTP/HTTPS access** to services inside the cluster.

It acts like a **smart router / reverse proxy** sitting at the edge of your cluster.

Think of it like a **hotel receptionist**:
- You walk in (HTTP request hits the cluster)
- The receptionist looks at your name/room request (URL/hostname)
- They direct you to the right room (service)

### Two Parts You Must Know

```
┌─────────────────────────────────────────────┐
│              Ingress = Two Things            │
│                                             │
│  1. Ingress Resource  →  The RULES (config) │
│  2. Ingress Controller → The ENGINE (runs)  │
└─────────────────────────────────────────────┘
```

**Ingress Resource** — A YAML file you write that defines routing rules.

**Ingress Controller** — A pod running inside the cluster that reads those rules and actually does the routing (NGINX, Traefik, HAProxy, etc.).

> ⚠️ **Critical:** Kubernetes does NOT ship with an Ingress Controller by default. You must install one.

---

## 3. Benefits of Using Ingress

| Benefit                    | Description                                                          |
|----------------------------|----------------------------------------------------------------------|
| 💰 **Cost Savings**         | One LoadBalancer instead of one per service                         |
| 🔒 **TLS Termination**      | Handle HTTPS in one place, services can use plain HTTP internally   |
| 🧭 **URL Routing**          | Route `/api` to backend, `/` to frontend — all from one IP         |
| 🌐 **Virtual Hosting**      | `app1.com` → Service A, `app2.com` → Service B on same cluster     |
| 🔁 **Load Balancing**       | Built-in L7 load balancing with health checks                       |
| 🛡️ **Security**             | Add rate limiting, auth, IP whitelisting via annotations            |
| 📋 **Centralized Config**   | All routing logic in one place — easy to manage                     |
| 🔌 **Pluggable**            | Swap controllers (NGINX ↔ Traefik ↔ Istio) without changing rules  |

---

## 4. Core Concepts

### 4.1 Ingress Resource

The **Ingress Resource** is a Kubernetes object (YAML) where you write routing rules.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

Think of it as a **routing table** — just configuration, no logic.

---

### 4.2 Ingress Controller

The **Ingress Controller** is the actual component that:

- Watches the Kubernetes API for Ingress resources
- Configures itself (e.g., updates NGINX config)
- Forwards traffic to the right services

```
                   ┌──────────────────────────┐
                   │    Kubernetes API Server  │
                   └────────────┬─────────────┘
                                │ watches Ingress objects
                   ┌────────────▼─────────────┐
                   │    Ingress Controller     │
                   │  (e.g., NGINX pod)        │
                   │                          │
                   │  Reads rules → Configures │
                   │  its own routing logic    │
                   └────────────┬─────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
        frontend-svc        api-svc           auth-svc
```

**Popular Ingress Controllers:**

| Controller      | Best For                     |
|-----------------|------------------------------|
| NGINX           | General-purpose, most popular |
| Traefik         | Cloud-native, auto-discovery |
| HAProxy         | High performance             |
| AWS ALB         | AWS-native deployments       |
| GKE Ingress     | Google Cloud                 |
| Istio           | Service mesh + ingress       |
| Contour         | Envoy-based                  |

---

### 4.3 IngressClass

Used when you have **multiple controllers** installed. It tells an Ingress resource which controller to use.

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
spec:
  controller: k8s.io/ingress-nginx
```

Reference it in your Ingress:

```yaml
spec:
  ingressClassName: nginx   # ← tells this Ingress to use the NGINX controller
```

---

## 5. Architecture Diagram

```
                         INTERNET
                            │
                            ▼
               ┌────────────────────────┐
               │   Cloud Load Balancer  │  ← 1 per cluster (not per service!)
               │   (AWS ELB / GCP LB)  │
               └────────────┬───────────┘
                            │
               ┌────────────▼───────────┐
               │    Ingress Controller  │  ← Runs as a Pod in k8s
               │    (e.g., NGINX)       │
               │                        │
               │  Rules from Ingress    │
               │  Resource applied here │
               └──┬─────────┬───────────┘
                  │         │
         path:/   │         │  path:/api
                  ▼         ▼
          ┌───────────┐ ┌───────────┐
          │ frontend  │ │    api    │
          │  Service  │ │  Service  │
          └─────┬─────┘ └─────┬─────┘
                │             │
         ┌──────▼───┐   ┌─────▼────┐
         │ Pod  Pod │   │ Pod  Pod │
         └──────────┘   └──────────┘
```

---

## 6. How to Use Ingress in Kubernetes

### Step 1: Install NGINX Ingress Controller

```bash
# Using Helm (recommended)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# Verify it's running
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

You should see a **LoadBalancer** service with an external IP:

```
NAME                       TYPE           EXTERNAL-IP
ingress-nginx-controller   LoadBalancer   203.0.113.10
```

---

### Step 2: Deploy Sample Apps

```yaml
# frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

```yaml
# api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo
        args: ["-text=Hello from API"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 5678
```

Apply them:

```bash
kubectl apply -f frontend-deployment.yaml
kubectl apply -f api-deployment.yaml
```

---

### Step 3: Create an Ingress Resource

```yaml
# my-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

```bash
kubectl apply -f my-ingress.yaml

# Check status
kubectl get ingress
kubectl describe ingress my-app-ingress
```

---

## 7. Routing Rules Deep Dive

### 7.1 Path-Based Routing

Route different URL paths to different services.

```yaml
rules:
- host: myapp.com
  http:
    paths:
    - path: /          # → frontend
      pathType: Prefix
      backend:
        service:
          name: frontend-service
          port:
            number: 80
    - path: /api       # → api backend
      pathType: Prefix
      backend:
        service:
          name: api-service
          port:
            number: 80
    - path: /auth      # → auth service
      pathType: Prefix
      backend:
        service:
          name: auth-service
          port:
            number: 80
```

**PathType options:**

| PathType      | Behavior                                        |
|---------------|-------------------------------------------------|
| `Prefix`      | Matches if URL **starts with** the path         |
| `Exact`       | Matches **only** if URL is exactly the path     |
| `ImplementationSpecific` | Depends on the controller           |

---

### 7.2 Host-Based Routing

Route different domain names to different services (virtual hosting).

```yaml
rules:
# First app — app1.com
- host: app1.com
  http:
    paths:
    - path: /
      pathType: Prefix
      backend:
        service:
          name: app1-service
          port:
            number: 80

# Second app — app2.com
- host: app2.com
  http:
    paths:
    - path: /
      pathType: Prefix
      backend:
        service:
          name: app2-service
          port:
            number: 80
```

**Flow:**

```
app1.com ──────────────────► app1-service
                Ingress
app2.com ──────────────────► app2-service
```

---

### 7.3 Default Backend

What happens when no rules match? Define a fallback:

```yaml
spec:
  defaultBackend:
    service:
      name: default-service
      port:
        number: 80
  rules:
  - ...
```

---

## 8. TLS / HTTPS with Ingress

### Step 1: Create a TLS Secret

```bash
# Generate self-signed cert (for testing)
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=myapp.com"

# Create Kubernetes secret
kubectl create secret tls myapp-tls \
  --cert=tls.crt \
  --key=tls.key
```

### Step 2: Reference in Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.com
    secretName: myapp-tls       # ← your TLS secret
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### Step 3: Auto TLS with cert-manager (Production)

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml

# Create ClusterIssuer for Let's Encrypt
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your@email.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

Then add to your Ingress:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - myapp.com
    secretName: myapp-tls-auto   # cert-manager creates this automatically
```

---

## 9. Annotations — Fine-Tuning Behavior

Annotations let you control the controller's behavior without changing the spec.

### Common NGINX Annotations

```yaml
metadata:
  annotations:
    # Rewrite URL before forwarding
    nginx.ingress.kubernetes.io/rewrite-target: /

    # Force HTTPS redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"

    # Enable CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"

    # Proxy timeout
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"

    # Max body size
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"

    # Basic auth
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth

    # Whitelist specific IPs
    nginx.ingress.kubernetes.io/whitelist-source-range: "192.168.1.0/24"

    # Session affinity (sticky sessions)
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "INGRESSCOOKIE"
```

---

## 10. IngressClass — Multiple Controllers

When you have multiple Ingress Controllers installed (e.g., NGINX for apps, AWS ALB for APIs):

```yaml
# ingress-class-nginx.yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"  # ← make default
spec:
  controller: k8s.io/ingress-nginx
```

```yaml
# ingress-class-alb.yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: alb
spec:
  controller: ingress.k8s.aws/alb
```

Reference in your Ingress:

```yaml
spec:
  ingressClassName: alb   # use AWS ALB controller for this ingress
```

---

## 11. Common Ingress Controllers Compared

| Feature             | NGINX        | Traefik      | AWS ALB      | Istio        |
|---------------------|--------------|--------------|--------------|--------------|
| Ease of setup       | ⭐⭐⭐⭐⭐      | ⭐⭐⭐⭐       | ⭐⭐⭐        | ⭐⭐          |
| Performance         | ⭐⭐⭐⭐       | ⭐⭐⭐⭐       | ⭐⭐⭐⭐⭐     | ⭐⭐⭐⭐       |
| Auto TLS            | With cert-manager | Built-in | Via ACM     | Built-in     |
| Cloud-native        | ✅            | ✅            | AWS only    | ✅            |
| Service mesh        | ❌            | Partial       | ❌           | ✅ Full       |
| Dashboard           | ❌            | ✅ Built-in   | AWS Console | ✅ Kiali      |
| Community           | Very Large   | Large        | AWS-backed  | Large        |
| Best for            | General use  | Cloud-native | AWS EKS     | Microservices|

---

## 12. Real-World Practical Examples

### Example 1: E-commerce App (Multi-service)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"   # for file uploads
    nginx.ingress.kubernetes.io/limit-rps: "100"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - shop.mystore.com
    secretName: shop-tls
  rules:
  - host: shop.mystore.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: storefront-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: product-api-service
            port:
              number: 8080
      - path: /checkout
        pathType: Prefix
        backend:
          service:
            name: checkout-service
            port:
              number: 3000
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-panel-service
            port:
              number: 4000
```

---

### Example 2: Multi-Tenant SaaS (Host-Based)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multitenant-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - tenant1.saas.com
    - tenant2.saas.com
    secretName: wildcard-tls
  rules:
  - host: tenant1.saas.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tenant1-service
            port:
              number: 80
  - host: tenant2.saas.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tenant2-service
            port:
              number: 80
```

---

### Example 3: Canary Deployment (A/B Traffic Split)

```yaml
# Main ingress (90% traffic)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v1-service
            port:
              number: 80
---
# Canary ingress (10% traffic)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary-ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"   # 10% traffic
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v2-service
            port:
              number: 80
```

---

## 13. Debugging & Troubleshooting

### Check Ingress Status

```bash
# List all ingresses
kubectl get ingress

# Detailed view with events
kubectl describe ingress my-app-ingress

# Check if IP/hostname is assigned
kubectl get ingress my-app-ingress -o wide
```

### Check Controller Logs

```bash
# Get controller pod name
kubectl get pods -n ingress-nginx

# Tail logs
kubectl logs -n ingress-nginx ingress-nginx-controller-xxxx -f
```

### Common Problems & Fixes

| Problem | Symptom | Fix |
|---------|---------|-----|
| No ADDRESS assigned | `kubectl get ingress` shows empty ADDRESS | Check if LoadBalancer has external IP, check controller pods |
| 404 Not Found | Traffic reaches controller but wrong service | Check path rules and service names |
| 503 Service Unavailable | Service found but pods not ready | Check pod health, service selectors |
| TLS not working | HTTP instead of HTTPS | Check TLS secret, verify cert-manager |
| Wrong service routing | Traffic goes to wrong backend | Check path specificity (Exact vs Prefix) |
| Controller not starting | Pod CrashLoopBackOff | Check RBAC permissions, resource limits |

### Verify Service Connection

```bash
# Test from inside the cluster
kubectl run test --image=busybox --rm -it -- wget -O- http://frontend-service

# Check endpoints exist
kubectl get endpoints frontend-service
```

### Inspect NGINX Config (inside controller)

```bash
kubectl exec -n ingress-nginx ingress-nginx-controller-xxxx -- nginx -T
```

---

## 14. Quick Reference Cheat Sheet

```bash
# ── INSTALL ──────────────────────────────────────────────
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace

# ── VIEW ─────────────────────────────────────────────────
kubectl get ingress                        # list all
kubectl describe ingress <name>            # detailed info
kubectl get ingressclass                   # list ingress classes

# ── APPLY / DELETE ───────────────────────────────────────
kubectl apply -f ingress.yaml
kubectl delete ingress <name>

# ── TLS ──────────────────────────────────────────────────
kubectl create secret tls <name> --cert=tls.crt --key=tls.key
kubectl get secret <name> -o yaml

# ── DEBUG ────────────────────────────────────────────────
kubectl logs -n ingress-nginx <controller-pod> -f
kubectl exec -n ingress-nginx <pod> -- nginx -T
kubectl get events --field-selector reason=Sync -n ingress-nginx
```

---

## Summary — Mental Model

```
You Write:    Ingress Resource  (YAML rules — "route /api to api-service")
              ↓
K8s Stores:   etcd             (saves your config)
              ↓
Controller    Watches + Acts   (NGINX reads rules, reconfigures itself)
Reads It:     ↓
Traffic       Ingress Controller → routes to correct Service → Pod
Flows:
```

> 🎯 **Key Takeaway:** Ingress = Rules (Resource) + Engine (Controller). You write the rules. The controller enforces them.

---

*Generated with ❤️ for Kubernetes learners. Practice, break things, fix things — that's how you learn!*
