
# Kubernetes ServiceAccount: Complete Guide

> A deep dive into Kubernetes ServiceAccount with real-world scenarios, practical examples, and visual explanations

---

## Table of Contents

1. [Scenario Overview](#scenario-overview)
2. [What is a ServiceAccount?](#what-is-a-serviceaccount)
3. [Why Do We Need ServiceAccount?](#why-do-we-need-serviceaccount)
4. [Key Benefits](#key-benefits)
5. [How ServiceAccount Works](#how-serviceaccount-works)
6. [YAML Specifications](#yaml-specifications)
7. [Practical Examples](#practical-examples)
8. [Common Use Cases](#common-use-cases)
9. [Best Practices](#best-practices)
10. [Troubleshooting](#troubleshooting)

---

## Scenario Overview

### Real-World Scenario 1: Monitoring Application

**The Problem:**
You have a Kubernetes cluster with 10+ microservices. You want to deploy Prometheus to collect metrics from all pods. However:
- Prometheus needs permission to read pod metrics from the API server
- You don't want to use the default service account
- You want fine-grained control over what resources Prometheus can access
- You need audit trails for security compliance

**The Solution:**
Create a dedicated ServiceAccount for Prometheus with specific Role-based permissions (RBAC).

---

### Real-World Scenario 2: CI/CD Pipeline

**The Problem:**
Your Jenkins deployment inside Kubernetes needs to:
- Deploy applications to the cluster
- Scale deployments up/down
- Access pod logs
- Create/delete resources

If you use the default ServiceAccount, it has too few permissions. If you use the cluster admin account, it violates security best practices.

**The Solution:**
Create a ServiceAccount with a custom Role that grants only the necessary permissions.

---

### Real-World Scenario 3: Multi-Tenant Cluster

**The Problem:**
You're hosting multiple teams' applications in one cluster:
- Team A works on payments
- Team B works on notifications
- You need to prevent Team A from accessing Team B's secrets or pods

**The Solution:**
Create separate ServiceAccounts for each team's namespace with namespace-scoped RBAC permissions.

---

## What is a ServiceAccount?

### Definition

A **ServiceAccount** is a Kubernetes identity used by applications (pods) running in the cluster to authenticate with the Kubernetes API server. It's similar to a user account but designed for automated processes instead of humans.

### Simple Analogy

Imagine your Kubernetes cluster as an office building:
- **User Account** = Employee ID badge for people entering the building
- **ServiceAccount** = Robot/Bot ID badge for automated systems entering the building

Both authenticate and have specific permissions about what they can access.

### Key Characteristics

| Aspect | Detail |
|--------|--------|
| **Scope** | Namespace-specific (each namespace has its own ServiceAccounts) |
| **Authentication** | Uses JWT tokens stored in Secrets |
| **Authorization** | Works with RBAC (Role-Based Access Control) |
| **Default** | Every namespace gets a `default` ServiceAccount automatically |
| **Mounted** | Automatically mounted to pods as volumes |

---

## Why Do We Need ServiceAccount?

### The Security Problem

Without ServiceAccounts, how would you handle this?

```
Pod inside Kubernetes
        ↓
Needs to call Kubernetes API
        ↓
How does API server know the pod is legitimate?
        ↓
How do you control what that pod can do?
```

### The Access Control Gap

**Without ServiceAccounts:**
- ❌ All pods would need the same credentials
- ❌ No way to restrict what pods can access
- ❌ No audit trail of who did what
- ❌ Impossible to implement the principle of least privilege

**With ServiceAccounts:**
- ✅ Each pod can have its own identity
- ✅ Fine-grained permissions per pod/application
- ✅ Full audit trail of API calls
- ✅ Secure token management

---

## Key Benefits

### 1. **Identity & Authentication**
```
Pod with ServiceAccount
    ↓
JWT Token (auto-injected)
    ↓
Kubernetes API Server
    ↓
"Yes, you are who you claim to be"
```

### 2. **Authorization & RBAC**
```
Authenticated Pod
    ↓
"Check what this ServiceAccount can do"
    ↓
Role/RoleBinding attached to ServiceAccount
    ↓
"You can read pods, but not delete deployments"
```

### 3. **Principle of Least Privilege**
Each ServiceAccount gets only the minimum permissions it needs:
- Jenkins SA: Can deploy, scale, read logs
- Prometheus SA: Can read metrics, watch pods
- App SA: Can read ConfigMaps and Secrets only

### 4. **Security Isolation**
- Segregate different applications
- Team A cannot impersonate Team B
- Compromised pod cannot access everything

### 5. **Audit & Compliance**
```
API Server Audit Log:
2025-03-27 10:15:23 | system:serviceaccount:monitoring:prometheus | GET /api/v1/pods | ALLOW
2025-03-27 10:15:24 | system:serviceaccount:ci:jenkins | POST /api/v1/deployments | ALLOW
```

### 6. **Automatic Token Management**
- Tokens generated automatically
- Mounted into pods automatically
- No manual secret distribution needed

---

## How ServiceAccount Works

### Architecture Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                  Namespace: default                       │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │                                                           │  │
│  │  ┌─────────────────────────────────────────────────────┐ │  │
│  │  │  ServiceAccount: "my-app"                           │ │  │
│  │  │  ├─ Username: system:serviceaccount:default:my-app │ │  │
│  │  │  └─ Secret: my-app-token-xyz                       │ │  │
│  │  └─────────────────────────────────────────────────────┘ │  │
│  │           ↓ (linked via RoleBinding)                      │  │
│  │  ┌─────────────────────────────────────────────────────┐ │  │
│  │  │  Role: "read-pods"                                  │ │  │
│  │  │  ├─ rules[0]:                                       │ │  │
│  │  │  │  - apiGroups: [""]                             │ │  │
│  │  │  │  - resources: ["pods"]                         │ │  │
│  │  │  │  - verbs: ["get", "list", "watch"]            │ │  │
│  │  └─────────────────────────────────────────────────────┘ │  │
│  │                                                           │  │
│  │  ┌─────────────────────────────────────────────────────┐ │  │
│  │  │  Pod: "my-app-deployment-xyz"                       │ │  │
│  │  │  ├─ Container: my-app                              │ │  │
│  │  │  ├─ serviceAccountName: "my-app"                   │ │  │
│  │  │  └─ Volume Mount:                                  │ │  │
│  │  │     ├─ /var/run/secrets/kubernetes.io/serviceaccount │  │
│  │  │     │  ├─ token                                    │ │  │
│  │  │     │  ├─ ca.crt                                   │ │  │
│  │  │     │  └─ namespace                                │ │  │
│  │  └─────────────────────────────────────────────────────┘ │  │
│  │                                                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Step-by-Step Flow

#### Step 1: Creation
```yaml
# You create a ServiceAccount
kubectl create serviceaccount my-app
```

#### Step 2: Secret Generation
```
Kubernetes automatically creates:
├─ Secret (contains JWT token)
├─ CA Certificate
└─ Namespace file
```

#### Step 3: Pod Injection
```yaml
# When you create a pod
Pod Spec:
  serviceAccountName: my-app

Kubernetes automatically:
├─ Injects token into /var/run/secrets/...
├─ Sets KUBERNETES_SERVICE_HOST env
├─ Sets KUBERNETES_SERVICE_PORT env
└─ Makes pod ready to call API
```

#### Step 4: Authentication
```
Pod makes API call:
  curl -H "Authorization: Bearer $TOKEN" \
       --cacert /var/run/secrets/.../ca.crt \
       https://kubernetes.default.svc/api/v1/namespaces/default/pods

API Server:
  ├─ Verifies JWT signature ✓
  ├─ Identifies ServiceAccount ✓
  └─ Checks RBAC permissions...
```

#### Step 5: Authorization
```
RBAC Check:
  ├─ Find RoleBindings for ServiceAccount
  ├─ Find Roles referenced by those RoleBindings
  ├─ Check if action matches any Rule
  ├─ If match found → ALLOW
  └─ If no match → DENY
```

### Detailed Diagram: Token Flow

```
┌─────────────────────────────────────────────────────────────┐
│                     Pod (Container)                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Application Code:                                         │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ import os                                            │ │
│  │ token = open('/var/run/secrets/.../token').read()  │ │
│  │ headers = {"Authorization": f"Bearer {token}"}      │ │
│  │ response = requests.get(                            │ │
│  │   "https://kubernetes.default.svc/api/v1/pods",   │ │
│  │   headers=headers                                  │ │
│  │ )                                                   │ │
│  └──────────────────────────────────────────────────────┘ │
│                           ↓                                │
│  Volume Mount Injection (by kubelet):                     │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ /var/run/secrets/kubernetes.io/serviceaccount/      │ │
│  │ ├─ token (JWT)                                      │ │
│  │ ├─ ca.crt (Certificate)                             │ │
│  │ └─ namespace (Namespace name)                        │ │
│  └──────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                           ↓
                  (API Call with Token)
                           ↓
┌─────────────────────────────────────────────────────────────┐
│            Kubernetes API Server                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Step 1: Authentication                                   │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ Verify JWT Token:                                    │ │
│  │ ├─ Check signature (using public key) ✓             │ │
│  │ ├─ Check expiration ✓                               │ │
│  │ ├─ Extract claims:                                  │ │
│  │ │  - Subject: system:serviceaccount:default:my-app │ │
│  │ │  - UID: abc123xyz                                │ │
│  │ └─ Result: AUTHENTICATED                            │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                             │
│  Step 2: Authorization (RBAC)                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ Check RoleBindings:                                  │ │
│  │ ├─ Find ServiceAccount: my-app ✓                    │ │
│  │ ├─ Find RoleBindings → my-app-binding ✓             │ │
│  │ ├─ Get Role → read-pods                             │ │
│  │ ├─ Check Rules:                                      │ │
│  │ │  - Verb: GET ✓                                    │ │
│  │ │  - Resource: pods ✓                               │ │
│  │ │  - APIGroup: "" ✓                                 │ │
│  │ └─ Result: ALLOWED                                   │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                             │
│  Step 3: Execute Request                                  │
│  ├─ Fetch pods from etcd ✓                               │
│  └─ Return results ✓                                      │
└─────────────────────────────────────────────────────────────┘
```

---

## YAML Specifications

### 1. Creating a ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: default
  labels:
    app: my-app
    version: v1
  annotations:
    description: "ServiceAccount for my-app deployment"
```

**Command equivalent:**
```bash
kubectl create serviceaccount my-app -n default
```

### 2. Creating a Role (Permissions Definition)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: read-pods-role
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/logs"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
```

**Explanation:**
| Field | Meaning |
|-------|---------|
| `apiGroups: [""]` | Core API group (empty string = core APIs like pods, configmaps) |
| `apiGroups: ["apps"]` | Apps API group (deployments, statefulsets, daemonsets) |
| `resources: ["pods"]` | Which resource type |
| `verbs: ["get", "list", "watch"]` | Which actions allowed |

**Common Verbs:**
- `get` - Read single resource
- `list` - List resources
- `watch` - Watch for changes in real-time
- `create` - Create new resource
- `update` - Update existing resource
- `patch` - Partially update
- `delete` - Delete resource

### 3. Creating a RoleBinding (Connect SA to Role)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-read-pods-binding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: read-pods-role
subjects:
- kind: ServiceAccount
  name: my-app
  namespace: default
```

**What it means:**
```
"Give the 'read-pods-role' Role to 'my-app' ServiceAccount in 'default' namespace"
```

### 4. Pod Using ServiceAccount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
  namespace: default
spec:
  serviceAccountName: my-app
  automountServiceAccountToken: true
  containers:
  - name: app
    image: my-app:latest
    volumeMounts:
    - name: sa-token
      mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      readOnly: true
  volumes:
  - name: sa-token
    projected:
      sources:
      - serviceAccountToken:
          audience: https://kubernetes.default.svc
          expirationSeconds: 3600
          path: token
```

**Explanation:**
| Field | Meaning |
|-------|---------|
| `serviceAccountName: my-app` | Use this ServiceAccount |
| `automountServiceAccountToken: true` | Auto-mount token (default) |
| `automountServiceAccountToken: false` | Don't mount token (security) |

### 5. Cluster-Wide Role (ClusterRole)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-monitoring
rules:
- apiGroups: [""]
  resources: ["nodes", "pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["metrics.k8s.io"]
  resources: ["pods", "nodes"]
  verbs: ["get", "list"]
```

**Difference from Role:**
- `Role` = Namespace-scoped permissions
- `ClusterRole` = Cluster-wide permissions

### 6. ClusterRoleBinding (Cluster-Wide Binding)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-monitoring
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-monitoring
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
```

---

## Practical Examples

### Example 1: Simple Read-Only Application

**Scenario:** Your app needs to read pod information but not modify anything.

**Step 1: Create ServiceAccount**
```yaml
# sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-readonly
  namespace: default
```

**Step 2: Create Role**
```yaml
# role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list"]
```

**Step 3: Create RoleBinding**
```yaml
# rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-read-pods
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
- kind: ServiceAccount
  name: app-readonly
  namespace: default
```

**Step 4: Deploy Pod**
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: app-readonly
      containers:
      - name: app
        image: my-app:1.0
        env:
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
```

**Step 5: Application Code**
```python
import os
import requests
from kubernetes import client, config

# Read ServiceAccount token
with open('/var/run/secrets/kubernetes.io/serviceaccount/token', 'r') as f:
    token = f.read()

# Read CA certificate
with open('/var/run/secrets/kubernetes.io/serviceaccount/ca.crt', 'r') as f:
    ca_cert = f.read()

# Make API request
headers = {"Authorization": f"Bearer {token}"}
response = requests.get(
    "https://kubernetes.default.svc/api/v1/pods",
    headers=headers,
    verify="/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
)

print("Pods in cluster:")
for pod in response.json()['items']:
    print(f"- {pod['metadata']['name']}")
```

**Apply & Test:**
```bash
# Apply YAML files
kubectl apply -f sa.yaml
kubectl apply -f role.yaml
kubectl apply -f rolebinding.yaml
kubectl apply -f deployment.yaml

# Get pod logs
kubectl logs -f deployment/my-app

# Verify permissions
kubectl auth can-i get pods --as=system:serviceaccount:default:app-readonly
# Output: yes
```

---

### Example 2: Jenkins CI/CD with Limited Permissions

**Scenario:** Jenkins needs to deploy, update, and manage deployments but not access secrets.

**Complete manifest:**
```yaml
---
# Jenkins ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: ci-cd
  labels:
    app: jenkins

---
# Jenkins Role with deployment permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: jenkins-deployer
  namespace: ci-cd
rules:
# Can manage deployments
- apiGroups: ["apps"]
  resources: ["deployments", "deployments/scale"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
  
# Can manage pods (restart, etc.)
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
  
# Can manage services
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "create", "update", "patch"]
  
# Can view pod logs
- apiGroups: [""]
  resources: ["pods/logs"]
  verbs: ["get", "list"]
  
# Can manage configmaps (but NOT secrets!)
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "create", "update"]

---
# Bind role to ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-deployer-binding
  namespace: ci-cd
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: jenkins-deployer
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: ci-cd

---
# Jenkins Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: ci-cd
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: jenkins-home
          mountPath: /var/jenkins_home
      volumes:
      - name: jenkins-home
        emptyDir: {}
```

**Verification:**
```bash
# Apply
kubectl apply -f jenkins-manifest.yaml

# What CAN Jenkins do?
kubectl auth can-i create deployments --as=system:serviceaccount:ci-cd:jenkins -n ci-cd
# Output: yes

kubectl auth can-i delete pods --as=system:serviceaccount:ci-cd:jenkins -n ci-cd
# Output: yes

# What CAN'T Jenkins do?
kubectl auth can-i get secrets --as=system:serviceaccount:ci-cd:jenkins -n ci-cd
# Output: no

kubectl auth can-i delete nodes --as=system:serviceaccount:ci-cd:jenkins -n ci-cd
# Output: no (cross-namespace, requires ClusterRole)
```

---

### Example 3: Multi-Tenant with Namespace Isolation

**Scenario:** Team A and Team B work on the same cluster but in different namespaces.

```yaml
---
# Namespace for Team A
apiVersion: v1
kind: Namespace
metadata:
  name: team-a

---
# Namespace for Team B
apiVersion: v1
kind: Namespace
metadata:
  name: team-b

---
# Team A ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: team-a-app
  namespace: team-a

---
# Team B ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: team-b-app
  namespace: team-b

---
# Team A Role - can only see own namespace resources
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: team-a-role
  namespace: team-a
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list"]

---
# Team B Role - can only see own namespace resources
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: team-b-role
  namespace: team-b
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list"]

---
# Team A Binding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-binding
  namespace: team-a
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: team-a-role
subjects:
- kind: ServiceAccount
  name: team-a-app
  namespace: team-a

---
# Team B Binding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-b-binding
  namespace: team-b
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: team-b-role
subjects:
- kind: ServiceAccount
  name: team-b-app
  namespace: team-b

---
# Team A Pod
apiVersion: v1
kind: Pod
metadata:
  name: team-a-pod
  namespace: team-a
spec:
  serviceAccountName: team-a-app
  containers:
  - name: app
    image: team-a-app:1.0

---
# Team B Pod
apiVersion: v1
kind: Pod
metadata:
  name: team-b-pod
  namespace: team-b
spec:
  serviceAccountName: team-b-app
  containers:
  - name: app
    image: team-b-app:1.0
```

**Security test:**
```bash
# Team A can see Team A's pods
kubectl auth can-i get pods --as=system:serviceaccount:team-a:team-a-app -n team-a
# Output: yes

# Team A CANNOT see Team B's pods
kubectl auth can-i get pods --as=system:serviceaccount:team-a:team-a-app -n team-b
# Output: no

# Team B CANNOT see Team A's pods
kubectl auth can-i get pods --as=system:serviceaccount:team-b:team-b-app -n team-a
# Output: no
```

---

### Example 4: Prometheus Cluster Monitoring

**Scenario:** Prometheus needs to collect metrics from all nodes and pods across the cluster.

```yaml
---
# Monitoring namespace
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring

---
# Prometheus ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring

---
# Cluster-wide permissions for Prometheus
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
# Read pods
- apiGroups: [""]
  resources: ["nodes", "nodes/proxy", "pods"]
  verbs: ["get", "list", "watch"]

# Read metrics
- apiGroups: [""]
  resources: ["services", "endpoints"]
  verbs: ["get", "list", "watch"]

# Metrics API
- apiGroups: ["extensions"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch"]

---
# Bind to all namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring

---
# Prometheus Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        args:
          - "--config.file=/etc/prometheus/prometheus.yml"
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config
          mountPath: /etc/prometheus
        - name: storage
          mountPath: /prometheus
      volumes:
      - name: config
        configMap:
          name: prometheus-config
      - name: storage
        emptyDir: {}

---
# Prometheus Config
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
```

**Verification:**
```bash
# Apply
kubectl apply -f prometheus-manifest.yaml

# Check Prometheus can access all namespaces
kubectl auth can-i watch pods --as=system:serviceaccount:monitoring:prometheus
# Output: yes

kubectl auth can-i get nodes --as=system:serviceaccount:monitoring:prometheus
# Output: yes

# Check Prometheus sees metrics
kubectl logs -f -n monitoring deployment/prometheus
```

---

## Common Use Cases

### Use Case 1: Application Accessing ConfigMaps

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-config-reader
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-config-reader-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: configmap-reader
subjects:
- kind: ServiceAccount
  name: app-config-reader
```

### Use Case 2: Backup Service (Needs Everything)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: backup-service
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list"]
```

### Use Case 3: Operator Managing Custom Resources

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: custom-operator
rules:
- apiGroups: ["mycompany.com"]
  resources: ["databases", "caches"]
  verbs: ["*"]
- apiGroups: ["apps"]
  resources: ["statefulsets"]
  verbs: ["get", "list", "create", "update", "delete"]
```

---

## Best Practices

### 1. **Always Use Specific ServiceAccounts**
❌ **Bad:**
```yaml
spec:
  serviceAccountName: default  # Default SA has minimal permissions
```

✅ **Good:**
```yaml
spec:
  serviceAccountName: my-app  # Custom SA with specific permissions
```

---

### 2. **Principle of Least Privilege**
❌ **Bad:**
```yaml
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]  # Can do EVERYTHING
```

✅ **Good:**
```yaml
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]  # Only what's needed
```

---

### 3. **Use Namespace-Scoped Roles When Possible**
❌ **Bad (Too Powerful):**
```yaml
kind: ClusterRole
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
```

✅ **Good (Namespace Scoped):**
```yaml
kind: Role
metadata:
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]  # Only in specific namespace
```

---

### 4. **Disable Auto-mounting When Not Needed**
```yaml
spec:
  serviceAccountName: my-app
  automountServiceAccountToken: false  # Pod doesn't need token
```

---

### 5. **Use ResourceQuota with ServiceAccounts**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: team-a
spec:
  hard:
    pods: "10"
    cpu: "100"
```

---

### 6. **Audit ServiceAccount Usage**
```bash
# See all ServiceAccounts
kubectl get serviceaccounts -A

# Get ServiceAccount details
kubectl describe sa my-app -n default

# List RoleBindings for a ServiceAccount
kubectl get rolebindings -A | grep my-app

# Test permissions
kubectl auth can-i get pods --as=system:serviceaccount:default:my-app
```

---

### 7. **Use Labels for Organization**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  labels:
    app: my-app
    team: backend
    environment: production
```

---

## Troubleshooting

### Problem 1: Pod Cannot Access API

**Symptom:**
```
Error: Unauthorized (401)
```

**Causes & Solutions:**

```bash
# 1. Check ServiceAccount exists
kubectl get sa my-app -n default

# 2. Check ServiceAccount is assigned to pod
kubectl get pod my-pod -o yaml | grep serviceAccountName

# 3. Check token is mounted
kubectl exec my-pod -- ls -la /var/run/secrets/kubernetes.io/serviceaccount/

# 4. Check token is valid
kubectl exec my-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

---

### Problem 2: Permission Denied

**Symptom:**
```
Error: Forbidden (403)
```

**Causes & Solutions:**

```bash
# 1. Check RoleBinding exists
kubectl get rolebindings -n default

# 2. Check Role has correct rules
kubectl get role my-role -o yaml

# 3. Test what SA can do
kubectl auth can-i get pods --as=system:serviceaccount:default:my-app
# If 'no' → permissions missing

# 4. Check RoleBinding references correct Role
kubectl describe rolebinding my-binding -n default

# 5. Verify ServiceAccount is in RoleBinding subjects
kubectl get rolebinding my-binding -o yaml | grep -A 5 subjects
```

---

### Problem 3: Token Expired

**Symptom:**
```
Error: Token has expired
```

**Solution:**
```bash
# Token auto-refreshes, but check token lifetime
kubectl get secret <sa-token-name> -o yaml

# For short-lived tokens, recreate
kubectl delete secret <sa-token-name>
kubectl apply -f serviceaccount.yaml
```

---

### Problem 4: Cross-Namespace Access Not Working

**Symptom:**
```
ServiceAccountName works in one namespace but not another
```

**Solution:**
```bash
# You need ClusterRole/ClusterRoleBinding
# Role only works within same namespace

# Use ClusterRole:
kubectl get clusterrolebindings | grep your-sa

# Create ClusterRoleBinding:
kubectl create clusterrolebinding my-access \
  --clusterrole=read-pods \
  --serviceaccount=ns1:my-sa
```

---

### Debugging Checklist

```bash
# Complete debugging flow
echo "1. Check ServiceAccount exists"
kubectl get sa <sa-name> -n <namespace>

echo "2. Check pod uses SA"
kubectl get pod <pod-name> -o yaml -n <namespace> | grep serviceAccount

echo "3. Check token mounted"
kubectl exec <pod-name> -n <namespace> -- \
  cat /var/run/secrets/kubernetes.io/serviceaccount/token

echo "4. Check RoleBindings"
kubectl get rolebindings -n <namespace> -o wide

echo "5. Test permissions"
kubectl auth can-i <verb> <resource> \
  --as=system:serviceaccount:<namespace>:<sa-name>

echo "6. Check Role/ClusterRole"
kubectl get role,clusterrole -n <namespace> | grep <sa-name>

echo "7. View API server logs"
kubectl logs -f -n kube-system deployment/kube-apiserver | grep <sa-name>
```

---

## Quick Reference Commands

### Create & Manage ServiceAccounts

```bash
# Create ServiceAccount
kubectl create serviceaccount my-app

# Get ServiceAccount
kubectl get serviceaccounts
kubectl get sa

# Describe ServiceAccount
kubectl describe sa my-app

# Delete ServiceAccount
kubectl delete sa my-app
```

### Create & Manage Roles/RoleBindings

```bash
# Create Role from manifest
kubectl apply -f role.yaml

# List Roles
kubectl get roles

# Describe Role
kubectl describe role pod-reader

# Delete Role
kubectl delete role pod-reader

# Create RoleBinding
kubectl apply -f rolebinding.yaml

# List RoleBindings
kubectl get rolebindings

# List ClusterRoleBindings
kubectl get clusterrolebindings
```

### Test Permissions

```bash
# Can I do this?
kubectl auth can-i get pods --as=system:serviceaccount:default:my-app

# Can I do this in another namespace?
kubectl auth can-i get pods -n other-ns --as=system:serviceaccount:default:my-app

# Dry run and see what would happen
kubectl get pods --dry-run=client -o yaml \
  --as=system:serviceaccount:default:my-app
```

### View Secret Generated by ServiceAccount

```bash
# ServiceAccounts auto-create a Secret with token
kubectl get secrets

# View token
kubectl get secret my-app-token-xyz -o jsonpath='{.data.token}' | base64 -d

# View CA certificate
kubectl get secret my-app-token-xyz -o jsonpath='{.data.ca\.crt}' | base64 -d
```

---

## Complete End-to-End Example

**All files needed to deploy a secure application:**

**1. namespace.yaml**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
```

**2. serviceaccount.yaml**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
  namespace: myapp
```

**3. role.yaml**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myapp-role
  namespace: myapp
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

**4. rolebinding.yaml**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-binding
  namespace: myapp
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: myapp-role
subjects:
- kind: ServiceAccount
  name: myapp
  namespace: myapp
```

**5. deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      serviceAccountName: myapp
      containers:
      - name: app
        image: myapp:1.0
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
```

**Deploy Everything:**
```bash
# Create namespace and resources
kubectl apply -f namespace.yaml
kubectl apply -f serviceaccount.yaml
kubectl apply -f role.yaml
kubectl apply -f rolebinding.yaml
kubectl apply -f deployment.yaml

# Verify
kubectl get all -n myapp

# Test permissions
kubectl auth can-i get configmaps --as=system:serviceaccount:myapp:myapp -n myapp
# Output: yes

kubectl auth can-i delete pods --as=system:serviceaccount:myapp:myapp -n myapp
# Output: no

# View logs
kubectl logs -f deployment/myapp -n myapp
```

---

## Summary

| Concept | What | Why | How |
|---------|------|-----|-----|
| **ServiceAccount** | Pod identity | Authenticate pods to API | Create SA, attach to pod |
| **Role** | Permission rules | Define what can be done | List resources & verbs |
| **RoleBinding** | SA + Role linkage | Apply rules to specific SA | Create binding with subjects |
| **Token** | Authentication credential | Prove identity to API | Auto-injected in pod |
| **RBAC** | Authorization system | Control what authenticated users can do | Roles + RoleBindings |

---

## Resources for Learning

- [Kubernetes RBAC Docs](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [ServiceAccount Docs](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
- [API Access Control](https://kubernetes.io/docs/concepts/security/controlling-access/)

---

**Created with ❤️ for Kubernetes learners**

Last Updated: March 27, 2025
