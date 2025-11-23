

## 🎯 Understanding HELM-yaml-files.yaml: What You Can Change

This guide explains each file in HELM-yaml-files.yaml and **exactly what you need to change** to run your own application.

---

# 📋 File 1: Chart.yaml (Metadata)

## What It Is
Information **about** your Helm chart (not the actual configuration).

## Original File
```yaml
apiVersion: v2
name: my-first-app
description: A simple Helm chart for Kubernetes
type: application
version: 1.0.0
appVersion: "1.0"
author:
  name: Your Name
  email: your@email.com
home: https://github.com/yourname/my-first-app
sources:
  - https://github.com/yourname/my-first-app
keywords:
  - kubernetes
  - helm
  - application
maintainers:
  - name: Your Name
    email: your@email.com
```

## What To Change

### 1. Name (IMPORTANT!)
```yaml
# CHANGE THIS
name: my-first-app

# TO YOUR APP NAME
name: my-webapp
# or
name: my-database
# or
name: payment-service
```

**Why:** This is your app's name in the Helm ecosystem

### 2. Description
```yaml
# CHANGE THIS
description: A simple Helm chart for Kubernetes

# TO DESCRIBE YOUR APP
description: Helm chart for payment service
# or
description: Database deployment chart
```

### 3. appVersion (Your App Version)
```yaml
# CHANGE THIS
appVersion: "1.0"

# TO YOUR APP VERSION
appVersion: "2.5"     # Your actual app version
appVersion: "1.2.3"   # Follow semantic versioning
```

### 4. version (Chart Version)
```yaml
# CHANGE THIS
version: 1.0.0

# WHEN YOU UPDATE THE CHART
version: 1.0.1   # Bug fix
version: 1.1.0   # New feature
version: 2.0.0   # Breaking change
```

### 5. Author Info
```yaml
# CHANGE THIS
author:
  name: Your Name
  email: your@email.com

# TO YOUR INFO
author:
  name: John Developer
  email: john@mycompany.com
```

### 6. Project URLs
```yaml
# CHANGE THESE
home: https://github.com/yourname/my-first-app
sources:
  - https://github.com/yourname/my-first-app

# TO YOUR PROJECT URLS
home: https://github.com/mycompany/my-webapp
sources:
  - https://github.com/mycompany/my-webapp
  - https://gitlab.com/mycompany/my-webapp
```

### 7. Keywords
```yaml
# CHANGE THESE
keywords:
  - kubernetes
  - helm
  - application

# TO DESCRIBE YOUR APP
keywords:
  - payment
  - api
  - microservice
# or
keywords:
  - database
  - postgresql
  - deployment
```

## Example: For a Payment Service
```yaml
apiVersion: v2
name: payment-service
description: Helm chart for payment microservice
type: application
version: 1.0.0
appVersion: "2.1"
author:
  name: Finance Team
  email: finance@mycompany.com
home: https://github.com/mycompany/payment-service
sources:
  - https://github.com/mycompany/payment-service
keywords:
  - payment
  - microservice
  - api
maintainers:
  - name: Finance Team
    email: finance@mycompany.com
```

---

# 📋 File 2: values.yaml (Configuration - MOST IMPORTANT!)

## What It Is
**The main configuration file** - defines defaults for your app.

## Original File
```yaml
replicaCount: 2

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "latest"

service:
  type: ClusterIP
  port: 80
  targetPort: 80

ingress:
  enabled: false
  className: "nginx"
  annotations: {}
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

env:
  LOG_LEVEL: info
  APP_ENV: development

labels:
  app: my-app
  version: v1.0
```

## What To Change (DETAILED)

### 1. replicaCount (Number of copies)
```yaml
# ORIGINAL (2 copies running)
replicaCount: 2

# CHANGE TO:
replicaCount: 1        # Only 1 copy (for dev/testing)
replicaCount: 3        # 3 copies (for production)
replicaCount: 5        # 5 copies (for high traffic)
```

