# Kubernetes RBAC (Role-Based Access Control)

## What is RBAC?

**RBAC** is a security system in Kubernetes that controls **who can do what** with your cluster resources.

Think of it like this: You have a company with different departments (developers, admins, monitoring). Each department needs different access levels. RBAC ensures developers can create pods but can't delete namespaces, while admins can do everything.

**In simple terms:** RBAC = Who + Can Do What + On Which Resources = Permission ✓

---

## RBAC Structure: Visual Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         KUBERNETES RBAC                             │
└─────────────────────────────────────────────────────────────────────┘

╔═══════════════════════════════════════════════════════════════════════╗
║                      SERVICE ACCOUNT                                 ║
║              (For Apps/Pods Running Inside Cluster)                  ║
╚═══════════════════════════════════════════════════════════════════════╝

              ↓ Creates ↓

┌─────────────────────────────────────────────────────────────────────┐
│                        NAMESPACE SCOPE                              │
│                  (Limited to Single Namespace)                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌─────────────┐         ┌──────────────┐                        │
│   │    ROLE     │ ←─ ─ →  │  ROLEBINDING │                        │
│   │  (Rules)    │  Binds   │  (Connect)   │                        │
│   └─────────────┘         └──────────────┘                        │
│                                 ↑                                   │
│                    Connects ServiceAccount                         │
│                      to Role Permissions                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

╔═══════════════════════════════════════════════════════════════════════╗
║                          USER / HUMAN                                ║
║              (For People Accessing the Cluster)                      ║
╚═══════════════════════════════════════════════════════════════════════╝

              ↓ Creates ↓

┌─────────────────────────────────────────────────────────────────────┐
│                     CLUSTER-WIDE SCOPE                              │
│                  (Access Across All Namespaces)                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌────────────────┐      ┌──────────────────┐                    │
│   │  CLUSTERROLE  │ ←─ ─ → │ CLUSTERROLEBIND │                    │
│   │   (Rules)     │  Binds  │    (Connect)    │                    │
│   └────────────────┘      └──────────────────┘                    │
│                                 ↑                                   │
│           Connects User/ServiceAccount                            │
│           to ClusterRole Permissions                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

```

---

## Why Do You Need RBAC?

Imagine your Kubernetes cluster without RBAC:
- Any person with cluster access could delete anything
- Apps running in pods could do anything they want
- No security or control
- Compliance violations

**RBAC solves this by:**

1. **Least Privilege Principle:** Users get only the permissions they need, nothing more
2. **Security:** Prevents accidental or malicious damage
3. **Multi-team Support:** Different teams can work safely in shared clusters
4. **Compliance:** Meets security requirements (SOC 2, HIPAA, PCI DSS, etc.)
5. **Audit Trail:** Clear record of who has access to what

---

## Benefits of RBAC

| Benefit | What It Means |
|---------|--------------|
| **Security** | Restricts unauthorized access to cluster resources |
| **Control** | Fine-grained permissions for specific resources and actions |
| **Compliance** | Meets security standards and auditing requirements |
| **Isolation** | Separates permissions between teams and environments |
| **Flexibility** | Works with users, groups, and service accounts |
| **Scalability** | Easy to manage permissions as your team grows |

---

## Prerequisites: What You Need to Know

Before working with RBAC, make sure you have:

1. **A running Kubernetes cluster** (local or cloud)
   ```bash
   # Check if your cluster is running
   kubectl cluster-info
   ```

2. **kubectl installed and configured**
   ```bash
   # Verify kubectl works
   kubectl version --client
   ```

3. **Basic kubectl commands knowledge**
   ```bash
   kubectl create
   kubectl apply
   kubectl describe
   kubectl get
   ```

4. **Understanding of Kubernetes namespaces** (optional but helpful)
   ```bash
   # Create a namespace for practice
   kubectl create namespace rbac-demo
   ```

5. **RBAC enabled on your cluster** (usually enabled by default)
   ```bash
   # Most modern clusters have RBAC enabled
   # No action needed in most cases
   ```

---

## RBAC Components Explained

### For SERVICE ACCOUNTS (Apps/Pods in Cluster)

#### 1. ServiceAccount (The "Who")

**What it is:** A Kubernetes identity for apps/processes running inside pods.

When a pod needs to talk to the Kubernetes API, it uses a ServiceAccount.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: default
```

#### 2. Role (Namespace Scope - Rules for One Namespace)

**What it is:** Defines permissions for resources in a **specific namespace only**.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

