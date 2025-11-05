
----
# Kubernetes Ingress

## 📋 Table of Contents
- [Real-World Problem Scenario](#real-world-problem-scenario)
- [What Is Kubernetes Ingress?](#what-is-kubernetes-ingress)
- [Why Kubernetes Ingress Matters](#why-kubernetes-ingress-matters)
- [Ingress vs LoadBalancer vs NodePort](#ingress-vs-loadbalancer-vs-nodeport)
- [Ingress Architecture](#ingress-architecture)
- [Core Components](#core-components)
- [How Ingress Works](#how-ingress-works)
- [Ingress Resource YAML](#ingress-resource-yaml)
- [Ingress Rules Types](#ingress-rules-types)
- [Path Routing](#path-routing)
- [Host-Based Routing](#host-based-routing)
- [TLS/SSL Configuration](#tlsssl-configuration)
- [Ingress Controllers](#ingress-controllers)
- [Real-World Use Cases](#real-world-use-cases)
- [Practical Examples](#practical-examples)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)
- [Important Operational Notes](#important-operational-notes)
- [Quick Reference](#quick-reference)

---

## 🚨 Real-World Problem Scenario

### The Problem

You have a microservices application with multiple services running in Kubernetes:

- **Restaurant API** - serves food ordering
- **Instamart API** - serves grocery shopping
- **Monitoring Dashboard** - internal analytics
- **Payment Service** - handles transactions

**Challenges:**

1. **Multiple Load Balancers** - Creating a LoadBalancer Service for each service is expensive (each gets its own public IP)
2. **Traffic Management** - Need to route different URLs/paths to different services
3. **SSL/TLS** - Need HTTPS for security, but managing certificates on each service is complex
4. **Single Entry Point** - Want users to access everything through one domain (example.com)

### Without Ingress (Using Multiple LoadBalancers)

```
example.com (LB1) → restaurant-service
api.example.com (LB2) → api-service
shop.example.com (LB3) → shop-service
dash.example.com (LB4) → dashboard-service

❌ 4 separate public IPs
❌ 4 separate load balancers
❌ Each costs money
❌ DNS management nightmare
❌ Certificate management for each domain
```

### With Ingress (Single Entry Point)

```
Load Balancer (1 public IP) → Ingress Controller (1 entry point)
                              ↓
                         Routes based on rules:
                         • example.com → restaurant-service
                         • example.com/api → api-service
                         • example.com/shop → shop-service
                         • example.com/dashboard → dashboard-service

✅ Single public IP
✅ Single load balancer
✅ One certificate for all services
✅ Easy to manage
✅ Cost-effective
```

---

## 🔍 What Is Kubernetes Ingress?

**Ingress** is a Kubernetes API object that manages external HTTP/HTTPS access to services within a cluster. It acts as a smart router and load balancer at the edge of your cluster, routing incoming traffic to appropriate backend services based on configurable rules.

### Key Points:[web:84][web:91][web:86][web:93]

- **Single Entry Point** - All external traffic enters through one Ingress
- **Rule-Based Routing** - Routes based on hostname, path, or headers
- **Layer 7 (Application Layer)** - Works at HTTP/HTTPS level (unlike LoadBalancer at Layer 4)
- **Resource + Controller** - Ingress resource defines rules, Ingress Controller executes them
- **Cost-Effective** - One load balancer serves multiple services

---

## 🤔 Why Kubernetes Ingress Matters

### Problems It Solves:[web:94][web:97][web:87]

1. **Multiple Load Balancers**
   - ❌ Each LoadBalancer service gets its own IP (expensive)
   - ✅ Ingress uses one load balancer for all services

2. **Advanced Routing**
   - ❌ LoadBalancer can only route to one service
   - ✅ Ingress routes to multiple services based on host/path

3. **SSL/TLS Management**
   - ❌ Each service manages its own certificates
   - ✅ Ingress terminates TLS centrally

4. **Domain Management**
   - ❌ Multiple domains for different services
   - ✅ Multiple services under one domain

5. **Load Balancing**
   - Distributes traffic across multiple pod replicas
   - Ensures high availability and scalability

6. **Name-Based Virtual Hosting**
   - Multiple services accessed through different hostnames on same IP

---

## 📊 Ingress vs LoadBalancer vs NodePort

| Feature | NodePort | LoadBalancer | Ingress |
|---------|----------|--------------|---------|
| **Access Method** | Node IP:Port | Cloud LB IP | Domain/DNS |
| **Layer** | Layer 4 (Transport) | Layer 4 (Transport) | Layer 7 (Application) |
| **Port Range** | 30000-32767 | Any port | 80, 443 |
| **Services Per LB** | 1 (per port) | 1 | Multiple |
| **Cost** | Free | 💰 Per LB | 💰💰 Single LB for all |
| **Routing** | None (port-based) | IP-based | Host/path-based |
| **TLS Support** | ❌ No | ✅ Yes | ✅ Yes |
| **Use Case** | Development | Simple apps | Production, microservices |
| **External IP** | Node IP | Cloud LB IP | Load Balancer IP → Ingress |
| **Example** | 192.168.1.10:32000 | 1.2.3.4:80 | app.example.com |

### When to Use What:

**NodePort:**
- ✅ Development/testing
- ✅ Debugging
- ✅ Internal services

**LoadBalancer:**
- ✅ Single simple service
- ✅ Non-HTTP protocols (TCP/UDP)
- ✅ When you need direct IP access

**Ingress:**
- ✅ Multiple HTTP/HTTPS services
- ✅ Production microservices
- ✅ Complex routing rules
- ✅ Cost optimization

---

## 🏗️ Ingress Architecture

### Complete Traffic Flow

<img width="1264" height="809" alt="SCR-20251105-rbds" src="https://github.com/user-attachments/assets/9bfb5c3e-d170-4dd0-beba-9c52a0784876" />


### Request Path:

1. **Client Request** → DNS resolves `example.com` to Load Balancer IP
2. **Load Balancer** → Forwards to Ingress Controller pod
3. **Ingress Controller** → Reads request hostname/path
4. **Route Matching** → Finds matching Ingress rule
5. **Service Selection** → Routes to appropriate Service
6. **Pod Selection** → Service load balances to pod replicas
7. **Response** → Pod responds back through same path

---

## 🔑 Core Components

### 1. Ingress Resource

The YAML definition of routing rules. This is what you create.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

### 2. Ingress Controller

The actual component that executes the Ingress rules. This is what you deploy.

Popular Ingress Controllers:[web:96][web:104][web:106]
- **NGINX** - Most popular, simple, reliable
- **Traefik** - Modern, dynamic, easy to configure
- **HAProxy** - High performance, battle-tested
- **Kong** - API gateway + Ingress
- **AWS ALB** - AWS-specific, integrates with ALB
- **Istio Ingress** - Service mesh integration

### 3. IngressClass

Specifies which controller should handle an Ingress.

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
spec:
  controller: k8s.io/ingress-nginx
```

### 4. Backend Service

The Kubernetes Service that Ingress routes to.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

---

## ⚙️ How Ingress Works

### Step-by-Step Workflow:[web:93][web:95][web:113]

**Step 1: Deploy Ingress Controller**
Admin deploys ingress controller (e.g., NGINX) to cluster.

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx
```

**Step 2: Create Ingress Resource**
Developer defines routing rules in Ingress YAML.

```bash
kubectl apply -f ingress.yaml
```

**Step 3: Controller Watches**
Ingress Controller watches for new/updated Ingress resources.

**Step 4: Configuration Update**
Controller reads Ingress rules and updates its configuration (nginx.conf, Traefik config, etc.)

**Step 5: Create Load Balancer Service**
Controller creates a Service of type LoadBalancer (if cloud provider).

**Step 6: Get External IP**
Cloud provider provisions load balancer and assigns public IP.

**Step 7: Route Traffic**
- Client requests `example.com/api`
- DNS resolves to Load Balancer IP
- Load Balancer forwards to Ingress Controller
- Controller matches request to Ingress rule
- Routes to appropriate Service
- Service forwards to Pod

**Step 8: Response**
Pod responds, traffic returns through same path.

### Controller Monitors Continuously

Ingress Controller continuously watches for:
- New Ingress resources created
- Existing Ingress rules changed
- Ingress resources deleted
- Updates configuration dynamically without restart

---

## 📝 Ingress Resource YAML

### Complete YAML Structure

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: default
  labels:
    app: my-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  # Which controller should handle this ingress
  ingressClassName: nginx
  
  # TLS/SSL certificates
  tls:
  - hosts:
    - example.com
    - api.example.com
    secretName: my-tls-secret
  
  # Routing rules
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

### Field Explanations

| Field | Purpose | Required |
|-------|---------|----------|
| **apiVersion** | API version | ✅ Yes |
| **kind** | Resource type (Ingress) | ✅ Yes |
| **metadata.name** | Ingress name | ✅ Yes |
| **metadata.namespace** | Kubernetes namespace | ❌ No (default) |
| **spec.ingressClassName** | Controller to use | ✅ Yes |
| **spec.tls** | SSL certificates | ❌ No (HTTP only) |
| **spec.tls[].hosts** | Domains covered by cert | For TLS |
| **spec.tls[].secretName** | Secret with certificate | For TLS |
| **spec.rules[].host** | Hostname to match | ✅ Yes |
| **spec.rules[].http.paths[].path** | URL path to match | ✅ Yes |
| **spec.rules[].http.paths[].pathType** | How to match path | ✅ Yes |
| **spec.rules[].http.paths[].backend.service.name** | Service to route to | ✅ Yes |
| **spec.rules[].http.paths[].backend.service.port.number** | Service port | ✅ Yes |

---

## 🔀 Ingress Rules Types

### 1. Single Service Ingress

Routes all traffic to one service (simplest).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: single-service
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

**Use Case:**
- Simple application with one service
- No complex routing needed

### 2. Simple Fanout (Path-Based Routing)

Routes to different services based on URL path.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fanout
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /shop
        pathType: Prefix
        backend:
          service:
            name: shop-service
            port:
              number: 8080
```

**Routing:**
- `example.com/` → web-service
- `example.com/api` → api-service
- `example.com/shop` → shop-service

**Use Case:**
- Microservices architecture
- Different services at different paths

### 3. Name-Based Virtual Hosting

Routes to different services based on hostname.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: virtual-hosting
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
  
  - host: shop.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: shop-service
            port:
              number: 80
```

**Routing:**
- `example.com` → web-service
- `api.example.com` → api-service
- `shop.example.com` → shop-service

**Use Case:**
- Multiple subdomain-based applications
- Clear service separation by domain

### 4. Combination (Path + Host)

Mix both path and host-based routing.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: combo
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /api/v1
        pathType: Prefix
        backend:
          service:
            name: api-v1-service
            port:
              number: 8080
      - path: /api/v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-service
            port:
              number: 8080
```

---

## 🛣️ Path Routing

### Path Matching Types:[web:110][web:113]

#### Exact Path

Exact string match.

```yaml
paths:
- path: /api/users
  pathType: Exact
  backend:
    service:
      name: users-service
      port:
        number: 80
```

**Matches:**
- ✅ `/api/users`

**Does NOT match:**
- ❌ `/api/users/`
- ❌ `/api/users/123`
- ❌ `/api/user`

#### Prefix Path (Most Common)

Matches anything starting with the path.

```yaml
paths:
- path: /api
  pathType: Prefix
  backend:
    service:
      name: api-service
      port:
        number: 80
```

**Matches:**
- ✅ `/api`
- ✅ `/api/users`
- ✅ `/api/v1/products`

**Does NOT match:**
- ❌ `/ap`
- ❌ `/apis`

#### Implementation Type

(Implementation-specific, typically not used)

```yaml
pathType: ImplementationSpecific
```

### Path Rewriting

Sometimes your service doesn't expect the path you're routing. Use rewrite annotation.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-example
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /  # Rewrite /api to /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

**Without rewrite:**
- Request: `/api/users`
- Sent to backend: `/api/users` ❌ (backend may not expect this)

**With rewrite to `/`:**
- Request: `/api/users`
- Sent to backend: `/users` ✅ (backend receives clean path)

### Important: Longest Match Wins

If multiple paths match, the longest matching path is used.

```yaml
paths:
- path: /api
  backend:
    service:
      name: general-api
- path: /api/users
  backend:
    service:
      name: users-api
- path: /api/users/admin
  backend:
    service:
      name: admin-api
```

For request `/api/users/admin/roles`:
- `/api` matches ✅
- `/api/users` matches ✅
- `/api/users/admin` matches ✅ (LONGEST - wins!)

Routes to: `admin-api`

---

## 🏠 Host-Based Routing

### Basic Host Routing

Different hosts route to different services.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-routing
spec:
  ingressClassName: nginx
  rules:
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
  
  - host: dashboard.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: dashboard-service
            port:
              number: 3000
```

### Wildcard Hosts

Match multiple hosts with a wildcard.

```yaml
rules:
- host: "*.example.com"
  http:
    paths:
    - path: /
      pathType: Prefix
      backend:
        service:
          name: wildcard-service
          port:
            number: 80
```

**Matches:**
- ✅ `api.example.com`
- ✅ `web.example.com`
- ✅ `shop.example.com`
- ❌ `example.com` (no subdomain)

### DNS Setup

For host-based routing to work, DNS must resolve all hostnames to the same Ingress IP.

```bash
# Get Ingress IP
kubectl get ingress

# DNS records needed:
web.example.com    A 1.2.3.4
api.example.com    A 1.2.3.4
shop.example.com   A 1.2.3.4
*.example.com      A 1.2.3.4
```

---

## 🔒 TLS/SSL Configuration

### How Ingress TLS Works:[web:108][web:111][web:114]

1. Client makes HTTPS request to domain
2. Ingress Controller handles TLS handshake
3. Controller presents certificate (SSL/TLS termination)
4. After HTTPS negotiation, traffic is DECRYPTED
5. Traffic sent to backend service in plaintext (HTTP)
6. Backend pod responds with HTTP
7. Ingress Controller encrypts response back to client

### Steps to Configure TLS

#### Step 1: Create TLS Certificate & Key

For development (self-signed):

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=example.com"
```

For production (Let's Encrypt):

```bash
# Using cert-manager (recommended)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml
```

#### Step 2: Create Kubernetes TLS Secret

```bash
kubectl create secret tls my-tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  --namespace=default
```

Or as YAML:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-tls-secret
  namespace: default
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-certificate>
  tls.key: <base64-encoded-private-key>
```

#### Step 3: Configure Ingress with TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    - api.example.com
    secretName: my-tls-secret
  
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

### Important TLS Notes

⚠️ **TLS Block Requirements:**
- Host in TLS block MUST match host in rules
- Secret must be in same namespace as Ingress
- Certificate must include all domains (including both with/without www)

⚠️ **HTTP → HTTPS Redirect:**

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
```

⚠️ **Multiple Certificates:**

If you have multiple domains with different certificates:

```yaml
tls:
- hosts:
  - example.com
  secretName: example-tls
- hosts:
  - api.example.com
  secretName: api-tls
```

### Let's Encrypt with cert-manager

Automatically provision and renew certificates:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: letsencrypt-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    secretName: example-tls-cert  # cert-manager creates this
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

cert-manager automatically:
- ✅ Requests certificate from Let's Encrypt
- ✅ Creates Kubernetes Secret
- ✅ Configures Ingress
- ✅ Renews before expiration

---

## 🎮 Ingress Controllers

### Popular Controllers Comparison:[web:96][web:104][web:106][web:101]

| Controller | Performance | Simplicity | Features | Best For |
|------------|-------------|-----------|----------|----------|
| **NGINX** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Standard | Most use cases |
| **Traefik** | ⭐⭐⭐ | ⭐⭐⭐⭐ | Modern, dynamic | Easy setup |
| **HAProxy** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | High performance | High traffic |
| **Kong** | ⭐⭐⭐⭐ | ⭐⭐⭐ | API gateway | API management |
| **AWS ALB** | ⭐⭐⭐⭐ | ⭐⭐⭐ | AWS-native | AWS EKS |
| **Istio** | ⭐⭐⭐ | ⭐⭐ | Service mesh | Microservices |

### Installing NGINX Ingress Controller

```bash
# Using Helm (recommended)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# Verify installation
kubectl get pods -n ingress-nginx
kubectl get services -n ingress-nginx
```

### Installing Traefik Ingress Controller

```bash
# Using Helm
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace

# Verify installation
kubectl get pods -n traefik
```

---

## 💼 Real-World Use Cases

### Use Case 1: Microservices Architecture:[web:99]

**Scenario:** Swiggy-like food delivery app with multiple services

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: foodapp-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.foodapp.com
    secretName: foodapp-tls
  rules:
  - host: app.foodapp.com
    http:
      paths:
      - path: /restaurants
        pathType: Prefix
        backend:
          service:
            name: restaurant-service
            port:
              number: 80
      - path: /instamart
        pathType: Prefix
        backend:
          service:
            name: grocery-service
            port:
              number: 80
      - path: /payments
        pathType: Prefix
        backend:
          service:
            name: payment-service
            port:
              number: 80
      - path: /monitoring
        pathType: Prefix
        backend:
          service:
            name: dashboard-service
            port:
              number: 3000
```

**Benefits:**
- Single entry point for all services
- Unified TLS certificate
- Centralized routing logic
- Easy to add/remove services

### Use Case 2: Multiple Environments

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-env-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-prod
            port:
              number: 80
  
  - host: staging.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-staging
            port:
              number: 80
  
  - host: dev.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-dev
            port:
              number: 80
```

### Use Case 3: API Versioning

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-versioning
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1
            port:
              number: 8080
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2
            port:
              number: 8080
      - path: /v3
        pathType: Prefix
        backend:
          service:
            name: api-v3
            port:
              number: 8080
```

### Use Case 4: Load Balancing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: load-balanced
  annotations:
    nginx.ingress.kubernetes.io/load-balance: "round_robin"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service  # Service load balances across 3 pod replicas
            port:
              number: 80
```

---

## 🛠️ Practical Examples

### Example 1: Simple Web Application

```yaml
---
# Service
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
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
        ports:
        - containerPort: 8080
---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### Example 2: Multi-Service with TLS

```yaml
---
# Create TLS Secret
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-cert>
  tls.key: <base64-key>
---
# API Service
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
  - port: 8080
    targetPort: 8080
---
# Web Service
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
---
# Ingress with TLS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-service-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    - api.example.com
    secretName: tls-secret
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

---

## 🐛 Troubleshooting

### Issue 1: Ingress Controller Not Running:[web:112]

**Symptom:** Ingress created but no external IP, no traffic flowing

**Check:**

```bash
# Check if controller is running
kubectl get pods -n ingress-nginx
kubectl get pods -n traefik

# View logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

**Solution:** Install controller if missing

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx
```

### Issue 2: Ingress Not Routing Traffic:[web:112]

**Symptom:** Ingress shows ADDRESS but requests get 404/503 errors

**Check:**

```bash
# Verify Ingress configuration
kubectl describe ingress my-ingress

# Check if service exists and has endpoints
kubectl get svc my-service
kubectl get endpoints my-service

# If endpoints empty, service selector doesn't match pod labels
kubectl get pods --show-labels
```

**Common Causes:**
- ❌ Service doesn't exist
- ❌ Service selector doesn't match pod labels
- ❌ Wrong port in Ingress configuration
- ❌ Service has no endpoints

**Solution:**

```bash
# Fix service selector
kubectl patch svc my-service -p '{"spec":{"selector":{"app":"correct-app"}}}'

# Or recreate service with correct selector
```

### Issue 3: Path-Based Routing Not Working:[web:110][web:112]

**Symptom:** Only `/` path works, other paths return 404

**Check:**

```bash
# Backend application must serve content at those paths
# Example: if routing /api to service, service must respond to /api

# Test directly to pod
kubectl port-forward pod/my-pod 8080:8080
curl http://localhost:8080/api
```

**Solution:** Either fix backend OR use path rewriting

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
```

### Issue 4: TLS Not Working:[web:112]

**Symptom:** Browser shows "Your connection is not private"

**Check:**

```bash
# Certificate validity
kubectl get secret my-tls-secret -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text

# Verify host matches certificate
# Check ingress TLS configuration
kubectl describe ingress my-ingress
```

**Common Issues:**
- ❌ Certificate expired
- ❌ Hostname in TLS block doesn't match rules
- ❌ Secret in wrong namespace
- ❌ Certificate doesn't match domain

**Solution:**

```bash
# Renew certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt

# Update secret
kubectl create secret tls my-tls-secret --key tls.key --cert tls.crt --dry-run=client -o yaml | kubectl apply -f -
```

### Issue 5: DNS Not Resolving:[web:109][web:112]

**Symptom:** Can't reach application at domain name

**Check:**

```bash
# Get Ingress external IP
kubectl get ingress my-ingress

# Test DNS resolution
nslookup example.com
dig example.com

# Test with curl using IP
curl -H "Host: example.com" http://<ingress-ip>
```

**Solution:** Update DNS records

```bash
# DNS A record should point to Ingress IP
example.com    A  1.2.3.4
api.example.com A  1.2.3.4
```

### Issue 6: IngressClass Mismatch:[web:112]

**Symptom:** Ingress created but controller ignores it

**Check:**

```bash
# Available IngressClasses
kubectl get ingressclass

# Ingress ingressClassName
kubectl get ingress -o yaml | grep ingressClassName
```

**Solution:** Match ingressClassName in Ingress with deployed controller

```yaml
spec:
  ingressClassName: nginx  # Must match controller
```

### Debugging Checklist

```bash
# 1. Controller running?
kubectl get pods -n ingress-nginx

# 2. Service exists?
kubectl get svc my-service

# 3. Service has endpoints?
kubectl get endpoints my-service

# 4. Ingress correctly configured?
kubectl describe ingress my-ingress

# 5. Pod responding to requests?
kubectl port-forward pod/my-pod 8080:8080
curl http://localhost:8080/

# 6. Ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# 7. DNS resolving?
nslookup example.com

# 8. Can reach via IP?
curl -H "Host: example.com" http://<ingress-ip>
```

---

## ✅ Best Practices

### 1. Version Control Everything:[web:100][web:103]

```bash
# Keep all Ingress manifests in Git
# Use GitOps for automatic deployment
# Every change tracked and auditable
```

### 2. Secure TLS Configuration:[web:100][web:103]

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.2 TLSv1.3"
spec:
  tls:
  - hosts:
    - example.com
    secretName: tls-secret  # Use strong ciphers
```

### 3. Enable TLS by Default

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

### 4. Use Health Checks

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
```

### 5. Set Rate Limiting:[web:103][web:100]

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-connections: "5"
```

### 6. Implement Authentication

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
```

### 7. Use Consistent Naming Conventions

```yaml
metadata:
  name: app-name-ingress
  labels:
    app: app-name
    tier: edge
    environment: production
```

### 8. Monitor and Alert:[web:100][web:103]

```bash
# Monitor Ingress metrics
# Alert on high latency, error rates, certificate expiration
```

### 9. Test Before Production

```bash
# Test in staging with same config as prod
# Test DNS, TLS, routing, failover
```

### 10. Multiple Ingress Controllers:[web:104][web:103]

```bash
# Can run multiple controllers for different routes
# Example: NGINX for web, Kong for APIs
```

### 11. Use Annotations Carefully

```yaml
metadata:
  annotations:
    # Well-known annotations are safe
    cert-manager.io/cluster-issuer: letsencrypt-prod
    # Custom annotations should be controller-specific
    nginx.ingress.kubernetes.io/rewrite-target: /
```

### 12. Certificate Rotation Automation

```bash
# Use cert-manager for automatic renewal
# Test certificate renewal before production
```

### 13. Network Policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress
spec:
  podSelector:
    matchLabels:
      app: my-app
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
```

### 14. Horizontal Scaling

```bash
# Run multiple Ingress controller replicas
# Auto-scale with HPA based on metrics
```

### 15. Use Managed Ingress (Cloud Providers)

```bash
# AWS: ALB Ingress Controller
# GCP: Ingress with load balancer
# Azure: Azure Application Gateway
```

---

## ⚠️ Important Operational Notes

### Limitations

- ⚠️ **No TCP/UDP** - Ingress only supports HTTP/HTTPS
- ⚠️ **Layer 7 Only** - Cannot do Layer 4 load balancing
- ⚠️ **No gRPC** - Use Service mesh (Istio) for gRPC
- ⚠️ **Single Port** - TLS only on port 443
- ⚠️ **Ingress requires Controller** - Resource alone does nothing

### Security Considerations

- 🔒 Always use TLS for production
- 🔒 Rotate certificates regularly
- 🔒 Use RBAC to control Ingress creation
- 🔒 Implement rate limiting to prevent abuse
- 🔒 Use NetworkPolicies to restrict controller access

### Performance Tuning

- ⚡ Scale Ingress controller based on traffic
- ⚡ Tune buffer sizes for your workload
- ⚡ Use connection pooling
- ⚡ Monitor latency and adjust timeouts

### Cost Optimization

- 💰 Single Ingress replaces multiple LoadBalancers
- 💰 Share one load balancer across many services
- 💰 Use smaller controller instances if low traffic
- 💰 Auto-scale controller based on metrics

### High Availability

- 🔄 Run multiple Ingress controller replicas
- 🔄 Spread across multiple nodes
- 🔄 Use pod disruption budgets
- 🔄 Implement health checks and auto-restart

---

## 📚 Quick Reference

### Common Commands

```bash
# List all ingresses
kubectl get ingress
kubectl get ing

# Describe ingress
kubectl describe ingress my-ingress

# Get ingress YAML
kubectl get ingress my-ingress -o yaml

# Create ingress
kubectl apply -f ingress.yaml

# Delete ingress
kubectl delete ingress my-ingress

# Watch ingress
kubectl get ingress -w

# Check ingress controller status
kubectl get pods -n ingress-nginx

# View ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx -f

# Port forward to ingress controller
kubectl port-forward -n ingress-nginx svc/nginx-ingress 8080:80

# Get ingress IP
kubectl get ingress -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}'
```

### Ingress YAML Template

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: default
  labels:
    app: my-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    secretName: tls-secret
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

### Troubleshooting Checklist

- [ ] Ingress controller installed?
- [ ] Ingress resource created?
- [ ] Service exists and has endpoints?
- [ ] Service selector matches pod labels?
- [ ] IngressClass matches controller?
- [ ] Backend service responds to requests?
- [ ] DNS resolving correctly?
- [ ] Can reach via Ingress IP?
- [ ] TLS certificate valid?
- [ ] Certificate hostname matches ingress rules?

---

## 🎯 Summary

**Kubernetes Ingress** is a powerful API object that manages external HTTP/HTTPS access to your cluster. It provides:

✅ **Single entry point** for multiple services
✅ **Advanced routing** based on hostname and path
✅ **Cost efficiency** - one load balancer serves many services
✅ **TLS/SSL termination** at the edge
✅ **Easy management** - rules defined in YAML
✅ **Production-ready** - scalable, secure, reliable

By mastering Ingress configuration and troubleshooting, you can build robust, scalable APIs and web services that efficiently route millions of requests to appropriate backends.

---

**Created for DevOps and Cloud Engineers | Kubernetes Networking Guide**