**Why:** Controls how many copies of your app run at the same time.

**Real Example:**
```yaml
# Development (1 copy = cheap)
replicaCount: 1

# Staging (3 copies = balanced)
replicaCount: 3

# Production (5 copies = high availability)
replicaCount: 5
```

---

### 2. Image (Docker Image)
```yaml
# ORIGINAL (uses nginx)
image:
  repository: nginx              # Docker image name
  pullPolicy: IfNotPresent       # When to pull
  tag: "latest"                  # Image version

# CHANGE TO YOUR APP:
image:
  repository: mycompany/payment-service
  pullPolicy: IfNotPresent       # Use local if exists
  tag: "1.2.3"                   # Your app version

# OR FOR DIFFERENT REGISTRIES:
image:
  repository: docker.io/mycompany/app
  pullPolicy: Always             # Always pull latest
  tag: "v2.0"

# OR FROM PRIVATE REGISTRY:
image:
  repository: private-registry.com/myapp
  pullPolicy: Always
  tag: "latest"
```

**Key Points:**
- `repository`: Where your Docker image is stored
- `pullPolicy`: When to pull (IfNotPresent = use local, Always = always pull)
- `tag`: Which version of your image to use

**Real Examples:**
```yaml
# Example 1: Public Docker Hub
image:
  repository: nginx
  tag: "1.19"

# Example 2: Company Registry
image:
  repository: registry.mycompany.com/apps/payment-api
  tag: "v2.1"

# Example 3: Develop/Test
image:
  repository: mycompany/myapp
  pullPolicy: Always
  tag: "latest"

# Example 4: Production (specific version)
image:
  repository: mycompany/myapp
  pullPolicy: IfNotPresent
  tag: "v1.2.3"
```

---

### 3. Service (Expose Your App)
```yaml
# ORIGINAL
service:
  type: ClusterIP         # Type of service
  port: 80                # External port
  targetPort: 80          # Container port

# CHANGE TO:
service:
  type: ClusterIP         # Only within cluster
  port: 8080              # External: 8080
  targetPort: 8080        # Container listening: 8080

# OR FOR EXTERNAL ACCESS:
service:
  type: LoadBalancer      # External IP
  port: 443               # External: 443
  targetPort: 8443        # Container: 8443

# OR FOR NODE ACCESS:
service:
  type: NodePort          # Access via node IP
  port: 80                # Service port
  targetPort: 80
```

**What to Change:**
- `port`: The port users connect to
- `targetPort`: The port your app listens on inside container
- `type`: How to expose the service

**Real Examples:**
```yaml
# Web App (nginx/apache)
service:
  type: ClusterIP
  port: 80
  targetPort: 80

# API Service
service:
  type: LoadBalancer
  port: 443
  targetPort: 8443

# Database
service:
  type: ClusterIP
  port: 5432
  targetPort: 5432

# Node.js App
service:
  type: ClusterIP
  port: 3000
  targetPort: 3000
```

---

### 4. Ingress (Route External Traffic)
```yaml
# ORIGINAL (disabled)
ingress:
  enabled: false          # Not exposed via ingress

# CHANGE TO (enable and configure):
ingress:
  enabled: true           # Enable ingress
  className: "nginx"      # Ingress controller
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix

# REAL EXAMPLE FOR YOUR APP:
ingress:
  enabled: true
  className: "nginx"
  hosts:
    - host: payment-service.mycompany.com
      paths:
        - path: /
          pathType: Prefix
    - host: api.mycompany.com
      paths:
        - path: /payments
          pathType: Prefix
```

**What to Change:**
- `enabled`: true = expose via ingress
- `host`: Your domain name
- `paths`: URL paths to route

**Real Examples:**
```yaml
# Single domain
ingress:
  enabled: true
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix

# Multiple domains
ingress:
  enabled: true
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
    - host: web.example.com
      paths:
        - path: /
          pathType: Prefix

# Multiple paths on same domain
ingress:
  enabled: true
  hosts:
    - host: mycompany.com
      paths:
        - path: /api
          pathType: Prefix
        - path: /web
          pathType: Prefix
```