**What this means:** "Allow reading (get, list, watch) pods in the default namespace ONLY"

#### 3. RoleBinding (Namespace Scope - Connect ServiceAccount to Role)

**What it is:** Connects a ServiceAccount to a Role in a **specific namespace**.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: my-app-sa
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**What this means:** "Let ServiceAccount 'my-app-sa' use the permissions from Role 'pod-reader' in the default namespace"

---

### For USERS / HUMANS (People Accessing Cluster)

#### 1. ClusterRole (Cluster-Wide Scope - Rules for Entire Cluster)

**What it is:** Defines permissions for resources **across the entire cluster** (all namespaces).

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

**What this means:** "Allow reading pods in ALL namespaces across the entire cluster"

#### 2. ClusterRoleBinding (Cluster-Wide Scope - Connect User to ClusterRole)

**What it is:** Connects a User/ServiceAccount to a ClusterRole **across entire cluster**.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-read-pods-binding
subjects:
- kind: User
  name: john@example.com
roleRef:
  kind: ClusterRole
  name: cluster-pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**What this means:** "Let user 'john@example.com' use the permissions from ClusterRole 'cluster-pod-reader' in ALL namespaces"

---

## Comparison: ServiceAccount vs User RBAC

| Type | Scope | Used For | Binding | ClusterBinding |
|------|-------|----------|---------|----------------|
| **ServiceAccount** | Namespace or Cluster | Apps/Pods | RoleBinding (namespace) | ClusterRoleBinding (cluster) |
| **User** | Cluster-wide | People | (via ClusterRoleBinding) | ClusterRoleBinding (cluster) |

---

## RBAC Path Flow

### Path 1: ServiceAccount → Role → RoleBinding (Namespace Scope)

```
┌──────────────────┐
│ ServiceAccount   │
│ (in namespace)   │
└────────┬─────────┘
         │
         │ "Give me permissions"
         ↓
┌──────────────────┐
│ Role             │
│ (rules)          │
│ - get pods       │
│ - list pods      │
└────────┬─────────┘
         │
         │ "I'm bound to you"
         ↓
┌──────────────────┐
│ RoleBinding      │
│ (connects them)  │
│ Namespace: dev   │
└──────────────────┘

RESULT: ServiceAccount can perform Role permissions in that namespace only
```

### Path 2: User → ClusterRole → ClusterRoleBinding (Cluster-Wide Scope)

```
┌──────────────────┐
│ User/Person      │
│ (john@example)   │
└────────┬─────────┘
         │
         │ "Give me permissions everywhere"
         ↓
┌──────────────────┐
│ ClusterRole      │
│ (rules for all)  │
│ - get all pods   │
│ - list all nodes │
└────────┬─────────┘
         │
         │ "I'm bound to you everywhere"
         ↓
┌──────────────────┐
│ ClusterRoleBindin│
│ (connects them)  │
│ Scope: Cluster   │
└──────────────────┘

RESULT: User can perform ClusterRole permissions across entire cluster
```

---

## Core RBAC Verbs: What Actions Can Be Performed

These are the actions users/apps can perform on resources:

| Verb | Meaning | Example |
|------|---------|---------|
| `get` | Get a specific resource | Get a single pod by name |
| `list` | List all resources | List all pods in namespace |
| `watch` | Watch for changes in real-time | Watch pod status changes |
| `create` | Create a new resource | Create a new pod |
| `update` | Update an existing resource | Update pod configuration |
| `patch` | Partially update a resource | Change one pod setting |
| `delete` | Delete a resource | Delete a pod |
| `deletecollection` | Delete multiple resources | Delete all pods at once |
| `*` | All actions | Allow everything (use carefully!) |

---

## Common Resources in RBAC

These are the Kubernetes objects you can control access to:

```
pods                    # Pod objects
services                # Service objects
deployments             # Deployment objects
statefulsets            # StatefulSet objects
configmaps              # ConfigMap objects
secrets                 # Secret objects
namespaces              # Namespace objects
persistentvolumes       # Persistent volumes
persistentvolumeclaims  # Persistent volume claims
nodes                   # Node objects
```

---

# 🎯 HANDS-ON PRACTICAL EXERCISES

## Hands-On Exercise 1: Create Your First ServiceAccount RBAC Setup (Namespace Scope)

### Objective
Create a ServiceAccount with permission to read pods in a namespace, then test if it works.

### Prerequisites
- A running Kubernetes cluster
- kubectl installed

### Step-by-Step Instructions

**Step 1: Create a namespace for practice**

