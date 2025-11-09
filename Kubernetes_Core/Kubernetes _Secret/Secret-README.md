# Secret in Kubernetes

## Table of Contents
- [What is a Secret?](#what-is-a-secret)
- [Why Use Secret?](#why-use-secret)
- [How Secret Works](#how-secret-works)
- [Secret Types](#secret-types)
- [Secret vs ConfigMap](#secret-vs-configmap)
- [Creating Secrets](#creating-secrets)
- [Using Secrets in Pods](#using-secrets-in-pods)
- [Real-World Problem Scenarios](#real-world-problem-scenarios)
- [Secret Manifest Structure](#secret-manifest-structure)
- [Practical Examples](#practical-examples)
- [Important Operations](#important-operations)
- [Limitations & Considerations](#limitations--considerations)
- [Best Practices](#best-practices)

---

## What is a Secret?

A **Secret** is a Kubernetes object used to store **sensitive data** such as passwords, API keys, tokens, SSH keys, and TLS certificates. It decouples sensitive information from application code, preventing secrets from being exposed in container images, logs, or configuration files.

Secrets store:
- Database passwords and usernames
- API keys and tokens
- TLS/SSL certificates and private keys
- OAuth credentials
- Docker registry credentials
- SSH keys

**Key characteristics:**
- Data stored in **base64-encoded format** (not encrypted by default, but can be configured)
- Maximum size limit of **1 MB** per Secret
- Namespace-scoped (available to pods in the same namespace)
- Can be created from files, directories, or literal values
- Supports multiple types for different use cases
- Should be restricted with RBAC (Role-Based Access Control)

---

## Why Use Secret?

### Problem Statement: Why Not Just Put Credentials in ConfigMap?

Imagine you're deploying a **microservice that connects to a database**:

```dockerfile
# ❌ Wrong: Credentials in ConfigMap (visible to everyone)
FROM python:3.9
WORKDIR /app
COPY . .

# Don't do this - credentials visible in logs and configuration
RUN pip install -r requirements.txt
ENV DB_PASSWORD="mysql-password-123"
ENV API_KEY="sk-1234567890abcdef"
```

**Problems that occur:**
1. **Credentials visible in plain text** → `kubectl get cm` shows passwords
2. **No encryption** → Anyone with cluster access can read credentials
3. **Accidentally exposed in logs** → `kubectl logs` might leak secrets
4. **Version control risk** → Credentials might end up in Git history
5. **Lack of access control** → All pods can access all data
6. **Audit trail missing** → No tracking of who accessed secrets

### Solution: Use Secret

```yaml
# ✅ Correct: Sensitive data in Secret
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  DB_PASSWORD: "mysql-password-123"
  API_KEY: "sk-1234567890abcdef"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DB_PASSWORD
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: API_KEY
```

**Benefits:**
1. **Sensitive data separated** → Not visible in regular ConfigMaps
2. **RBAC control** → Restrict who can read secrets
3. **Audit logging** → Track secret access (with proper setup)
4. **Rotation possible** → Update secrets without rebuilding images
5. **Encryption ready** → Can be encrypted at rest with KMS
6. **Purpose-specific** → Different types for different credential types

---

## How Secret Works

### Data Flow

```
1. Create Secret
   ↓
   kubectl apply -f secret.yaml
   ↓
   Secret data is base64-encoded and stored in etcd

2. Pod references Secret
   ↓
   Pod specification includes: env.valueFrom.secretKeyRef or volumeMount
   ↓
   Data injected into container (decoded from base64)

3. Two injection methods:

   Method A: Environment Variables
   ├─ Data available as $ENV_VAR in container (decoded)
   └─ Cannot be updated without pod restart

   Method B: Volume Mount
   ├─ Data available as files in mounted directory (decoded)
   └─ Updates visible ~30-60 seconds (kubelet syncs)
```

### Base64 Encoding (Not Encryption)

```
Secret Storage in etcd:
┌─────────────────────────────────┐
│ DB_PASSWORD: bXlzcWwtcGFzcw==   │  Base64-encoded
│ API_KEY: c2stMTIzNDU2Nzg5MA==   │  (NOT encrypted)
└─────────────────────────────────┘
         ↓ (Decoded in pod)
         ↓
Container Environment:
DB_PASSWORD=mysql-pass
API_KEY=sk-1234567890
```

**Important:** Base64 encoding ≠ encryption. It's just encoding!
- Anyone with cluster access can decode it
- Enable encryption at rest with KMS for true protection

---

## Secret Types

### 1. Opaque (Default)

The default and most common type for arbitrary user-defined sensitive data.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  password: bXlzZWNyZXRwYXNz  # base64-encoded
  api_key: c2stMTIzNDU2Nzg5MA==
```

**Use for:**
- Database passwords
- API keys
- Generic credentials
- Any sensitive data

### 2. docker-registry

For storing Docker registry credentials (`.docker/config.json`).

```bash
# Create Docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.azurecr.io \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myemail@example.com
```

**YAML representation:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: regcred
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: eyJhdXRocyI6eyJteXJlZ2lzdHJ5Lm... # base64-encoded config
```

**Use in Pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-image-pod
spec:
  imagePullSecrets:
  - name: regcred  # References docker-registry secret
  containers:
  - name: app
    image: myregistry.azurecr.io/myapp:1.0
```

### 3. kubernetes.io/service-account-token

Auto-generated token for ServiceAccount authentication.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: default-token-abc123
type: kubernetes.io/service-account-token
data:
  ca.crt: LS0tLS1CRUdJTi...  # CA certificate
  namespace: ZGVmYXVsdA==      # namespace (base64)
  token: eyJhbGciOiJIUzI1NiI... # ServiceAccount token
```

**Auto-created by Kubernetes for each ServiceAccount**

### 4. kubernetes.io/basic-auth

For storing basic authentication credentials (username + password).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth
type: kubernetes.io/basic-auth
stringData:
  username: "admin"
  password: "secure123"
```

**Use for:**
- HTTP Basic Auth
- Database credentials

### 5. kubernetes.io/ssh-auth

For storing SSH authentication credentials.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ssh-auth-secret
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: LS0tLS1CRUdJTi... # base64-encoded private key
```

**Create from SSH key:**

```bash
kubectl create secret generic ssh-auth-secret \
  --from-file=ssh-privatekey=~/.ssh/id_rsa \
  --type=kubernetes.io/ssh-auth
```

### 6. kubernetes.io/tls

For storing TLS certificates and their associated private keys.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi... # base64-encoded certificate
  tls.key: LS0tLS1CRUdJTi... # base64-encoded private key
```

**Create from certificate files:**

```bash
kubectl create secret tls tls-secret \
  --cert=path/to/cert.pem \
  --key=path/to/key.pem
```

**Use in Ingress:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - example.com
    secretName: tls-secret  # References TLS secret
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: web-service
            port:
              number: 80
```

---

## Secret vs ConfigMap

| Aspect | Secret | ConfigMap |
|--------|--------|-----------|
| **Purpose** | Sensitive data (passwords, tokens, keys) | Non-sensitive configuration |
| **Data Format** | Base64-encoded (not encrypted by default) | Plain text (readable) |
| **Visibility** | Hidden from `kubectl get` (requires `-o yaml`) | Visible in `kubectl describe` |
| **Encryption** | Can be encrypted at rest with KMS | Not encrypted |
| **RBAC** | Should restrict access via RBAC | Usually openly accessible |
| **Size Limit** | 1 MB per Secret | 1 MB per ConfigMap |
| **Use Cases** | Passwords, API keys, TLS certs, tokens | URLs, log levels, feature flags |
| **Update Behavior** | Env vars: requires pod restart; Volume: ~30-60s sync | Env vars: requires pod restart; Volume: ~30-60s sync |
| **Immutability** | Supported via `immutable: true` | Supported via `immutable: true` |

### Decision Matrix

| Data Type | ConfigMap | Secret |
|-----------|-----------|--------|
| Database URL | ✅ Yes | ❌ No |
| Database Password | ❌ No | ✅ Yes |
| API Endpoint | ✅ Yes | ❌ No |
| API Key | ❌ No | ✅ Yes |
| Feature Flags | ✅ Yes | ❌ No |
| SSH Private Key | ❌ No | ✅ Yes |
| SSH Public Key | ✅ Maybe | ✅ Better |
| Log Level | ✅ Yes | ❌ No |
| TLS Certificate | ❌ No | ✅ Yes |
| Application Settings | ✅ Yes | ❌ No |
| OAuth Token | ❌ No | ✅ Yes |
| Slack Webhook | ❌ No | ✅ Yes |

---

## Creating Secrets

### Method 1: Declarative (YAML) - Recommended for Production

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
  labels:
    app: myapp
    environment: production
type: Opaque

# Method 1: stringData (auto base64-encoded)
stringData:
  username: "dbuser"
  password: "my-secure-password"
  connection_string: "postgresql://dbuser:my-secure-password@localhost:5432/mydb"

# Method 2: data (manually base64-encoded)
# data:
#   password: bXktc2VjdXJlLXBhc3N3b3Jk  # base64-encoded "my-secure-password"
```

```bash
# Apply to cluster
kubectl apply -f secret.yaml

# Verify
kubectl get secret db-credentials
kubectl get secret db-credentials -o yaml  # Shows base64-encoded data
```

### Method 2: From Literal Values (Quick Testing)

```bash
# Single value
kubectl create secret generic db-secret --from-literal=password=mypassword

# Multiple values
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=secure123 \
  --from-literal=api_key=sk-1234567890

# With type
kubectl create secret generic db-secret \
  --type=kubernetes.io/basic-auth \
  --from-literal=username=admin \
  --from-literal=password=secure123
```

### Method 3: From File

```bash
# Create from single file
# File name becomes key, file content becomes value
kubectl create secret generic ssh-secret --from-file=id_rsa=~/.ssh/id_rsa

# Create from multiple files
kubectl create secret generic certs \
  --from-file=tls.crt=./cert.pem \
  --from-file=tls.key=./key.pem
```

### Method 4: From Directory

```bash
# All files in directory become Secret entries
kubectl create secret generic app-secrets --from-file=./secrets/

# Directory structure:
# ./secrets/
# ├── db_password
# ├── api_key
# └── tls_cert

# Results in Secret with keys: db_password, api_key, tls_cert
```

### Method 5: Docker Registry Secret

```bash
# Create Docker registry credential secret
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.azurecr.io \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myemail@example.com
```

### Method 6: TLS Secret

```bash
# Create from certificate files
kubectl create secret tls tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key
```

### Method 7: Basic Auth Secret

```bash
kubectl create secret generic basic-auth \
  --type=kubernetes.io/basic-auth \
  --from-literal=username=admin \
  --from-literal=password=secure123
```

---

## Using Secrets in Pods

### Scenario 1: Secret as Environment Variables (valueFrom)

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
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials  # Secret name
          key: username          # Key in Secret
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    - name: DB_HOST
      value: "postgres.default.svc.cluster.local"  # Non-secret can use plain value
```

**Result inside container:**
```bash
$ env | grep DB_
DB_USER=dbuser
DB_PASSWORD=my-secure-password
DB_HOST=postgres.default.svc.cluster.local
```

### Scenario 2: All Secret Data as Environment Variables (envFrom)

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
    - secretRef:
        name: db-credentials  # All keys become env vars
```

**Result inside container:**
```bash
$ env | grep -v '^KUBE_' | grep -v '^KUBERNETES_'
username=dbuser
password=my-secure-password
connection_string=postgresql://dbuser:...
```

### Scenario 3: Secret Mounted as Files

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
    - name: secrets-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secrets-volume
    secret:
      secretName: db-credentials  # Secret name
      items:
      - key: username
        path: db_user.txt         # File name in container
      - key: password
        path: db_pass.txt
      defaultMode: 0400           # File permissions (read-only)
```

**Result inside container:**
```bash
$ ls -la /etc/secrets/
-r-------- 1 root root   7 db_user.txt
-r-------- 1 root root  18 db_pass.txt

$ cat /etc/secrets/db_user.txt
dbuser

$ cat /etc/secrets/db_pass.txt
my-secure-password
```

### Scenario 4: SSH Key as Secret

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: git-clone-pod
spec:
  containers:
  - name: app
    image: alpine:latest
    command: ["sh", "-c", "git clone git@github.com:user/repo.git && cat /root/.ssh/id_rsa"]
    volumeMounts:
    - name: ssh-key
      mountPath: /root/.ssh
      readOnly: true
  volumes:
  - name: ssh-key
    secret:
      secretName: ssh-key-secret
      items:
      - key: ssh-privatekey
        path: id_rsa
        mode: 0400  # Read-only for owner
```

### Scenario 5: TLS Certificate in Ingress

```yaml
# Create TLS secret
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi...  # base64-encoded cert
  tls.key: LS0tLS1CRUdJTi...  # base64-encoded key
---
# Use in Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
spec:
  tls:
  - hosts:
    - example.com
    secretName: tls-secret      # References TLS secret
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

### Scenario 6: Docker Registry Secret for Private Images

```yaml
# Create docker-registry secret
apiVersion: v1
kind: Secret
metadata:
  name: regcred
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: eyJhdXRocyI6eyJteXJlZ2lzdHJ5...
---
# Use in Pod
apiVersion: v1
kind: Pod
metadata:
  name: private-image-pod
spec:
  imagePullSecrets:
  - name: regcred  # Reference docker-registry secret
  containers:
  - name: app
    image: myregistry.azurecr.io/private/myapp:1.0
```

---

## Real-World Problem Scenarios

### Scenario 1: Database Credentials Management

**Problem:** Multiple microservices need database credentials without hardcoding them.

**Solution:**

```yaml
# Secret for production database
apiVersion: v1
kind: Secret
metadata:
  name: prod-db-credentials
  namespace: production
type: kubernetes.io/basic-auth
stringData:
  username: "prod_app_user"
  password: "prod_secure_password_12345"
  connection_url: "postgresql://prod_app_user:prod_secure_password_12345@prod-postgres.default.svc.cluster.local:5432/production_db"
---
# Deployment using secret
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  namespace: production
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: api
        image: myapi:1.0
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: prod-db-credentials
              key: connection_url
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: prod-db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: prod-db-credentials
              key: password
```

**Workflow:**
1. Create secret in production namespace only
2. Different secrets for dev/staging/prod
3. Rotate passwords by updating secret
4. No code changes needed

---

### Scenario 2: API Keys and Service Tokens

**Problem:** Application needs to call third-party APIs securely.

**Solution:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: external-api-keys
  namespace: production
type: Opaque
stringData:
  # Stripe payment API
  STRIPE_API_KEY: "sk_live_51234567890abcdefghij"
  STRIPE_SECRET_KEY: "rk_live_secret_key_here"
  
  # SendGrid email API
  SENDGRID_API_KEY: "SG.1234567890abcdefghijklmnopqrst"
  
  # AWS credentials for S3
  AWS_ACCESS_KEY_ID: "AKIAIOSFODNN7EXAMPLE"
  AWS_SECRET_ACCESS_KEY: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
  
  # Slack webhook
  SLACK_WEBHOOK_URL: "https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXX"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  template:
    spec:
      containers:
      - name: payment-processor
        image: payment-service:1.0
        envFrom:
        - secretRef:
            name: external-api-keys  # All keys available as env vars
        volumeMounts:
        - name: config
          mountPath: /app/config
      volumes:
      - name: config
        configMap:
          name: app-config  # Non-sensitive config
```

**Rotation procedure:**
```bash
# Update secret with new API key
kubectl patch secret external-api-keys -p '{"stringData":{"STRIPE_API_KEY":"sk_live_new_key"}}'

# Or recreate from new file
kubectl delete secret external-api-keys
kubectl create secret generic external-api-keys --from-file=./api_keys.env
```

---

### Scenario 3: TLS Certificates for HTTPS

**Problem:** Web application needs SSL/TLS certificate for HTTPS.

**Solution:**

```bash
# Create TLS secret from certificate files
kubectl create secret tls app-tls-secret \
  --cert=./certs/app.crt \
  --key=./certs/app.key
```

**Use in Ingress:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-app-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls-secret  # TLS secret
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 8080
```

**Auto-renewal with cert-manager:**

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: app-tls
spec:
  secretName: app-tls-secret  # Secret to store cert
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - app.example.com
  - www.app.example.com
```

---

### Scenario 4: Private Docker Registry Access

**Problem:** Pods need to pull images from private Docker registry.

**Solution:**

```bash
# Create docker-registry secret
kubectl create secret docker-registry my-registry-creds \
  --docker-server=myregistry.azurecr.io \
  --docker-username=username \
  --docker-password=password \
  --docker-email=email@example.com
```

**Use in Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      imagePullSecrets:
      - name: my-registry-creds  # Docker registry secret
      containers:
      - name: app
        image: myregistry.azurecr.io/myapp:1.0
        ports:
        - containerPort: 8080
```

**Alternative: Create in ServiceAccount:**

```bash
# Default service account can use the secret
kubectl patch serviceaccount default -p '{"imagePullSecrets":[{"name":"my-registry-creds"}]}'
```

---

### Scenario 5: SSH Keys for Git Operations

**Problem:** Pod needs to clone private GitHub repositories.

**Solution:**

```bash
# Create SSH key secret
kubectl create secret generic github-ssh-key \
  --from-file=ssh-privatekey=~/.ssh/id_rsa \
  --from-file=ssh-publickey=~/.ssh/id_rsa.pub \
  --type=kubernetes.io/ssh-auth
```

**Use in Pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: git-clone-pod
spec:
  containers:
  - name: cloner
    image: alpine/git:latest
    command: 
    - sh
    - -c
    - |
      mkdir -p ~/.ssh
      ssh-keyscan github.com >> ~/.ssh/known_hosts
      git clone git@github.com:myorg/private-repo.git /repo
      cat /repo/README.md
    volumeMounts:
    - name: ssh-key
      mountPath: /root/.ssh
      readOnly: true
  volumes:
  - name: ssh-key
    secret:
      secretName: github-ssh-key
      items:
      - key: ssh-privatekey
        path: id_rsa
        mode: 0400  # Owner read-only
```

---

## Secret Manifest Structure

### Complete Manifest Breakdown

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret                    # Secret name
  namespace: production              # Namespace scope
  labels:                            # Labels for filtering
    app: myapp
    environment: production
    sensitive: "true"
  annotations:                       # Metadata
    description: "Database credentials"
    owner: "platform-team"
    rotation_required: "90days"

type: Opaque                         # Secret type (Opaque, TLS, basic-auth, etc.)

# Method 1: stringData (recommended for YAML - auto base64-encoded)
stringData:
  username: "dbuser"                 # Automatically base64-encoded
  password: "my-secure-password"
  connection_string: "postgresql://..."

# Method 2: data (manually base64-encoded - use for sensitive workflows)
# data:
#   password: bXktc2VjdXJlLXBhc3N3b3Jk  # "my-secure-password" in base64

immutable: false                     # Set to true to prevent modifications
```

### Secret Type Specifications

| Type | data Keys | Purpose |
|------|-----------|---------|
| **Opaque** | Any custom keys | Generic sensitive data |
| **kubernetes.io/basic-auth** | username, password | HTTP Basic Auth |
| **kubernetes.io/ssh-auth** | ssh-privatekey (optional: ssh-publickey) | SSH authentication |
| **kubernetes.io/tls** | tls.crt, tls.key | TLS certificates |
| **kubernetes.io/dockercfg** | .dockercfg | Docker config (legacy) |
| **kubernetes.io/dockerconfigjson** | .dockerconfigjson | Docker config (current) |
| **kubernetes.io/service-account-token** | token, ca.crt, namespace | ServiceAccount token |
| **bootstrap.kubernetes.io/token** | token, description, expiration, ttl, usage-auth | Bootstrap token |

---

## Practical Examples

### Example 1: Simple Password Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: simple-password
type: Opaque
stringData:
  password: "supersecret123"
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
    - name: APP_PASSWORD
      valueFrom:
        secretKeyRef:
          name: simple-password
          key: password
```

### Example 2: Multi-key Secret with Different Data Types

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  api_key: "sk_live_1234567890"
  db_user: "appuser"
  db_pass: "secure_password"
  ssl_cert: |
    -----BEGIN CERTIFICATE-----
    MIIDXTCCAkWgAwIBAgIJAJC1/iNAZwqDMA0GCSqGSIb3...
    -----END CERTIFICATE-----
  config.json: |
    {
      "apiVersion": "v1",
      "endpoint": "https://api.example.com",
      "timeout": 30
    }
```

### Example 3: Basic Auth Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth-secret
type: kubernetes.io/basic-auth
stringData:
  username: "admin"
  password: "secure_admin_password"
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
    - name: AUTH_USER
      valueFrom:
        secretKeyRef:
          name: basic-auth-secret
          key: username
    - name: AUTH_PASS
      valueFrom:
        secretKeyRef:
          name: basic-auth-secret
          key: password
```

### Example 4: TLS Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUR...  # base64-encoded cert
  tls.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUVWQ...  # base64-encoded key
```

### Example 5: Docker Registry Secret

```bash
# Create command
kubectl create secret docker-registry docker-secret \
  --docker-server=myregistry.azurecr.io \
  --docker-username=myuser \
  --docker-password=mypass \
  --docker-email=myemail@example.com
```

### Example 6: Secret with Environment File

```bash
# File: .env.production
DB_HOST=postgres.production.svc.cluster.local
DB_USER=produser
DB_PASSWORD=prod_secure_password
API_KEY=sk_prod_1234567890
SECRET_TOKEN=token_prod_xyz

# Create secret from .env file
kubectl create secret generic app-env-secret --from-env-file=.env.production
```

---

## Important Operations

### Create Secret

```bash
# From YAML file (recommended)
kubectl apply -f secret.yaml

# From literal values
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=secure123

# From file
kubectl create secret generic certs \
  --from-file=tls.crt=./cert.pem \
  --from-file=tls.key=./key.pem

# From directory
kubectl create secret generic all-secrets --from-file=./secrets/

# Docker registry
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.azurecr.io \
  --docker-username=user \
  --docker-password=pass

# TLS
kubectl create secret tls tls-secret \
  --cert=./tls.crt \
  --key=./tls.key

# With dry-run for verification
kubectl create secret generic test-secret \
  --from-literal=key=value \
  --dry-run=client -o yaml
```

### List Secrets

```bash
# List all secrets in current namespace
kubectl get secret

# List in specific namespace
kubectl get secret -n production

# List all secrets across all namespaces
kubectl get secret --all-namespaces

# Show additional info
kubectl get secret -o wide
```

### View Secret Details

```bash
# Show secret (values are base64-encoded)
kubectl get secret my-secret -o yaml

# Decode specific value
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d

# Describe secret
kubectl describe secret my-secret
```

### Edit Secret

```bash
# Edit in default editor
kubectl edit secret my-secret

# Edit in specific namespace
kubectl edit secret my-secret -n production

# Patch specific key
kubectl patch secret my-secret -p '{"stringData":{"password":"newpass"}}'
```

### Delete Secret

```bash
# Delete secret
kubectl delete secret my-secret

# Delete in specific namespace
kubectl delete secret my-secret -n production

# Delete all secrets in namespace
kubectl delete secret --all -n production

# Force delete (immediate)
kubectl delete secret my-secret --grace-period=0 --force
```

### Base64 Encoding/Decoding

```bash
# Encode string to base64
echo -n "mysecretpassword" | base64
# Output: bXlzZWNyZXRwYXNzd29yZA==

# Decode from base64
echo "bXlzZWNyZXRwYXNzd29yZA==" | base64 -d
# Output: mysecretpassword

# Encode file
cat secret_file.txt | base64

# Decode and save
echo "bXlzZWNyZXRwYXNzd29yZA==" | base64 -d > decoded_file.txt
```

### Copy Secret Between Namespaces

```bash
# Export secret
kubectl get secret my-secret -n source -o yaml > secret.yaml

# Import to another namespace (edit namespace field in YAML)
kubectl apply -f secret.yaml -n destination

# Or in one command
kubectl get secret my-secret -n source -o yaml | \
  sed 's/namespace: source/namespace: destination/' | \
  kubectl apply -f -
```

---

## Limitations & Considerations

### 1. Base64 Encoding is NOT Encryption

```bash
# ⚠️ WARNING: Base64 is encoding, not encryption
echo -n "my-secret-password" | base64
# Output: bXktc2VjcmV0LXBhc3N3b3Jk

# Anyone can decode it:
echo "bXktc2VjcmV0LXBhc3N3b3Jk" | base64 -d
# Output: my-secret-password

# ✅ Solution: Enable encryption at rest with KMS
```

**Enable encryption at rest:**

```yaml
# Requires KMS provider configuration on API server
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}
```

### 2. RBAC Access Not Restricted by Default

```bash
# ⚠️ Anyone with cluster access can read secrets
kubectl get secret my-secret -o yaml
# Shows all data in base64

# ✅ Solution: Use RBAC to restrict access
```

**Restrict secret access with RBAC:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
  resourceNames: ["db-credentials"]  # Only this secret
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-db-secret
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: secret-reader
subjects:
- kind: ServiceAccount
  name: app-service-account
  namespace: default
```

### 3. Size Limitation (1 MB Max)

```yaml
# ❌ This will fail if total > 1MB
apiVersion: v1
kind: Secret
metadata:
  name: huge-secret
stringData:
  huge-data: |
    # Large content here
    # If exceeds 1MB: "The Secret "huge" is invalid: data: Too long: must have at most 1048576 characters"
```

**Solutions for large secrets:**
- Split into multiple secrets
- Use external secrets manager (HashiCorp Vault, AWS Secrets Manager)
- Store in database with reference in Secret

```yaml
# Split large secret
apiVersion: v1
kind: Secret
metadata:
  name: secret-part1
stringData:
  cert1: |
    # First half of certificate

---
apiVersion: v1
kind: Secret
metadata:
  name: secret-part2
stringData:
  cert2: |
    # Second half of certificate
```

### 4. No Auto-Rotation

```bash
# ❌ Secrets don't auto-rotate
# Must manually update secrets

# ✅ Solution: Implement rotation mechanism
```

**Manual rotation workflow:**

```bash
# Create new secret with updated credentials
kubectl create secret generic db-credentials-v2 \
  --from-literal=username=newuser \
  --from-literal=password=newpassword

# Update Deployment to reference new secret
kubectl patch deployment api-service -p '{"spec":{"template":{"spec":{"volumes":[{"name":"secrets","secret":{"secretName":"db-credentials-v2"}}]}}}}'

# Wait for pods to restart
kubectl rollout status deployment/api-service

# Delete old secret
kubectl delete secret db-credentials
```

### 5. Not Suitable for Large Binary Data

```yaml
# ❌ Large binary files should not be stored as secrets
apiVersion: v1
kind: Secret
metadata:
  name: large-binary
binaryData:
  large-file: <base64-encoded-100MB-file>  # Bad idea!
```

**Solutions:**
- Use external storage (S3, GCS, PVC)
- Store only reference/key in Secret

```yaml
# ✅ Store reference instead of file
apiVersion: v1
kind: Secret
metadata:
  name: storage-credentials
stringData:
  aws_access_key: "AKIA..."
  aws_secret_key: "..."
  s3_bucket: "my-app-secrets-bucket"
  file_path: "secrets/app-config.bin"
```

### 6. Environment Variables Not Updated Automatically

```yaml
# ❌ Env vars from secrets are set once at pod startup
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password

# If you update the secret:
# kubectl patch secret db-credentials -p '{"stringData":{"password":"newpass"}}'
# The pod's env vars are NOT updated!
```

**Solutions:**
- Use volume mounts (auto-update within 30-60 seconds)
- Restart pods to reload env vars
- Implement watch mechanism in application

```bash
# Force pod restart
kubectl rollout restart deployment api-service
```

### 7. Secrets Visible in Pod Spec

```bash
# ⚠️ Secret names visible in pod spec
kubectl describe pod my-pod
# Shows: "secret" : "db-credentials"

# Secret VALUES are not visible, but names are
```

### 8. Immutable Secrets Cannot Be Updated

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: immutable-secret
immutable: true  # Once set, cannot be changed
stringData:
  api_key: "sk_live_1234567890"

# ❌ Cannot update
kubectl patch secret immutable-secret -p '{"stringData":{"api_key":"new_key"}}'
# Error: field is immutable

# ✅ Solution: Create new secret with different name
```

---

## Best Practices

### 1. Use stringData for YAML Manifests

```yaml
# ✅ Recommended: stringData (auto base64-encoded)
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
stringData:
  password: "my-secure-password"  # Readable in YAML

# ❌ Avoid: data (requires manual base64)
# data:
#   password: bXktc2VjdXJlLXBhc3N3b3Jk
```

### 2. Separate Secrets from ConfigMaps

```yaml
# ❌ Never mix sensitive and non-sensitive
apiVersion: v1
kind: ConfigMap
data:
  API_ENDPOINT: "https://api.example.com"
  DB_PASSWORD: "secret123"  # WRONG!

# ✅ Separate clearly
# ConfigMap for non-sensitive
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  API_ENDPOINT: "https://api.example.com"

# Secret for sensitive
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
stringData:
  DB_PASSWORD: "secret123"
```

### 3. Use RBAC to Restrict Access

```yaml
# Limit who can read secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-secrets-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
  resourceNames: ["db-credentials", "api-keys"]  # Only these secrets
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-read-secrets
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-secrets-reader
subjects:
- kind: ServiceAccount
  name: app-service-account
  namespace: production
```

### 4. Enable Encryption at Rest

```bash
# Configure KMS encryption for secrets in etcd
# This requires modification of kube-apiserver configuration
# Recommended for production clusters
```

### 5. Use Volume Mounts for Files

```yaml
# ✅ Better for files (auto-updates within 30-60s)
volumeMounts:
- name: secrets
  mountPath: /app/secrets
  readOnly: true
volumes:
- name: secrets
  secret:
    secretName: app-secrets
    defaultMode: 0400  # Owner read-only

# ❌ Less ideal for files as env vars
env:
- name: TLS_CERT_FILE
  value: "-----BEGIN CERT-----..."  # Large content in env var
```

### 6. Set Appropriate File Permissions

```yaml
volumes:
- name: secrets
  secret:
    secretName: my-secret
    defaultMode: 0400  # Owner read-only (recommended)
    items:
    - key: id_rsa
      path: id_rsa
      mode: 0400      # SSH key: read-only for owner
    - key: config
      path: config
      mode: 0644      # Config file: readable
```

### 7. Implement Secret Rotation

```bash
# Monthly secret rotation process
# 1. Generate new credentials
# 2. Create new secret
kubectl create secret generic db-credentials-v2 \
  --from-literal=username=newuser \
  --from-literal=password=newpass

# 3. Update pod specification
kubectl patch deployment api-service --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/volumes/0/secret/secretName", "value":"db-credentials-v2"}]'

# 4. Wait for rollout
kubectl rollout status deployment/api-service

# 5. Delete old secret
kubectl delete secret db-credentials
```

### 8. Document Secret Usage

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  annotations:
    description: "Application secrets for production"
    documentation: |
      Used by: api-service, payment-processor
      Rotation frequency: 90 days
      Owner: platform-security-team
      Emergency contact: oncall@company.com
      Last rotated: 2024-01-15
```

### 9. Never Commit Secrets to Git

```bash
# ✅ Add to .gitignore
echo "*.key" >> .gitignore
echo "secrets/" >> .gitignore
echo ".env" >> .gitignore

# ✅ Use git-secrets hook to prevent accidents
git config core.hooksPath hooks
```

### 10. Use External Secrets Manager

```yaml
# For highly sensitive environments, use external secrets
# HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, etc.

# SecretStore pointing to external system
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.company.com"
      path: "secret"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "my-app-role"

---
# Syncs external secret to Kubernetes Secret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: db-credentials  # Creates this K8s Secret
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: database/prod
      property: username
  - secretKey: password
    remoteRef:
      key: database/prod
      property: password
```

---

## Common Troubleshooting

### Problem: Secret Not Found Error

```bash
# Error: secrets "my-secret" not found
kubectl describe pod app-pod

# Causes:
# 1. Secret doesn't exist in the namespace
kubectl get secret -n production

# 2. Wrong namespace referenced
kubectl get pod -n production

# 3. Typo in secret name

# Solutions:
# Create missing secret
kubectl create secret generic my-secret --from-literal=key=value -n production

# Verify secret exists
kubectl get secret my-secret -n production -o yaml
```

### Problem: Pod Can't Decode Secret Value

```bash
# Error inside pod: Invalid UTF-8 sequence
# Cause: Binary data not properly base64-encoded

# Check secret encoding
kubectl get secret my-secret -o jsonpath='{.data.mykey}'

# Manually verify base64
echo "value-from-secret" | base64 -d
```

### Problem: Permission Denied When Mounting Secret

```bash
# Error: Permission denied reading /etc/secrets/file
# Cause: File permissions too restrictive or wrong

# Check pod's user
kubectl exec app-pod -- id
# uid=1000(app) gid=3000(app) groups=3000(app)

# Update secret permissions
kubectl patch secret my-secret --type=merge \
  -p '{"metadata":{"managedFields":null}}'

# Or recreate with correct permissions
```

### Problem: Large Secret Exceeds 1MB

```bash
# Error: The Secret "huge" is invalid: data: Too long

# Solution: Split into multiple secrets
kubectl create secret generic secret-part1 --from-file=file1.crt=...
kubectl create secret generic secret-part2 --from-file=file2.crt=...

# Reference both in pod
volumeMounts:
- name: certs1
  mountPath: /etc/certs1
- name: certs2
  mountPath: /etc/certs2
volumes:
- name: certs1
  secret:
    secretName: secret-part1
- name: certs2
  secret:
    secretName: secret-part2
```

---

## Summary

Kubernetes Secrets are essential for managing **sensitive data** securely. They provide:
- **Separation of concerns** → Sensitive data away from code and configs
- **Access control** → RBAC to restrict who can read secrets
- **Encryption ready** → Can be encrypted at rest with KMS
- **Multiple types** → For different credential types (TLS, Docker, SSH, etc.)
- **Flexible injection** → As env vars or volume-mounted files

**Key takeaways:**
- Use Secret for **sensitive** data (passwords, tokens, keys)
- Use ConfigMap for **non-sensitive** configuration
- Base64 encoding ≠ encryption (enable KMS for real encryption)
- Implement RBAC to restrict secret access
- Rotate secrets regularly (manually or with external system)
- Never commit secrets to Git
- Use volume mounts for auto-updates (~30-60 seconds)
- Consider external secrets manager for production

**When to use Secrets:**
- Database credentials
- API keys and tokens
- TLS/SSL certificates
- SSH keys
- Docker registry credentials
- Any sensitive data needed by pods

Secrets, combined with ConfigMaps and RBAC, form a complete secrets management strategy in Kubernetes.