---

### 5. Resources (CPU & Memory)
```yaml
# ORIGINAL
resources:
  limits:
    cpu: 500m             # Max CPU
    memory: 512Mi         # Max memory
  requests:
    cpu: 250m             # Minimum CPU
    memory: 256Mi         # Minimum memory

# CHANGE BASED ON YOUR APP:
# Light app (static files, simple API)
resources:
  limits:
    cpu: 250m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

# Medium app (Node.js, Python)
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

# Heavy app (ML, database, Java)
resources:
  limits:
    cpu: 2000m            # 2 CPUs
    memory: 2Gi           # 2 GB
  requests:
    cpu: 1000m
    memory: 1Gi
```

**What to Change:**
- `limits.cpu`: Maximum CPU your app can use
- `limits.memory`: Maximum memory your app can use
- `requests.cpu`: Minimum CPU to allocate
- `requests.memory`: Minimum memory to allocate

**Real Examples:**
```yaml
# Static website
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi

# Node.js API
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

# Python ML Service
resources:
  limits:
    cpu: 2000m
    memory: 4Gi
  requests:
    cpu: 1000m
    memory: 2Gi

# Java Application
resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi
```

---

### 6. Autoscaling (Auto-Scale Your App)
```yaml
# ORIGINAL (disabled)
autoscaling:
  enabled: false

# ENABLE FOR PRODUCTION:
autoscaling:
  enabled: true           # Turn on autoscaling
  minReplicas: 3          # Always run at least 3
  maxReplicas: 10         # Never more than 10
  targetCPUUtilizationPercentage: 70    # Scale at 70% CPU

# CHANGE BASED ON YOUR NEEDS:
# Development (no scaling)
autoscaling:
  enabled: false

# Production (aggressive scaling)
autoscaling:
  enabled: true
  minReplicas: 5
  maxReplicas: 50
  targetCPUUtilizationPercentage: 60
```

**What to Change:**
- `enabled`: true/false
- `minReplicas`: Minimum copies always running
- `maxReplicas`: Maximum copies to spawn
- `targetCPUUtilizationPercentage`: At what % CPU to scale

**Real Examples:**
```yaml
# Dev environment (cheap)
autoscaling:
  enabled: false

# Staging environment (moderate)
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 75

# Production environment (aggressive)
autoscaling:
  enabled: true
  minReplicas: 5
  maxReplicas: 50
  targetCPUUtilizationPercentage: 60
```

---

### 7. Environment Variables (App Settings)
```yaml
# ORIGINAL
env:
  LOG_LEVEL: info
  APP_ENV: development

# CHANGE TO YOUR APP'S NEEDS:
env:
  LOG_LEVEL: debug
  APP_ENV: production
  DATABASE_HOST: db.example.com
  DATABASE_PORT: "5432"
  API_KEY: "your-secret-key"
  CACHE_ENABLED: "true"
```

**What to Change:**
Any variables your app needs to run

**Real Examples:**

For a Payment Service:
```yaml
env:
  LOG_LEVEL: info
  APP_ENV: production
  DATABASE_HOST: postgres.default.svc.cluster.local
  DATABASE_PORT: "5432"
  DATABASE_NAME: payments
  STRIPE_API_KEY: "sk_live_xxx"
  WEBHOOK_URL: "https://mycompany.com/webhooks"
```

For a Node.js App:
```yaml
env:
  NODE_ENV: production
  LOG_LEVEL: error
  CACHE_ENABLED: "true"
  REDIS_HOST: redis.default.svc.cluster.local
  REDIS_PORT: "6379"
  API_PORT: "3000"
```

For a Python App:
```yaml
env:
  FLASK_ENV: production
  DEBUG: "false"
  LOG_LEVEL: info
  DATABASE_URL: "postgresql://user:pass@db:5432/myapp"
  SECRET_KEY: "your-secret-key"
```