```bash
kubectl create namespace rbac-practice
```

Verify it was created:
```bash
kubectl get namespace rbac-practice
```

**Step 2: Create a ServiceAccount**

```bash
kubectl create serviceaccount pod-reader-sa -n rbac-practice
```

Verify the ServiceAccount:
```bash
kubectl get serviceaccount -n rbac-practice
kubectl describe serviceaccount pod-reader-sa -n rbac-practice
```

**Step 3: Create a Role with pod read permissions**

Save this file as `role-pod-reader.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader-role
  namespace: rbac-practice
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

Apply it:
```bash
kubectl apply -f role-pod-reader.yaml
```

Verify the Role:
```bash
kubectl get role -n rbac-practice
kubectl describe role pod-reader-role -n rbac-practice
```

**Step 4: Create a RoleBinding to connect ServiceAccount to Role**

Save this file as `rolebinding-pod-reader.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: rbac-practice
subjects:
- kind: ServiceAccount
  name: pod-reader-sa
  namespace: rbac-practice
roleRef:
  kind: Role
  name: pod-reader-role
  apiGroup: rbac.authorization.k8s.io
```

Apply it:
```bash
kubectl apply -f rolebinding-pod-reader.yaml
```

Verify the RoleBinding:
```bash
kubectl get rolebinding -n rbac-practice
kubectl describe rolebinding pod-reader-binding -n rbac-practice
```

**Step 5: Verify permissions with kubectl auth can-i**

Check if the ServiceAccount can read pods:
```bash
kubectl auth can-i get pods --as=system:serviceaccount:rbac-practice:pod-reader-sa -n rbac-practice
# Output: yes ✓
```

Check if the ServiceAccount can delete pods (should fail):
```bash
kubectl auth can-i delete pods --as=system:serviceaccount:rbac-practice:pod-reader-sa -n rbac-practice
# Output: no ✗
```

**Congratulations!** You've successfully created your first ServiceAccount RBAC setup. ✓

---

## Hands-On Exercise 2: Test RBAC with a Real Pod

### Objective
Create a pod using the ServiceAccount and verify it can access the Kubernetes API.

### Prerequisites
- Completed Exercise 1
- Basic understanding of curl command

### Step-by-Step Instructions

**Step 1: Create some test pods in the namespace**

```bash
kubectl run test-pod-1 --image=nginx -n rbac-practice
kubectl run test-pod-2 --image=nginx -n rbac-practice
```

Verify they're running:
```bash
kubectl get pods -n rbac-practice
```

**Step 2: Create a debugging pod with the ServiceAccount**

Save this file as `debug-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug-pod
  namespace: rbac-practice
spec:
  serviceAccountName: pod-reader-sa
  containers:
  - name: debug
    image: curlimages/curl
    command: ['sh', '-c', 'sleep 3600']
```

Apply it:
```bash
kubectl apply -f debug-pod.yaml
```

Wait for the pod to be ready:
```bash
kubectl get pods -n rbac-practice
```

**Step 3: Get into the pod and test permissions**

```bash
# Get into the pod
kubectl exec -it debug-pod -n rbac-practice -- sh

# Inside the pod, run these commands:

# Store the token
TOKEN=$(cat /run/secrets/kubernetes.io/serviceaccount/token)

# Get the API server URL
API_SERVER=https://kubernetes.default.svc.cluster.local

# List all pods in rbac-practice namespace
curl -H "Authorization: Bearer $TOKEN" \
  -k $API_SERVER/api/v1/namespaces/rbac-practice/pods

# You should see the pods list! ✓
```

**Step 4: Try to delete a pod (should fail)**

Still inside the pod:

```bash
# Try to delete a pod
curl -X DELETE \
  -H "Authorization: Bearer $TOKEN" \
  -k $API_SERVER/api/v1/namespaces/rbac-practice/pods/test-pod-1

# You'll get a 403 Forbidden error ✓
# This proves the RBAC is working!
```

**Step 5: Exit the pod**

```bash
exit
```

**Congratulations!** You've tested RBAC with a real pod. ✓

---

## Hands-On Exercise 3: Create Write Permissions for ServiceAccount

### Objective
Create a ServiceAccount Role that allows creating and deleting pods, then test it.

### Prerequisites
- Completed Exercises 1 & 2

### Step-by-Step Instructions

**Step 1: Create a new ServiceAccount for write operations**

```bash
kubectl create serviceaccount pod-writer-sa -n rbac-practice
```

**Step 2: Create a Role with write permissions**

Save this file as `role-pod-writer.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-writer-role
  namespace: rbac-practice
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "create", "delete", "update", "patch"]
```

Apply it:
```bash
kubectl apply -f role-pod-writer.yaml
```

**Step 3: Create a RoleBinding**

Save this file as `rolebinding-pod-writer.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-writer-binding
  namespace: rbac-practice
