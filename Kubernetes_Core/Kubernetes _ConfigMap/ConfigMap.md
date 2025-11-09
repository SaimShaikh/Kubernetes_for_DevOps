# ConfigMap in Kubernetes

## Table of Contents
- [What is a ConfigMap?](#what-is-a-configmap)
- [Why Use ConfigMap?](#why-use-configmap)
- [How ConfigMap Works](#how-configmap-works)
- [Core Concepts](#core-concepts)
- [ConfigMap vs Secret](#configmap-vs-secret)
- [Creating ConfigMaps](#creating-configmaps)
- [Using ConfigMaps in Pods](#using-configmaps-in-pods)
- [Real-World Problem Scenarios](#real-world-problem-scenarios)
- [ConfigMap Manifest Structure](#configmap-manifest-structure)
- [Practical Examples](#practical-examples)
- [Important Operations](#important-operations)
- [Limitations & Considerations](#limitations--considerations)
- [Best Practices](#best-practices)

---

## What is a ConfigMap?

A **ConfigMap** is a Kubernetes object used to store **non-sensitive configuration data** as key-value pairs. It decouples configuration settings from application code, allowing you to manage environment-specific settings separately from container images.

ConfigMaps store:
- Simple key-value pairs (application settings, URLs, feature flags)
- Configuration files (NGINX configs, app properties files)
- Binary data (images, certificates - base64 encoded)
- Environment variables for containers

**Key characteristics:**
- Data stored in plain text (not encrypted like Secrets)
- Maximum size limit of **1 MB** per ConfigMap
- Namespace-scoped (available to pods in the same namespace)
- Can be created from files, directories, or literal values
- Immutable option available for enhanced security

---

## Why Use ConfigMap?

### Problem Statement: Why Not Just Hardcode Configuration?

Imagine you're deploying a **web application** across multiple environments (dev, staging, production):

```dockerfile
# ❌ Wrong: Hardcoded configuration in container
FROM node:18
WORKDIR /app
COPY . .
RUN npm install

# Configuration hardcoded - must rebuild container for each environment
ENV DB_HOST=production-mysql.company.local
ENV DB_PORT=3306
ENV LOG_LEVEL=info
ENV API_TIMEOUT=30000

CMD ["npm", "start"]
```

**Problems that occur:**
1. **Image rebuild required** → Every configuration change needs a new container image
2. **Security risk** → Sensitive-looking configs are baked into the image
3. **Not portable** → Same image can't run in different environments without modification
4. **Version explosion** → Multiple image versions for same code with different configs
5. **Difficult updates** → Updating a feature flag requires rebuilding and redeploying

### Solution: Use ConfigMap

```yaml
# ✅ Correct: Configuration externalized
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DB_HOST: "production-mysql.company.local"
  DB_PORT: "3306"
  LOG_LEVEL: "info"
  API_TIMEOUT: "30000"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0  # Same image for all environments
        envFrom:
        - configMapRef:
            name: app-config  # Reference ConfigMap
```

**Benefits:**
1. **Single image build** → `myapp:1.0` used everywhere
2. **Environment-specific** → Use different ConfigMaps for dev/staging/prod
3. **Dynamic updates** → Change config without rebuilding image
4. **Portable** → "Build once, run anywhere" promise
5. **Easy management** → Centralized configuration storage

---

## How ConfigMap Works

### Data Flow

```
1. Create ConfigMap
   ↓
   kubectl apply -f configmap.yaml
   ↓
   ConfigMap stored in Kubernetes cluster (etcd)

2. Pod references ConfigMap
   ↓
   Pod specification includes: envFrom, env.valueFrom, or volumeMount
   ↓
   Data injected into container

3. Two injection methods:

   Method A: Environment Variables
   ├─ Data available as $ENV_VAR in container
   └─ Cannot be updated without pod restart

   Method B: Volume Mount
   ├─ Data available as files in mounted directory
   └─ Updates visible almost immediately (kubelet syncs)
```

### ConfigMap Storage & Access

```
Cluster (etcd)
├── ConfigMap: "app-config"
│   ├── DB_HOST: "mysql.default.svc.cluster.local"
│   ├── DB_PORT: "3306"
│   └── LOG_LEVEL: "debug"
│
└── Pod: "web-app-pod-123"
    ├── Reads app-config ConfigMap
    ├── Injects as environment variables
    └── Application can access: $DB_HOST, $DB_PORT, $LOG_LEVEL
```

---

## Core Concepts

### 1. Data vs BinaryData

ConfigMaps support two data fields:

#### **data** field (Plain text, UTF-8)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-settings
data:
  app.properties: |
    server.port=8080
    server.context-path=/api
  database.url: "jdbc:mysql://localhost:3306/mydb"
  feature.flags: "DARK_MODE=true,BETA_UI=false"
```

- Best for: configuration files, URLs, simple strings
- Readable in kubectl output
- No encoding needed
- Max size per entry: reasonably large (part of 1MB total)

#### **binaryData** field (Base64-encoded)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-binary
binaryData:
  certificate.crt: LS0tLS1CRUdJTi... # Base64 encoded
  logo.png: iVBORw0KGgoAAAANSU... # Base64 encoded
```

- Best for: images, certificates, compiled configs
- Data must be base64-encoded
- Kubectl displays as opaque
- Good for non-text data

**Important:** Keys can appear in either `data` OR `binaryData`, not both.

```yaml
# ✅ Valid
data:
  config.yaml: "..."
binaryData:
  image.png: "..."

# ❌ Invalid (same key in both)
data:
  config: "value1"
binaryData:
  config: "value2"  # Error: duplicate key
```

### 2. ConfigMap Creation Methods

| Method | Command | Use Case |
|--------|---------|----------|
| **Declarative (YAML)** | `kubectl apply -f configmap.yaml` | Production, version control |
| **From Literal** | `kubectl create configmap --from-literal=key=value` | Quick testing |
| **From File** | `kubectl create configmap --from-file=config.yaml` | Single file configs |
| **From Directory** | `kubectl create configmap --from-file=./config/` | Multiple config files |
| **From Env File** | `kubectl create configmap --from-env-file=.env` | .env file parsing |

### 3. Data Injection Methods

#### **Method 1: envFrom (All ConfigMap data as env vars)**

```yaml
containers:
- name: app
  envFrom:
  - configMapRef:
      name: app-config  # All keys become env vars
```

Result: Each key becomes an environment variable automatically
```
DB_HOST=mysql.default.svc...
DB_PORT=3306
LOG_LEVEL=debug
```

#### **Method 2: env with valueFrom (Selective env vars)**

```yaml
containers:
- name: app
  env:
  - name: DATABASE_URL       # Custom env var name
    valueFrom:
      configMapKeyRef:
        name: app-config     # ConfigMap name
        key: db_connection   # Specific key
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: logging_level
```

Result: Only selected keys become env vars with custom names

#### **Method 3: Volume Mount (Files in container)**

```yaml
containers:
- name: app
  volumeMounts:
  - name: config-volume
    mountPath: /etc/config   # Directory path in container
    readOnly: true
volumes:
- name: config-volume
  configMap:
    name: app-config         # ConfigMap name
    items:
    - key: app.properties    # ConfigMap key
      path: application.properties  # File name in container
```

Result: ConfigMap keys become files accessible via `/etc/config/application.properties`

### 4. Update Behavior

| Injection Method | Update Propagation | Action Required |
|------------------|-------------------|-----------------|
| **envFrom (env vars)** | Not automatic | Pod must be recreated (deleted/restarted) |
| **valueFrom (env vars)** | Not automatic | Pod must be recreated (deleted/restarted) |
| **Volume Mount** | ~30-60 seconds | None - automatic sync by kubelet |

**Key Difference:**
- Environment variables are set **once at pod startup**
- Volume-mounted files are **watched and updated** by kubelet

### 5. Immutable ConfigMaps

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: immutable-config
immutable: true  # ← Locks data and binaryData
data:
  api.key: "secret123"
  db.url: "mysql://localhost"
```

**When marked immutable:**
- Data cannot be changed
- Cannot revert to mutable state
- Kubelet stops watching for changes (performance improvement)
- Provides data consistency guarantee

**Updating immutable ConfigMaps:**
```bash
# ❌ This will fail
kubectl patch cm immutable-config -p '{"data":{"api.key":"newvalue"}}'

# ✅ Solution: Create new ConfigMap and update Pod reference
kubectl apply -f new-configmap.yaml
# Update Deployment to reference new ConfigMap name
```

---

## ConfigMap vs Secret

| Aspect | ConfigMap | Secret |
|--------|-----------|--------|
| **Purpose** | Non-sensitive configuration | Sensitive data (credentials, tokens) |
| **Data Format** | Plain text in `data` field | Base64-encoded in `data` field |
| **Visibility** | Readable via `kubectl get cm` | Requires explicit flag: `kubectl get secret -o yaml` |
| **Security** | No encryption (accessible to anyone with read permissions) | Should be encrypted at rest (requires setup) |
| **Size Limit** | 1 MB | 1 MB |
| **Use Cases** | Database URLs, log levels, feature flags | Passwords, API keys, TLS certificates |
| **etcd Storage** | Unencrypted by default | Should be encrypted via encryption provider |

### When to Use ConfigMap:
- Database connection strings (host/port only, no credentials)
- API endpoints and URLs
- Feature flags and feature toggles
- Application settings (timeouts, retries, batch sizes)
- Logging levels and verbosity

### When to Use Secret:
- Database passwords and usernames
- API keys and tokens
- TLS/SSL certificates and keys
- OAuth credentials
- Any value you wouldn't want exposed in `kubectl describe`

---

## Creating ConfigMaps

### Method 1: Declarative (YAML) - Recommended for Production

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
  labels:
    app: myapp
    environment: production
data:
  # Simple key-value pairs
  app.name: "MyApplication"
  app.version: "1.0.0"
  
  # Multiline values
  nginx.conf: |
    server {
      listen 80;
      server_name _;
      location / {
        proxy_pass http://backend:8080;
      }
    }
  
  # Properties file format
  application.properties: |
    spring.datasource.url=jdbc:mysql://mysql:3306/mydb
    spring.datasource.username=root
    spring.jpa.hibernate.ddl-auto=update
    logging.level.root=INFO
```

```bash
# Apply to cluster
kubectl apply -f configmap.yaml

# Verify
kubectl get configmap
kubectl describe cm app-config
```

### Method 2: From Literal Values (Quick Testing)

```bash
# Single value
kubectl create configmap test-config --from-literal=key1=value1

# Multiple values
kubectl create configmap test-config \
  --from-literal=DB_HOST=localhost \
  --from-literal=DB_PORT=5432 \
  --from-literal=LOG_LEVEL=debug

# With dry-run for verification
kubectl create configmap test-config \
  --from-literal=key=value \
  --dry-run=client -o yaml
```

### Method 3: From File

```bash
# Create from single file
# File content becomes the value, filename becomes the key
kubectl create configmap app-config --from-file=app.properties

# Create from multiple files
kubectl create configmap app-config \
  --from-file=./config/app.properties \
  --from-file=./config/database.properties

# Create with custom key name
kubectl create configmap app-config --from-file=mykey=./config/app.properties
```

### Method 4: From Directory

```bash
# All files in directory become ConfigMap entries
# File name = key, File content = value
kubectl create configmap app-config --from-file=./config/

# Directory structure:
# ./config/
# ├── app.properties
# ├── database.properties
# └── logging.yaml

# Results in ConfigMap:
# app.properties: <file content>
# database.properties: <file content>
# logging.yaml: <file content>
```

### Method 5: From .env File

```bash
# Create ConfigMap from .env format file
# File: .env
# DB_HOST=mysql.example.com
# DB_PORT=3306
# LOG_LEVEL=debug

kubectl create configmap app-config --from-env-file=.env
```

---

## Using ConfigMaps in Pods

### Scenario 1: All ConfigMap Data as Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    envFrom:
    - configMapRef:
        name: app-config  # All keys become env vars
```

**Result inside container:**
```bash
$ env | grep -E '^(DB_|LOG_|APP_)'
DB_HOST=mysql.default.svc.cluster.local
DB_PORT=3306
LOG_LEVEL=info
APP_NAME=MyApplication
```

### Scenario 2: Specific ConfigMap Keys as Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    - name: DATABASE_URL        # Custom env var name
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: db_connection
    - name: LOG_LEVEL           # Another env var
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: logging_level
    - name: FIXED_VALUE
      value: "hardcoded-value"  # Can mix with regular values
```

### Scenario 3: ConfigMap Mounted as Files

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:
      - key: app.properties
        path: application.properties  # ConfigMap key → file name
      - key: database.properties
        path: database.properties
```

**Result inside container:**
```bash
$ ls -la /etc/config/
application.properties
database.properties

$ cat /etc/config/application.properties
spring.datasource.url=jdbc:mysql://mysql:3306/mydb
```

### Scenario 4: Mount Specific Key to Custom Path

```yaml
volumeMounts:
- name: config-volume
  mountPath: /app/config
volumes:
- name: config-volume
  configMap:
    name: app-config
    items:
    - key: nginx.conf
      path: ../../../etc/nginx/nginx.conf  # Relative path
      mode: 0644                           # File permissions (octal)
```

---

## Real-World Problem Scenarios

### Scenario 1: Multi-Environment Deployment (Dev, Staging, Prod)

**Problem:** Same application image needs different configurations for each environment.

**Solution:**

```yaml
# dev-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev
data:
  DB_HOST: "dev-mysql.database.svc.cluster.local"
  DB_PORT: "3306"
  LOG_LEVEL: "debug"
  API_TIMEOUT: "60000"
  CACHE_TTL: "300"
---
# staging-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: staging
data:
  DB_HOST: "staging-mysql.database.svc.cluster.local"
  DB_PORT: "3306"
  LOG_LEVEL: "info"
  API_TIMEOUT: "30000"
  CACHE_TTL: "3600"
---
# prod-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  DB_HOST: "prod-mysql.database.svc.cluster.local"
  DB_PORT: "3306"
  LOG_LEVEL: "warn"
  API_TIMEOUT: "10000"
  CACHE_TTL: "86400"
```

**Deployment (Same for all environments):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0
        envFrom:
        - configMapRef:
            name: app-config  # References local namespace's ConfigMap
```

**Deploy to each environment:**

```bash
# Deploy to dev
kubectl apply -f dev-config.yaml -n dev
kubectl apply -f deployment.yaml -n dev

# Deploy to staging
kubectl apply -f staging-config.yaml -n staging
kubectl apply -f deployment.yaml -n staging

# Deploy to prod
kubectl apply -f prod-config.yaml -n production
kubectl apply -f deployment.yaml -n production

# Same image (myapp:1.0) runs with different configs!
```

---

### Scenario 2: Application with Configuration Files

**Problem:** Application needs configuration files (nginx.conf, app.properties) that vary by environment.

**Solution:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    worker_processes auto;
    worker_connections 2048;
    
    upstream backend {
      server app-0.app-svc:8080;
      server app-1.app-svc:8080;
      server app-2.app-svc:8080;
    }
    
    server {
      listen 80;
      client_max_body_size 50M;
      
      location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
      }
      
      location /health {
        access_log off;
        return 200 "healthy\n";
      }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        volumeMounts:
        - name: config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf    # Mount specific key as single file
          readOnly: true
        - name: config
          mountPath: /etc/nginx/conf.d
          readOnly: true
        ports:
        - containerPort: 80
      volumes:
      - name: config
        configMap:
          name: nginx-config
          defaultMode: 0644
```

**Inside container:**
```bash
$ ls -la /etc/nginx/
nginx.conf  # From ConfigMap

$ nginx -t
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

---

### Scenario 3: Feature Flags & Dynamic Configuration

**Problem:** Need to enable/disable features without rebuilding application.

**Solution:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: feature-flags
data:
  features.json: |
    {
      "dark_mode": true,
      "beta_ui": false,
      "new_search": true,
      "admin_panel": false,
      "api_v2": true,
      "payment_retry": true,
      "email_notifications": true
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0
        volumeMounts:
        - name: features
          mountPath: /app/config/features.json
          subPath: features.json
          readOnly: true
      volumes:
      - name: features
        configMap:
          name: feature-flags
```

**Application code:**
```javascript
// Node.js example - reads features from mounted file
const fs = require('fs');
const features = JSON.parse(fs.readFileSync('/app/config/features.json'));

if (features.dark_mode) {
  // Enable dark mode
}

if (features.beta_ui) {
  // Show beta UI
}

// Update: Just change ConfigMap
// kubectl apply -f new-feature-flags.yaml
// kubelet watches and syncs changes (~30-60 seconds)
// No app restart needed!
```

**Update features without redeploying:**

```bash
# Edit ConfigMap
kubectl edit configmap feature-flags

# Or apply new version
kubectl apply -f updated-feature-flags.yaml

# Changes reflected in pods within 30-60 seconds
# (kubelet refresh interval)

# Verify
kubectl get cm feature-flags -o yaml
```

---

### Scenario 4: Application with Secrets (Database Credentials)

**Problem:** Need to store database URL AND keep credentials secure.

**Solution: Mix ConfigMap + Secret**

```yaml
# ConfigMap for non-sensitive data
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DB_HOST: "mysql.database.svc.cluster.local"
  DB_PORT: "3306"
  DB_NAME: "production_db"
  API_ENDPOINT: "https://api.example.com"
---
# Secret for sensitive data
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  DB_USER: "app_user"
  DB_PASSWORD: "securepassword123"
  API_TOKEN: "sk-1234567890"
---
# Deployment uses both
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0
        envFrom:
        - configMapRef:
            name: app-config    # Non-sensitive config
        - secretRef:
            name: app-secrets   # Sensitive data
        # Now container has env vars from both:
        # DB_HOST, DB_PORT, DB_NAME (from ConfigMap)
        # DB_USER, DB_PASSWORD, API_TOKEN (from Secret)
```

---

## ConfigMap Manifest Structure

### Complete Manifest Breakdown

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config              # ConfigMap name (used in references)
  namespace: default            # Namespace scope
  labels:                        # Optional labels for filtering
    app: myapp
    version: v1
    environment: production
  annotations:                  # Optional metadata
    description: "Application configuration"
    owner: "platform-team"
data:
  # String key-value pairs (plain text)
  # Keys must be valid DNS subdomain or valid RFC 1123 (alphanumeric, -, _, .)
  app.name: "MyApplication"
  app.version: "1.0.0"
  log_level: "info"
  
  # Multiline values
  config.yaml: |
    server:
      port: 8080
      timeout: 30
    database:
      host: mysql
      pool_size: 20
  
  # JSON format
  features.json: |
    {
      "darkMode": true,
      "betaUI": false
    }
  
  # File content
  app.properties: |
    spring.application.name=MyApp
    spring.datasource.url=jdbc:mysql://mysql:3306/mydb
    logging.level.root=INFO

binaryData:
  # Base64-encoded binary data
  # Use for certificates, images, compiled configs
  certificate.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0... # base64 encoded
  logo.png: iVBORw0KGgoAAAANSUhEUgAAAAUA... # base64 encoded

immutable: false  # ⚠️ Once set to true, cannot be changed back
                  # ✅ Improves performance (kubelet stops watching)
                  # ⚠️ Must create new ConfigMap to make changes
```

### ConfigMap Key Naming Rules

```yaml
# ✅ Valid key names
data:
  simple_key: value
  key-with-dashes: value
  key.with.dots: value
  APP_NAME: value
  _key_starting_underscore: value

# ❌ Invalid key names
data:
  key with spaces: value          # Spaces not allowed
  key@invalid: value              # Special chars not allowed
  123.numeric.start: value        # Cannot start with number
```

---

## Practical Examples

### Example 1: Simple Environment Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: simple-config
data:
  ENVIRONMENT: "production"
  LOG_LEVEL: "info"
  CACHE_ENABLED: "true"
---
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    envFrom:
    - configMapRef:
        name: simple-config
```

### Example 2: ConfigMap with Configuration Files

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-files
data:
  docker-entrypoint.sh: |
    #!/bin/bash
    echo "Starting application..."
    npm install
    npm start
  
  package.json: |
    {
      "name": "myapp",
      "version": "1.0.0",
      "scripts": {
        "start": "node server.js"
      }
    }
  
  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://backend:8080;
      }
    }
---
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: ubuntu:20.04
    volumeMounts:
    - name: config
      mountPath: /app/config
  volumes:
  - name: config
    configMap:
      name: app-files
      defaultMode: 0755
```

### Example 3: Selective ConfigMap Keys as Environment Variables

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: database-config
data:
  host: "mysql.default.svc.cluster.local"
  port: "3306"
  name: "mydb"
  user: "app_user"
---
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: database-config
          key: host
    - name: DB_PORT
      valueFrom:
        configMapKeyRef:
          name: database-config
          key: port
    - name: DB_NAME
      valueFrom:
        configMapKeyRef:
          name: database-config
          key: name
    # Note: 'user' key is not referenced - not injected as env var
```

### Example 4: Immutable ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: immutable-constants
immutable: true  # ← Locked from changes
data:
  API_VERSION: "v1"
  PROTOCOL: "https"
  TIMEOUT: "30"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: strict-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0
        envFrom:
        - configMapRef:
            name: immutable-constants
```

**To update immutable ConfigMap:**

```bash
# ❌ This will fail
kubectl patch cm immutable-constants -p '{"data":{"TIMEOUT":"60"}}'
# Error: field is immutable

# ✅ Create new ConfigMap and update Deployment
kubectl apply -f new-immutable-config.yaml
# Update Deployment to reference: immutable-constants-v2
```

---

## Important Operations

### Create ConfigMap

```bash
# From YAML file (recommended)
kubectl apply -f configmap.yaml

# From literal values
kubectl create configmap app-config \
  --from-literal=DB_HOST=localhost \
  --from-literal=DB_PORT=5432

# From file
kubectl create configmap app-config --from-file=./config/app.properties

# From directory
kubectl create configmap app-config --from-file=./config/

# From .env file
kubectl create configmap app-config --from-env-file=.env.production
```

### List ConfigMaps

```bash
# List all ConfigMaps in current namespace
kubectl get configmap

# List in specific namespace
kubectl get cm -n production

# List with additional info
kubectl get cm -o wide

# List all ConfigMaps across all namespaces
kubectl get cm --all-namespaces
```

### View ConfigMap Details

```bash
# Show ConfigMap content (YAML format)
kubectl get cm app-config -o yaml

# Show specific key value
kubectl get cm app-config -o jsonpath='{.data.DB_HOST}'

# Describe ConfigMap
kubectl describe cm app-config
```

### Edit ConfigMap

```bash
# Edit in default editor
kubectl edit cm app-config

# Edit in specific namespace
kubectl edit cm app-config -n production

# Edit using patch
kubectl patch cm app-config -p '{"data":{"LOG_LEVEL":"debug"}}'

# Replace entire ConfigMap
kubectl replace -f updated-configmap.yaml

# Apply with force replace
kubectl apply -f configmap.yaml --force
```

### Update ConfigMap Data

```bash
# Method 1: Edit via kubectl edit
kubectl edit cm app-config
# (Opens editor for manual editing)

# Method 2: Patch specific key
kubectl patch configmap app-config -p '{"data":{"LOG_LEVEL":"warn"}}'

# Method 3: Delete and recreate
kubectl delete cm app-config
kubectl apply -f configmap.yaml

# Verify pod sees update (if volume mounted)
kubectl exec -it app-pod -- cat /etc/config/app.properties
```

### Delete ConfigMap

```bash
# Delete ConfigMap
kubectl delete cm app-config

# Delete in specific namespace
kubectl delete cm app-config -n production

# Delete all ConfigMaps in namespace
kubectl delete cm --all

# Delete specific ConfigMaps
kubectl delete cm config1 config2 config3

# Force delete (immediate, no grace period)
kubectl delete cm app-config --grace-period=0 --force
```

### Copy ConfigMap to Another Namespace

```bash
# Export from one namespace
kubectl get cm app-config -n dev -o yaml > configmap.yaml

# Import to another namespace (manually edit namespace field)
kubectl apply -f configmap.yaml -n staging

# Or in one command
kubectl get cm app-config -n dev -o yaml | \
  sed 's/namespace: dev/namespace: staging/' | \
  kubectl apply -f -
```

### Export/Backup ConfigMaps

```bash
# Export single ConfigMap
kubectl get cm app-config -o yaml > backup-app-config.yaml

# Export all ConfigMaps in namespace
kubectl get cm -o yaml > all-configmaps-backup.yaml

# Export all ConfigMaps across all namespaces
kubectl get cm --all-namespaces -o yaml > all-configmaps-all-ns.yaml
```

---

## Limitations & Considerations

### 1. Size Limitation (1 MB Max)

```yaml
# ❌ This will fail if total > 1MB
apiVersion: v1
kind: ConfigMap
metadata:
  name: large-config
data:
  huge-file: |
    # Large content here
    # If this exceeds 1MB total, creation fails with:
    # "ConfigMap "huge" is invalid: data: Too long: must have at most 1048576 characters"
```

**Solutions for large configs:**
- Split into multiple ConfigMaps
- Use external configuration service
- Mount files from shared storage (NFS, PVC)
- Use database for configuration

**Recommended approach:**

```yaml
# Instead of one 2MB ConfigMap:
# Create multiple smaller ConfigMaps

# configmap-part1.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-part1
data:
  app.properties: |
    # First half of configuration

# configmap-part2.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-part2
data:
  database.properties: |
    # Second half of configuration

# In Pod, mount both
volumes:
- name: config1
  configMap:
    name: app-config-part1
- name: config2
  configMap:
    name: app-config-part2
```

### 2. Environment Variables Not Updated Automatically

```yaml
# ❌ Env vars are set once at pod startup
spec:
  containers:
  - name: app
    image: myapp:1.0
    envFrom:
    - configMapRef:
        name: app-config

# If you update ConfigMap:
# kubectl patch cm app-config -p '{"data":{"LOG_LEVEL":"debug"}}'
# The pod's env vars are NOT updated!
```

**Solutions:**
- Use volume mount instead (updates within 30-60 seconds)
- Restart pods to reload env vars
- Watch for ConfigMap changes in application code

```bash
# Force pod restart to reload env vars
kubectl delete pod app-pod
# Deployment will create new pod with updated ConfigMap env vars

# Or use Deployment rollout
kubectl rollout restart deployment web-app
```

### 3. Volume Mounts Have Sync Delay

```yaml
# ✅ Volume-mounted ConfigMaps update automatically
# But with ~30-60 second delay (kubelet sync interval)

volumeMounts:
- name: config
  mountPath: /etc/config
volumes:
- name: config
  configMap:
    name: app-config
```

**Issue:** Updates aren't instantaneous
```bash
# Update ConfigMap
kubectl patch cm app-config -p '{"data":{"LOG_LEVEL":"debug"}}'

# Inside pod, file updates after 30-60 seconds
# Until then, old value is visible:
$ cat /etc/config/LOG_LEVEL
# Shows old value for ~30-60 seconds
```

### 4. Binary Data Must Be Base64-Encoded

```bash
# To add binary file to ConfigMap:

# Create from binary file (automatically base64-encoded)
kubectl create configmap app-images --from-file=logo.png=./logo.png

# Or manually base64-encode:
base64 logo.png
# Copy output and paste into binaryData field

# ❌ This won't work (raw binary)
apiVersion: v1
kind: ConfigMap
metadata:
  name: images
binaryData:
  logo.png: [raw binary data]  # Won't work

# ✅ Must be base64-encoded
apiVersion: v1
kind: ConfigMap
metadata:
  name: images
binaryData:
  logo.png: iVBORw0KGgoAAAANSUhEUgAAAAUA...  # base64 encoded
```

### 5. Keys Must Follow Kubernetes Naming Rules

```yaml
# ✅ Valid keys
data:
  app_config: value          # underscore ok
  app-config: value          # dash ok
  app.config: value          # dot ok
  APP_CONFIG: value          # uppercase ok

# ❌ Invalid keys
data:
  app config: value          # space not allowed
  app@config: value          # @ not allowed
  123app: value              # cannot start with digit
  app config: value          # space not allowed
```

### 6. ConfigMap Data Visible to Anyone with Read Permissions

```bash
# Anyone can read ConfigMap values
kubectl get cm app-config -o yaml
# Shows all data in plaintext!

# Never put secrets in ConfigMap
# Use Kubernetes Secret instead
```

**Security consideration:**
```yaml
# ❌ Wrong: Credentials in ConfigMap
apiVersion: v1
kind: ConfigMap
data:
  db_password: "mypassword"  # VISIBLE TO EVERYONE!

# ✅ Correct: Use Secret for sensitive data
apiVersion: v1
kind: Secret
type: Opaque
stringData:
  db_password: "mypassword"  # Better (though still needs encryption)
```

### 7. Immutable ConfigMaps Cannot Be Updated

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: constants
immutable: true
data:
  API_VERSION: "v1"

# ❌ Any modification will fail
kubectl patch cm constants -p '{"data":{"API_VERSION":"v2"}}'
# Error: field is immutable

# ✅ Solution: Create new ConfigMap with different name
kubectl create configmap constants-v2 ...
# Update Deployment to reference constants-v2
```

---

## Best Practices

### 1. Use Descriptive Naming

```yaml
# ❌ Unclear
apiVersion: v1
kind: ConfigMap
metadata:
  name: config

# ✅ Clear, descriptive name
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-app-config
  namespace: production
```

### 2. Organize Configuration by Concern

```yaml
# ❌ Mix everything
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DB_HOST: mysql
  NGINX_WORKERS: "4"
  LOG_LEVEL: info
  REDIS_PORT: 6379
  APP_TIMEOUT: 30000

# ✅ Create separate ConfigMaps by component
apiVersion: v1
kind: ConfigMap
metadata:
  name: database-config
data:
  DB_HOST: mysql
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: logging-config
data:
  LOG_LEVEL: info
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: cache-config
data:
  REDIS_PORT: 6379
```

### 3. Use Labels and Annotations

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  labels:
    app: myapp
    environment: production
    version: v1
  annotations:
    description: "Production application configuration"
    owner: "platform-team@company.com"
    last-updated: "2024-01-15"
```

### 4. Separate ConfigMap from Secret

```yaml
# ❌ Mix sensitive and non-sensitive data
apiVersion: v1
kind: ConfigMap
data:
  DB_HOST: mysql
  DB_PASSWORD: "secret123"  # EXPOSED!

# ✅ Separate clearly
# configmap.yaml
apiVersion: v1
kind: ConfigMap
data:
  DB_HOST: mysql
  LOG_LEVEL: info

# secret.yaml
apiVersion: v1
kind: Secret
stringData:
  DB_PASSWORD: "secret123"
```

### 5. Use Volume Mounts for Configuration Files

```yaml
# ❌ Injecting large files as env vars
envFrom:
- configMapRef:
    name: large-config  # Contains 500KB file

# ✅ Mount as volume for files
volumeMounts:
- name: config
  mountPath: /etc/config
volumes:
- name: config
  configMap:
    name: app-config
```

### 6. Document Configuration Parameters

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  annotations:
    documentation: |
      Configuration for web application:
      - LOG_LEVEL: (debug|info|warn|error) - Controls logging verbosity
      - API_TIMEOUT: Integer milliseconds, recommended 5000-30000
      - CACHE_TTL: Integer seconds, recommended 300-3600
      - FEATURE_FLAGS: JSON object with feature toggles
data:
  LOG_LEVEL: "info"
  API_TIMEOUT: "30000"
  CACHE_TTL: "3600"
  FEATURE_FLAGS: |
    {
      "darkMode": true,
      "betaUI": false
    }
```

### 7. Use Immutable ConfigMaps for Constants

```yaml
# ✅ Lock constants from accidental changes
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-constants
immutable: true  # Prevents modifications
data:
  API_VERSION: "v1"
  PROTOCOL: "https"
  TIMEOUT: "30"
```

### 8. Environment-Specific ConfigMaps

```bash
# Create separate ConfigMaps per environment

# Directory structure:
# config/
# ├── configmap-dev.yaml
# ├── configmap-staging.yaml
# └── configmap-prod.yaml

# Deploy to specific environment
kubectl apply -f config/configmap-prod.yaml -n production
```

### 9. Version Your ConfigMaps

```yaml
# Add version suffix for rollback capability
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2  # Version in name
  labels:
    app: myapp
    version: v2
data:
  LOG_LEVEL: "info"

# To rollback, create ConfigMap with v1 name
# Update Deployment to reference v1
```

### 10. Use ConfigMap for Feature Toggles

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: feature-flags
data:
  features.json: |
    {
      "newDashboard": true,
      "betaPayment": false,
      "darkMode": true,
      "adminPanel": false,
      "twoFactorAuth": true
    }

# Update without redeploying:
# kubectl patch cm feature-flags -p '{"data":{"features.json":"{\"newDashboard\":false, ...}"}}'
# Apps reading from volume see changes within 30-60 seconds
```

---

## Common Troubleshooting

### Problem: ConfigMap Not Found When Pod Starts

```bash
# Error: Pod stuck in Pending, logs show ConfigMap missing
kubectl describe pod app-pod
# Warning: invalid value for volumeClaimTemplate specified

# Cause: ConfigMap doesn't exist or named incorrectly

# Solution:
# 1. Verify ConfigMap exists in correct namespace
kubectl get cm -n production

# 2. Check ConfigMap name in Pod spec
kubectl get pod app-pod -o yaml | grep configMap

# 3. Create ConfigMap before Pod
kubectl apply -f configmap.yaml -n production
kubectl apply -f pod.yaml -n production
```

### Problem: Environment Variables Not Updated

```bash
# Changed ConfigMap but pod env vars didn't update
kubectl patch cm app-config -p '{"data":{"LOG_LEVEL":"debug"}}'

# Inside pod, $LOG_LEVEL still shows old value

# Cause: Env vars are set once at startup, not watched

# Solution: Restart pod to reload
kubectl delete pod app-pod
# Deployment creates new pod with updated ConfigMap

# Or use volume mount instead (auto-updates)
```

### Problem: ConfigMap Size Exceeded 1MB

```bash
# Error creating ConfigMap: "data: Too long: must have at most 1048576 characters"

# Solution: Split into multiple ConfigMaps
kubectl create configmap app-config-part1 --from-file=file1.txt
kubectl create configmap app-config-part2 --from-file=file2.txt

# Mount both in Pod
volumeMounts:
- name: config1
  mountPath: /etc/config1
- name: config2
  mountPath: /etc/config2
```

### Problem: Changes to Immutable ConfigMap

```bash
# Error: Cannot update immutable ConfigMap
kubectl patch cm constants -p '{"data":{"VERSION":"v2"}}'
# Error: field is immutable

# Solution: Create new ConfigMap
kubectl create configmap constants-v2 ...
# Update Deployment to reference constants-v2
# Delete old immutable ConfigMap when pods are updated
```

---

## Summary

ConfigMaps are essential for **decoupling application configuration** from container images. They enable:
- **Environment-specific deployments** without rebuilding images
- **Dynamic configuration updates** (when using volume mounts)
- **Centralized configuration management** in the cluster
- **"Build once, run anywhere"** principle

**Key takeaways:**
- Use ConfigMap for **non-sensitive** configuration
- Use Secret for **sensitive** data (passwords, tokens, keys)
- Use volume mounts for files that need **dynamic updates**
- Use environment variables for **simple** key-value pairs
- Keep ConfigMaps **under 1MB** in size
- Document your configuration parameters
- Separate configuration by **environment** (dev, staging, prod)
- Use **immutable** ConfigMaps for constants

ConfigMaps, combined with Deployments and Services, form the foundation of portable, manageable Kubernetes applications.