---

### 8. Labels (Organize Your App)
```yaml
# ORIGINAL
labels:
  app: my-app
  version: v1.0

# CHANGE TO:
labels:
  app: payment-service
  version: v2.1
  team: finance
  environment: production
  tier: backend
```

**What to Change:**
Labels help identify and organize your app

---

## Complete values.yaml Example for Payment Service

```yaml
# Payment Service Helm Chart Values

replicaCount: 3

image:
  repository: mycompany/payment-service
  pullPolicy: IfNotPresent
  tag: "2.1"

service:
  type: LoadBalancer
  port: 443
  targetPort: 8443

ingress:
  enabled: true
  className: "nginx"
  hosts:
    - host: payments.mycompany.com
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

env:
  LOG_LEVEL: info
  APP_ENV: production
  DATABASE_HOST: postgres.default.svc.cluster.local
  DATABASE_PORT: "5432"
  DATABASE_NAME: payments
  STRIPE_API_KEY: "sk_live_xxx"
  WEBHOOK_URL: "https://mycompany.com/webhooks"

labels:
  app: payment-service
  version: v2.1
  team: finance
  environment: production
```

---

# 📋 File 3: templates/deployment.yaml

## What It Is
The **template** that creates Kubernetes Deployment

## What Uses Your values.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
spec:
  replicas: {{ .Values.replicaCount }}    # ← From values.yaml
  template:
    spec:
      containers:
      - name: app
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"  # ← From values.yaml
        ports:
        - containerPort: {{ .Values.service.port }}  # ← From values.yaml
        env:
        {{- range $key, $value := .Values.env }}     # ← From values.yaml
        - name: {{ $key }}
          value: "{{ $value }}"
        {{- end }}
        resources:
          limits:
            cpu: {{ .Values.resources.limits.cpu }}   # ← From values.yaml
            memory: {{ .Values.resources.limits.memory }}
```

## What Gets Generated

After Helm processes this, it creates:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-payment-deployment
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: "mycompany/payment-service:2.1"
        ports:
        - containerPort: 443
        env:
        - name: LOG_LEVEL
          value: "info"
        - name: DATABASE_HOST
          value: "postgres.default.svc.cluster.local"
        resources:
          limits:
            cpu: 1000m
            memory: 1Gi
```

**You don't change this file!** It automatically uses your values.yaml

---

# 📋 File 4: templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-service
spec:
  type: {{ .Values.service.type }}        # ← From values.yaml (LoadBalancer, ClusterIP, etc.)
  ports:
    - port: {{ .Values.service.port }}    # ← External port
      targetPort: {{ .Values.service.targetPort }}  # ← Container port
```

**You don't change this file!** It uses values from values.yaml

---

# 📋 File 5: values-prod.yaml (Production Overrides)

```yaml
# Production values overrides
replicaCount: 5                    # More copies

image:
  tag: "1.0"
  pullPolicy: Always               # Always pull latest

service:
  type: LoadBalancer               # External IP

ingress:
  enabled: true                    # Enable ingress

resources:
  limits:
    cpu: 1000m                     # More resources
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true                    # Scale automatically
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

env:
  LOG_LEVEL: error                 # Less logging
  APP_ENV: production              # Production mode
```

## How to Use:

```bash
# Use production values
helm install my-app ./chart -f values-prod.yaml

# Or upgrade to production
helm upgrade my-app ./chart -f values-prod.yaml
```

---

# 📋 File 6: values-dev.yaml (Development Overrides)

```yaml
# Development values overrides
replicaCount: 1                    # Only 1 copy

image:
  tag: "latest"
  pullPolicy: Always               # Always pull

service:
  type: ClusterIP                  # No external access

ingress:
  enabled: false                   # No ingress

resources:
  limits:
    cpu: 250m                      # Less resources
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false                   # No scaling