subjects:
- kind: ServiceAccount
  name: pod-writer-sa
  namespace: rbac-practice
roleRef:
  kind: Role
  name: pod-writer-role
  apiGroup: rbac.authorization.k8s.io
```

Apply it:
```bash
kubectl apply -f rolebinding-pod-writer.yaml
```

**Step 4: Test the permissions**

Test if the ServiceAccount can create pods:
```bash
kubectl auth can-i create pods --as=system:serviceaccount:rbac-practice:pod-writer-sa -n rbac-practice
# Output: yes ✓
```

Test if the ServiceAccount can delete pods:
```bash
kubectl auth can-i delete pods --as=system:serviceaccount:rbac-practice:pod-writer-sa -n rbac-practice
# Output: yes ✓
```

Test if the pod-reader-sa (from Exercise 1) can delete pods:
```bash
kubectl auth can-i delete pods --as=system:serviceaccount:rbac-practice:pod-reader-sa -n rbac-practice
# Output: no ✗
```

**Congratulations!** You've created write permissions for ServiceAccount. ✓

---

## Hands-On Exercise 4: Create ClusterRole for User Access (Cluster-Wide)

### Objective
Create a ClusterRole for cluster-wide pod reading access across all namespaces.

### Prerequisites
- Completed Exercise 1

### Step-by-Step Instructions

**Step 1: Create a ClusterRole**

Save this file as `clusterrole-pod-reader.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

Apply it:
```bash
kubectl apply -f clusterrole-pod-reader.yaml
```

Verify it:
```bash
kubectl get clusterrole cluster-pod-reader
kubectl describe clusterrole cluster-pod-reader
```

**Step 2: Create a ClusterRoleBinding for a User**

Save this file as `clusterrolebinding-user.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-pod-reader-user-binding
subjects:
# For a real user (person)
- kind: User
  name: john@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Apply it:
```bash
kubectl apply -f clusterrolebinding-user.yaml
```

Verify it:
```bash
kubectl get clusterrolebinding cluster-pod-reader-user-binding
kubectl describe clusterrolebinding cluster-pod-reader-user-binding
```

**Step 3: Test cluster-wide access**

Test if user john@example.com can read pods in rbac-practice namespace:
```bash
kubectl auth can-i get pods --as=john@example.com -n rbac-practice
# Output: yes ✓
```

Test if user can read pods in default namespace:
```bash
kubectl auth can-i get pods --as=john@example.com -n default
# Output: yes ✓ (Works in other namespaces too!)
```

Test if user can read pods in kube-system namespace:
```bash
kubectl auth can-i get pods --as=john@example.com -n kube-system
# Output: yes ✓ (Works in system namespace too!)
```

**Congratulations!** You've created cluster-wide permissions for a user. ✓

---

## Hands-On Exercise 5: Multiple Permissions in One Role

### Objective
Create a Role with access to multiple resources (pods, services, configmaps) for ServiceAccount.

### Prerequisites
- Completed Exercise 1

### Step-by-Step Instructions

**Step 1: Create a new ServiceAccount**

```bash
kubectl create serviceaccount multi-reader-sa -n rbac-practice
```

**Step 2: Create a Role with multiple resource permissions**

Save this file as `role-multi-resource.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: multi-resource-reader
  namespace: rbac-practice
rules:
# Allow reading pods
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

# Allow reading services
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list"]

# Allow reading configmaps
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]

# Allow reading deployments
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list"]
```

Apply it:
```bash
kubectl apply -f role-multi-resource.yaml
```

**Step 3: Create a RoleBinding**

Save this file as `rolebinding-multi-resource.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: multi-resource-binding
  namespace: rbac-practice
subjects:
- kind: ServiceAccount
  name: multi-reader-sa
  namespace: rbac-practice
roleRef:
  kind: Role
  name: multi-resource-reader
  apiGroup: rbac.authorization.k8s.io