env:
  LOG_LEVEL: debug                 # Debug logging
  APP_ENV: development             # Dev mode
```

## How to Use:

```bash
# Use development values
helm install my-app ./chart -f values-dev.yaml

# Or upgrade to development
helm upgrade my-app ./chart -f values-dev.yaml
```

---

# 🎯 Complete Real-World Example

## Scenario: Deploy Your Payment Service

### Step 1: Update Chart.yaml

```yaml
apiVersion: v2
name: payment-service
description: Helm chart for payment processing
type: application
version: 1.0.0
appVersion: "2.1"
```

### Step 2: Update values.yaml

```yaml
replicaCount: 1                    # Start with 1 for testing

image:
  repository: mycompany/payment-api
  pullPolicy: IfNotPresent
  tag: "2.1"

service:
  type: ClusterIP
  port: 8443
  targetPort: 8443

ingress:
  enabled: true
  hosts:
    - host: payment-api.mycompany.com
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: false

env:
  LOG_LEVEL: info
  APP_ENV: development
  DATABASE_HOST: postgres.default.svc.cluster.local
  DATABASE_PORT: "5432"
  DATABASE_NAME: payments
  API_URL: "https://payment-api.mycompany.com"
```

### Step 3: Test with Dev Values

```bash
helm install payment-test ./chart -f values-dev.yaml

# Check it worked
helm status payment-test
kubectl get pods
```

### Step 4: Deploy to Production with Prod Values

```yaml
# values-prod.yaml
replicaCount: 5
image:
  tag: "2.1"
  pullPolicy: Always
service:
  type: LoadBalancer
autoscaling:
  enabled: true
  minReplicas: 5
  maxReplicas: 50
env:
  LOG_LEVEL: error
  APP_ENV: production
```

```bash
helm install payment-prod ./chart -f values-prod.yaml

# Check it worked
helm status payment-prod
kubectl get pods
kubectl get svc
```

---

# 📊 Quick Reference: What to Change for Your App

| Section | Original | Change To |
|---------|----------|-----------|
| **Chart.yaml** name | my-first-app | your-app-name |
| **Chart.yaml** description | Generic | Your app description |
| **values.yaml** replicaCount | 2 | 1 (dev), 3 (prod), 5 (high-traffic) |
| **values.yaml** image.repository | nginx | your-registry/your-app |
| **values.yaml** image.tag | latest | your-version (e.g., v1.2.3) |
| **values.yaml** service.port | 80 | your-port (3000, 8080, 5432, etc.) |
| **values.yaml** service.targetPort | 80 | your-container-port |
| **values.yaml** resources.limits.cpu | 500m | your-needs (100m-2000m) |
| **values.yaml** resources.limits.memory | 512Mi | your-needs (128Mi-4Gi) |
| **values.yaml** ingress.hosts | myapp.example.com | your-domain |
| **values.yaml** env | LOG_LEVEL, APP_ENV | your-app-variables |
| **values-prod.yaml** All | Dev values | Production values |
| **values-dev.yaml** All | Prod values | Dev values |

---

# 🚀 Step-by-Step to Deploy Your Own App

### Step 1: Modify Chart.yaml
Change name, description, appVersion

### Step 2: Modify values.yaml
- Change image.repository to your Docker image
- Change image.tag to your app version
- Change service.port to your app's port
- Change resources to match your app's needs
- Add environment variables your app needs

### Step 3: Modify values-prod.yaml
Set production overrides

### Step 4: Modify values-dev.yaml
Set development overrides

### Step 5: Validate
```bash
helm lint ./chart
```

### Step 6: Test
```bash
helm install test ./chart --dry-run --debug
helm install test ./chart -f values-dev.yaml
```

### Step 7: Verify
```bash
helm status test
kubectl get pods
kubectl logs deployment/test-app
```

### Step 8: Deploy to Production
```bash
helm install prod ./chart -f values-prod.yaml
```

---