```

Apply it:
```bash
kubectl apply -f rolebinding-multi-resource.yaml
```

**Step 4: Test multiple permissions**

Test reading pods:
```bash
kubectl auth can-i get pods --as=system:serviceaccount:rbac-practice:multi-reader-sa -n rbac-practice
# Output: yes ✓
```

Test reading services:
```bash
kubectl auth can-i get services --as=system:serviceaccount:rbac-practice:multi-reader-sa -n rbac-practice
# Output: yes ✓
```

Test reading configmaps:
```bash
kubectl auth can-i get configmaps --as=system:serviceaccount:rbac-practice:multi-reader-sa -n rbac-practice
# Output: yes ✓
```

Test delete pods (should fail):
```bash
kubectl auth can-i delete pods --as=system:serviceaccount:rbac-practice:multi-reader-sa -n rbac-practice
# Output: no ✗
```

**Congratulations!** You've created multiple permissions in one Role. ✓

---

## Hands-On Exercise 6: Restrict Permissions (What NOT to Allow)

### Objective
Create a Developer Role with specific permissions but explicitly NO access to secrets.

### Prerequisites
- Completed Exercise 1

### Step-by-Step Instructions

**Step 1: Create a new ServiceAccount**

```bash
kubectl create serviceaccount developer-sa -n rbac-practice
```

**Step 2: Create a Role for developer (can manage pods, services, but NOT secrets)**

Save this file as `role-developer.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: rbac-practice
rules:
# Allow managing pods
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "create", "delete", "update", "patch"]

# Allow managing services
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "create", "delete"]

# Allow managing deployments
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]

# Explicitly NO access to secrets
# (by not including secrets in the rules)
```

Apply it:
```bash
kubectl apply -f role-developer.yaml
```

**Step 3: Create a RoleBinding**

Save this file as `rolebinding-developer.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: rbac-practice
subjects:
- kind: ServiceAccount
  name: developer-sa
  namespace: rbac-practice
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

Apply it:
```bash
kubectl apply -f rolebinding-developer.yaml
```

**Step 4: Test restrictions**

Test if developer can create pods:
```bash
kubectl auth can-i create pods --as=system:serviceaccount:rbac-practice:developer-sa -n rbac-practice
# Output: yes ✓
```

Test if developer can read secrets:
```bash
kubectl auth can-i get secrets --as=system:serviceaccount:rbac-practice:developer-sa -n rbac-practice
# Output: no ✗ (Secrets not allowed!)
```

Test if developer can create secrets:
```bash
kubectl auth can-i create secrets --as=system:serviceaccount:rbac-practice:developer-sa -n rbac-practice
# Output: no ✗ (Secrets not allowed!)
```

**Congratulations!** You've restricted access to sensitive resources. ✓

---

## Hands-On Exercise 7: Complete Real-World Scenario (Multi-Team Setup)

### Objective
Set up RBAC for a complete microservices scenario with frontend and backend teams, each with their own namespaces.

### Prerequisites
- Completed Exercises 1-3

### Step-by-Step Instructions

**Step 1: Create namespaces for different teams**

```bash
kubectl create namespace frontend-team
kubectl create namespace backend-team
```

Verify:
```bash
kubectl get namespace frontend-team backend-team
```

**Step 2: Create ServiceAccounts for each team**

Frontend team:
```bash
kubectl create serviceaccount frontend-developer -n frontend-team
```

Backend team:
```bash
kubectl create serviceaccount backend-developer -n backend-team
```

**Step 3: Create Role for frontend team (can manage pods and services)**

Save as `role-frontend.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: frontend-role
  namespace: frontend-team
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "create", "update", "delete"]
```

Apply it:
```bash
kubectl apply -f role-frontend.yaml
```

**Step 4: Create Role for backend team (can manage pods, services, deployments, and secrets)**

Save as `role-backend.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: backend-role
  namespace: backend-team
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list"]
```

Apply it:
```bash
kubectl apply -f role-backend.yaml
```

**Step 5: Create RoleBindings**

Frontend team binding:

Save as `rolebinding-frontend.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: frontend-binding
  namespace: frontend-team
subjects:
- kind: ServiceAccount
  name: frontend-developer
  namespace: frontend-team
roleRef:
  kind: Role
  name: frontend-role
  apiGroup: rbac.authorization.k8s.io
```

Apply it:
```bash
kubectl apply -f rolebinding-frontend.yaml
```

Backend team binding:

Save as `rolebinding-backend.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-binding
  namespace: backend-team
subjects:
- kind: ServiceAccount
  name: backend-developer
  namespace: backend-team
roleRef:
  kind: Role
  name: backend-role
  apiGroup: rbac.authorization.k8s.io
```

Apply it:
```bash
kubectl apply -f rolebinding-backend.yaml
```

**Step 6: Test the scenario**

Frontend team can manage pods in their namespace:
```bash
kubectl auth can-i create pods --as=system:serviceaccount:frontend-team:frontend-developer -n frontend-team
# Output: yes ✓
```

Frontend team cannot manage pods in backend namespace:
```bash
kubectl auth can-i create pods --as=system:serviceaccount:frontend-team:frontend-developer -n backend-team
# Output: no ✗
```

Backend team can manage secrets in their namespace:
```bash
kubectl auth can-i create secrets --as=system:serviceaccount:backend-team:backend-developer -n backend-team
# Output: yes ✓
```

Frontend team cannot create secrets:
```bash
kubectl auth can-i create secrets --as=system:serviceaccount:frontend-team:frontend-developer -n frontend-team
# Output: no ✗
```

**Congratulations!** You've set up a real-world multi-team RBAC scenario. ✓

---

## Hands-On Exercise 8: Monitor and Audit RBAC

### Objective
Learn how to view and audit RBAC configurations in your cluster.

### Prerequisites
- Completed Exercise 7

### Step-by-Step Instructions

**Step 1: List all ServiceAccounts in all namespaces**

```bash
kubectl get serviceaccount -A
```

**Step 2: List all Roles in a specific namespace**

```bash
kubectl get roles -n rbac-practice
kubectl get roles -n frontend-team
```

**Step 3: List all RoleBindings**

```bash
kubectl get rolebinding -n rbac-practice
kubectl get rolebinding -A
```

**Step 4: Describe RBAC objects to see details**

Describe a ServiceAccount:
```bash
kubectl describe serviceaccount pod-reader-sa -n rbac-practice
```

Describe a Role:
```bash
kubectl describe role pod-reader-role -n rbac-practice
```

Describe a RoleBinding:
```bash
kubectl describe rolebinding pod-reader-binding -n rbac-practice
```

**Step 5: View ClusterRoles and ClusterRoleBindings**

List all ClusterRoles:
```bash
kubectl get clusterroles
```

List all ClusterRoleBindings:
```bash
kubectl get clusterrolebindings
```

Check how many clusterroles Kubernetes has by default:
```bash
kubectl get clusterroles | wc -l
```

**Step 6: Export RBAC configuration for backup**

Export a Role:
```bash
kubectl get role pod-reader-role -n rbac-practice -o yaml > backup-role.yaml
```

Export a RoleBinding:
```bash
kubectl get rolebinding pod-reader-binding -n rbac-practice -o yaml > backup-rolebinding.yaml
```

Export a ClusterRole:
```bash
kubectl get clusterrole cluster-pod-reader -o yaml > backup-clusterrole.yaml
```

**Step 7: Create a summary of all RBAC in your namespace**

```bash
# View all RBAC objects in rbac-practice
kubectl get serviceaccount,role,rolebinding -n rbac-practice

# View with more details
kubectl get serviceaccount,role,rolebinding -n rbac-practice -o wide
```

**Step 8: View specific bindings for a ServiceAccount**

Find all RoleBindings for a specific ServiceAccount:
```bash
kubectl get rolebinding -A | grep pod-reader-sa
```

**Congratulations!** You've learned to monitor and audit RBAC. ✓

---

## Complete RBAC Comparison Table

| Aspect | ServiceAccount + Role + RoleBinding | User + ClusterRole + ClusterRoleBinding |
|--------|-------------------------------------|----------------------------------------|
| **Who** | Apps/Pods running in cluster | People accessing cluster (developers, admins) |
| **What** | ServiceAccount | User/Group (person's identity) |
| **Rules** | Role (namespace scope) | ClusterRole (cluster-wide scope) |
| **Binding** | RoleBinding (connects SA to Role in namespace) | ClusterRoleBinding (connects User to ClusterRole across cluster) |
| **Scope** | Limited to single namespace | Entire cluster, all namespaces |
| **Use Case** | My app needs to read pods in my namespace | Admin needs to manage everything in cluster |
| **Example** | Deployment's pod reads other pods | DevOps engineer managing infrastructure |

---

## Cleanup: Remove All Practice Resources

After completing the exercises, clean up your resources:

```bash
# Delete the entire rbac-practice namespace (deletes all resources inside)
kubectl delete namespace rbac-practice

# Delete frontend-team and backend-team namespaces
kubectl delete namespace frontend-team backend-team

# Delete cluster-wide ClusterRole and ClusterRoleBinding
kubectl delete clusterrole cluster-pod-reader
kubectl delete clusterrolebinding cluster-pod-reader-user-binding
```

Verify everything is deleted:
```bash
kubectl get namespace | grep -E "rbac-practice|frontend-team|backend-team"
# No output = everything deleted ✓
```

---

## Quick Reference: All Hands-On Exercises Summary

| Exercise | What You Learn | Time |
|----------|----------------|------|
| Exercise 1 | ServiceAccount + Role + RoleBinding (namespace scope) | 5-10 min |
| Exercise 2 | Test RBAC with a real pod using API calls | 10-15 min |
| Exercise 3 | Create write permissions for ServiceAccount | 5-10 min |
| Exercise 4 | Create ClusterRole and ClusterRoleBinding (cluster scope) for User | 5-10 min |
| Exercise 5 | Multiple resource permissions in one Role | 5-10 min |
| Exercise 6 | Restrict access to sensitive resources | 5-10 min |
| Exercise 7 | Real-world multi-team scenario | 15-20 min |
| Exercise 8 | Monitor and audit RBAC | 5-10 min |

**Total time: 1-2 hours** for all exercises

---

## Useful kubectl Commands for RBAC

```bash
# ============ ServiceAccount Commands ============
kubectl get serviceaccounts -n default
kubectl describe serviceaccount my-app-sa -n default
kubectl create serviceaccount my-sa -n default
kubectl delete serviceaccount my-sa -n default

# ============ Role Commands (Namespace Scope) ============
kubectl get roles -n default
kubectl describe role pod-reader-role -n default
kubectl create role pod-reader --verb=get,list --resource=pods -n default
kubectl delete role pod-reader-role -n default

# ============ RoleBinding Commands (Namespace Scope) ============
kubectl get rolebindings -n default
kubectl describe rolebinding pod-reader-binding -n default
kubectl create rolebinding read-pods --role=pod-reader --serviceaccount=default:my-app-sa -n default
kubectl delete rolebinding pod-reader-binding -n default

# ============ ClusterRole Commands (Cluster-Wide Scope) ============
kubectl get clusterroles
kubectl describe clusterrole cluster-pod-reader
kubectl create clusterrole global-pod-reader --verb=get,list --resource=pods
kubectl delete clusterrole global-pod-reader

# ============ ClusterRoleBinding Commands (Cluster-Wide Scope) ============
kubectl get clusterrolebindings
kubectl describe clusterrolebinding cluster-pod-reader-binding
kubectl create clusterrolebinding global-read-pods --clusterrole=global-pod-reader --serviceaccount=default:my-app-sa
kubectl delete clusterrolebinding global-read-pods

# ============ Permission Testing Commands ============
# Test ServiceAccount permissions (namespace scope)
kubectl auth can-i get pods --as=system:serviceaccount:default:my-app-sa -n default

# Test User permissions (cluster-wide)
kubectl auth can-i get pods --as=john@example.com

# Check what permissions a ServiceAccount has
kubectl auth can-i --list --as=system:serviceaccount:default:my-app-sa -n default

# ============ Export/Backup Commands ============
kubectl get role pod-reader-role -n default -o yaml > role-backup.yaml
kubectl get rolebinding pod-reader-binding -n default -o yaml > rolebinding-backup.yaml
kubectl get clusterrole cluster-pod-reader -o yaml > clusterrole-backup.yaml

# ============ View All RBAC Commands ============
kubectl get serviceaccount,role,rolebinding -n default
kubectl get clusterroles,clusterrolebindings -A
```

---

## Common Mistakes to Avoid

❌ **Mistake 1:** Using wildcard (`*`) for all permissions
```yaml
# DON'T DO THIS - TOO PERMISSIVE!
verbs: ["*"]
```

✓ **Instead:** Grant only needed permissions
```yaml
verbs: ["get", "list", "watch"]
```

---

❌ **Mistake 2:** Mixing namespace and cluster scope
```yaml
# This won't work - RoleBinding can't use ClusterRole
kind: RoleBinding
roleRef:
  kind: ClusterRole  # ✗ Wrong combination
```

✓ **Instead:** Use matching scopes
```yaml
# Use ClusterRoleBinding for ClusterRole
kind: ClusterRoleBinding
roleRef:
  kind: ClusterRole  # ✓ Correct
```

---

❌ **Mistake 3:** Forgetting the apiGroup
```yaml
# Missing apiGroup
roleRef:
  kind: Role
  name: my-role
  # ✗ Should have apiGroup
```

✓ **Instead:** Always include apiGroup
```yaml
roleRef:
  kind: Role
  name: my-role
  apiGroup: rbac.authorization.k8s.io  # ✓ Correct
```

---

❌ **Mistake 4:** Wrong namespace in RoleBinding
```yaml
# ServiceAccount is in 'default' but RoleBinding is in 'production'
kind: RoleBinding
metadata:
  namespace: production  # Different namespace!
subjects:
- kind: ServiceAccount
  name: my-sa
  namespace: default  # This must match
```

---

❌ **Mistake 5:** Using Role with ClusterRoleBinding
```yaml
# RoleBinding expects Role, ClusterRoleBinding expects ClusterRole
kind: ClusterRoleBinding
roleRef:
  kind: Role  # ✗ Wrong - should be ClusterRole
```

---

## Best Practices for RBAC

1. **Use Least Privilege:** Give only the permissions needed
2. **Use Namespaces:** Isolate teams/projects in different namespaces
3. **Use ServiceAccounts:** Don't share ServiceAccounts across pods
4. **Regular Audit:** Check who has what access regularly
5. **Document Roles:** Keep notes on why roles have specific permissions
6. **Test Permissions:** Verify roles work as expected with `kubectl auth can-i`
7. **Avoid Wildcards:** Don't use `*` unless absolutely necessary
8. **Group Permissions:** Use Roles to group related permissions
9. **Use Descriptive Names:** Role names should clearly show their purpose
10. **Separate Concerns:** Don't mix read and write permissions in one role

---

## Debugging RBAC Issues

### Problem: Pod can't access API

**Check 1:** Verify the pod has the right ServiceAccount
```bash
kubectl get pod my-pod -o yaml | grep serviceAccountName
```

**Check 2:** Verify the ServiceAccount has a RoleBinding
```bash
kubectl get rolebinding -n default | grep my-app-sa
```

**Check 3:** Test if permissions work
```bash
kubectl auth can-i get pods --as=system:serviceaccount:default:my-app-sa
```

**Check 4:** View the Role permissions
```bash
kubectl describe role pod-reader-role -n default
```

### Problem: User can't access cluster

**Check 1:** Verify the User has a ClusterRoleBinding
```bash
kubectl get clusterrolebinding | grep john@example.com
```

**Check 2:** Test user permissions
```bash
kubectl auth can-i get pods --as=john@example.com
```

**Check 3:** View ClusterRole permissions
```bash
kubectl describe clusterrole cluster-pod-reader
```

### Problem: Service account not found error

**Solution:** Make sure the ServiceAccount exists in the right namespace
```bash
kubectl get serviceaccount my-app-sa -n default
```

---

## Key Takeaways

1. **RBAC = Who + Can Do What + On Which Resources**

2. **Two Main RBAC Paths:**
   - ServiceAccount → Role → RoleBinding (namespace scope)
   - User → ClusterRole → ClusterRoleBinding (cluster-wide)

3. **ServiceAccount RBAC:**
   - Used for apps/pods running inside cluster
   - Limited to single namespace with Role/RoleBinding
   - Can be cluster-wide with ClusterRole/ClusterRoleBinding

4. **User RBAC:**
   - Used for people accessing cluster (developers, admins)
   - Always cluster-wide with ClusterRole/ClusterRoleBinding

5. **Always use least privilege** - give minimum permissions needed

6. **Test your RBAC** using `kubectl auth can-i` command

7. **Document everything** - keep notes on why each role exists

8. **Start simple** and add complexity as needed

---

## Next Steps

1. ✓ Complete all 8 hands-on exercises
2. ✓ Set up RBAC for your actual applications
3. ✓ Create roles for different teams in your organization
4. ✓ Monitor and audit RBAC regularly
5. Learn about advanced topics: RBAC with External Identity Providers
6. Learn about Pod Security Policies (PSP) - additional security layer
7. Learn about Network Policies - restrict pod-to-pod communication

---

## Common RBAC Rules Cheat Sheet

### For ServiceAccount: Allow reading pods in namespace
```yaml
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

### For ServiceAccount: Allow managing deployments
```yaml
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
```

### For ServiceAccount: Allow managing services
```yaml
rules:
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "create", "delete"]
```

### For ServiceAccount: Allow managing configmaps
```yaml
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
```

### For ServiceAccount: Allow reading secrets (restricted!)
```yaml
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]  # Never use "create", "update", "delete"
```

### For User: Cluster-wide admin access
```yaml
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]  # Use with caution!
```

### For User: Cluster-wide read-only
```yaml
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

---

**Happy securing your cluster!** 🔒🚀
