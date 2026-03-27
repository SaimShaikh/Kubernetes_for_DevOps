
# 🔐 Kubernetes RBAC — Role & ClusterRole In-Depth Guide

> **Who can do what, where — that's RBAC in one line.**

---

## 📚 Table of Contents

1. [The Scenario — Why Do We Need RBAC?](#1-the-scenario--why-do-we-need-rbac)
2. [What Is Kubernetes RBAC?](#2-what-is-kubernetes-rbac)
3. [The 4 RBAC Building Blocks](#3-the-4-rbac-building-blocks)
4. [Role vs ClusterRole — The Core Difference](#4-role-vs-clusterrole--the-core-difference)
5. [Architecture Diagram](#5-architecture-diagram)
6. [Benefits of RBAC](#6-benefits-of-rbac)
7. [How RBAC Works — Step by Step](#7-how-rbac-works--step-by-step)
8. [Role — Namespace Scoped](#8-role--namespace-scoped)
9. [ClusterRole — Cluster Scoped](#9-clusterrole--cluster-scoped)
10. [RoleBinding — Attach Role to User](#10-rolebinding--attach-role-to-user)
11. [ClusterRoleBinding — Attach ClusterRole to User](#11-clusterrolebinding--attach-clusterrole-to-user)
12. [Subjects — Users, Groups, ServiceAccounts](#12-subjects--users-groups-serviceaccounts)
13. [Practical Real-World Examples](#13-practical-real-world-examples)
14. [ServiceAccount + RBAC (Apps Inside K8s)](#14-serviceaccount--rbac-apps-inside-k8s)
15. [Built-in ClusterRoles You Should Know](#15-built-in-clusterroles-you-should-know)
16. [Common Mistakes & How to Avoid Them](#16-common-mistakes--how-to-avoid-them)
17. [Debugging & Troubleshooting](#17-debugging--troubleshooting)
18. [Quick Reference Cheat Sheet](#18-quick-reference-cheat-sheet)

---

## 1. The Scenario — Why Do We Need RBAC?

### 🏢 Imagine Your Company's Kubernetes Cluster

Your team has grown. Now you have:

| Person / Team      | What They Should Access             |
|--------------------|--------------------------------------|
| `alice` (Dev)      | Only her team's namespace (`dev`)    |
| `bob` (QA)         | Read-only access to `staging`        |
| `carol` (Ops)      | Full access to all namespaces        |
| `dave` (Intern)    | NOTHING — just onboarded             |
| `monitor-app`      | A pod that reads pod status          |
| `ci-pipeline`      | Deploy to `production` namespace only|

### ❌ Without RBAC — The Nightmare

```
Everyone gets admin access "just to make things work"
        ↓
Dave (intern) accidentally deletes the production database
        ↓
💥 Outage. Data loss. Panic.
```

Or the opposite:

```
Everyone is locked out
        ↓
Nothing works
        ↓
😤 Frustrated developers, no productivity
```

### ✅ With RBAC — The Right Way

```
alice  →  can create/update pods in namespace "dev" only
bob    →  can only READ resources in namespace "staging"
carol  →  cluster-admin (full access)
dave   →  no permissions at all (default)
ci-bot →  can deploy to "production" only
```

**RBAC = Right person, right access, right place.**

---

## 2. What Is Kubernetes RBAC?

> **RBAC (Role-Based Access Control)** is Kubernetes' built-in authorization system that controls **who** can perform **what actions** on **which resources**.

It answers three questions for every API request:

```
┌─────────────────────────────────────────────────┐
│  WHO    is making the request?  (Subject)        │
│  WHAT   do they want to do?     (Verb/Action)    │
│  WHERE  do they want to do it?  (Resource/Scope) │
└─────────────────────────────────────────────────┘
```

**Example decision:**

```
Alice wants to DELETE a Pod in namespace "production"
  WHO  = alice
  WHAT = delete
  WHERE = pods in namespace "production"

→ Does alice have a Role/ClusterRole allowing delete on pods?
  YES → ✅ Allowed
  NO  → ❌ 403 Forbidden
```

RBAC is enabled by default in Kubernetes since v1.8.

---

## 3. The 4 RBAC Building Blocks

Everything in RBAC is built from exactly 4 objects:

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│   Role             →  WHAT you can do (namespaced)  │
│   ClusterRole      →  WHAT you can do (cluster-wide)│
│                                                     │
│   RoleBinding      →  WHO gets a Role               │
│   ClusterRoleBinding → WHO gets a ClusterRole       │
│                                                     │
└─────────────────────────────────────────────────────┘
```

Think of it like a **key and a lock**:

- **Role/ClusterRole** = the key (defines permissions)
- **RoleBinding/ClusterRoleBinding** = handing that key to someone

> A Role alone does nothing. A user alone has no access. You need BOTH — a Role AND a Binding.

---

## 4. Role vs ClusterRole — The Core Difference

| Feature               | Role                          | ClusterRole                      |
|-----------------------|-------------------------------|----------------------------------|
| Scope                 | **One namespace only**        | **Entire cluster**               |
| Can access namespaced resources | ✅ Yes               | ✅ Yes                           |
| Can access cluster-scoped resources | ❌ No          | ✅ Yes (nodes, PVs, namespaces)  |
| Bound using           | RoleBinding                   | ClusterRoleBinding or RoleBinding|
| Example use case      | Dev access to their namespace | Ops/admin/monitoring             |

### Visual Scope Comparison

```
CLUSTER
├── namespace: dev
│     ├── pods
│     ├── services          ← Role works HERE (one namespace)
│     └── deployments
│
├── namespace: staging
│     ├── pods
│     └── services
│
├── namespace: production
│     └── pods
│
├── nodes                   ← ClusterRole needed for these
├── persistentvolumes       ← ClusterRole needed for these
└── namespaces              ← ClusterRole needed for these
        ↑
ClusterRole works EVERYWHERE above
```

---

## 5. Architecture Diagram

```
                    ┌─────────────┐
   kubectl/API      │  API Server │
   request ────────►│             │
                    │  AuthN ✓    │  (Who are you? cert/token)
                    │  AuthZ ?    │  (What can you do? → RBAC)
                    └──────┬──────┘
                           │
                    ┌──────▼──────────────────────────────┐
                    │          RBAC Authorizer              │
                    │                                      │
                    │  1. Find Subject (user/SA/group)     │
                    │  2. Find all RoleBindings for Subject│
                    │  3. Look up Role/ClusterRole rules   │
                    │  4. Check if action+resource matches │
                    └──────┬──────────┬───────────────────┘
                           │          │
                    ✅ Allow        ❌ Deny
                           │          │
                    ┌──────▼──┐  ┌───▼──────┐
                    │ etcd /  │  │ 403      │
                    │ kubelet │  │ Forbidden│
                    └─────────┘  └──────────┘


RBAC Objects and Their Relationships:

  ┌──────────────┐       ┌─────────────────┐
  │    Role       │◄──────│   RoleBinding   │
  │  (namespace) │       │  (namespace)    │
  └──────────────┘       └────────┬────────┘
                                  │ binds to
                         ┌────────▼────────┐
                         │    Subject      │
                         │  User / Group / │
                         │  ServiceAccount │
                         └─────────────────┘

  ┌──────────────┐       ┌──────────────────────┐
  │ ClusterRole  │◄──────│ ClusterRoleBinding   │
  │  (cluster)   │       │  (cluster)           │
  └──────────────┘       └────────┬─────────────┘
                                  │ binds to
                         ┌────────▼────────┐
                         │    Subject      │
                         │  User / Group / │
                         │  ServiceAccount │
                         └─────────────────┘

  ┌──────────────┐       ┌─────────────────┐
  │ ClusterRole  │◄──────│  RoleBinding    │  ← ClusterRole bound with
  │  (cluster)   │       │  (namespace)    │    RoleBinding = scoped to
  └──────────────┘       └─────────────────┘    ONE namespace only
```

---

## 6. Benefits of RBAC

| Benefit                       | Description                                                   |
|-------------------------------|---------------------------------------------------------------|
| 🔒 **Least Privilege**         | Users get only what they need — nothing more                  |
| 🏢 **Team Isolation**          | Dev team can't touch production namespace                     |
| 🤖 **App-level Security**      | Pods get only the API access they need (via ServiceAccounts)  |
| 📋 **Audit Trail**             | All API calls are logged — know who did what                  |
| 🔁 **Reusability**             | ClusterRoles can be reused across namespaces via RoleBindings |
| ⚖️ **Compliance**              | Meet SOC2, ISO 27001, HIPAA requirements for access control   |
| 🛡️ **Blast Radius Reduction**  | Compromised credentials = limited damage                      |
| 📦 **Multi-tenant Safety**     | Multiple teams share one cluster safely                       |

---

## 7. How RBAC Works — Step by Step

```
Step 1: Kubernetes receives an API request
        "alice wants to GET pods in namespace dev"

Step 2: Authentication — Who is alice?
        (certificate, token, OIDC, etc.)

Step 3: Authorization — What can alice do?
        RBAC looks for RoleBindings where subject = alice
        in namespace dev

Step 4: Find bound Role/ClusterRole
        Found: RoleBinding "alice-dev-binding"
        → points to Role "dev-reader"

Step 5: Check rules in Role "dev-reader"
        rules:
        - resources: ["pods"]
          verbs: ["get", "list", "watch"]

Step 6: Does "get" on "pods" match? → YES ✅
        Request proceeds

Step 7: If NO match found → 403 Forbidden ❌
```

---

## 8. Role — Namespace Scoped

A **Role** defines what actions are allowed on which resources, but **only within one namespace**.

### Basic Role Syntax

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader          # name of the role
  namespace: dev            # ← works ONLY in this namespace
rules:
- apiGroups: [""]           # "" = core API group (pods, services, etc.)
  resources: ["pods"]       # which resources
  verbs: ["get", "list", "watch"]  # what actions
```

### Understanding apiGroups

```
Core resources (pods, services, configmaps, secrets, nodes):
  apiGroups: [""]

Apps resources (deployments, replicasets, daemonsets):
  apiGroups: ["apps"]

Batch resources (jobs, cronjobs):
  apiGroups: ["batch"]

RBAC resources (roles, rolebindings):
  apiGroups: ["rbac.authorization.k8s.io"]

Networking (ingresses):
  apiGroups: ["networking.k8s.io"]

Custom resources (depends on CRD):
  apiGroups: ["your-custom-group.io"]
```

### Understanding Verbs

| Verb       | HTTP Method | Description                  |
|------------|-------------|------------------------------|
| `get`      | GET         | Read a single resource       |
| `list`     | GET         | List resources               |
| `watch`    | GET         | Watch for changes            |
| `create`   | POST        | Create a resource            |
| `update`   | PUT         | Full update of a resource    |
| `patch`    | PATCH       | Partial update               |
| `delete`   | DELETE      | Delete a resource            |
| `deletecollection` | DELETE | Delete multiple resources |
| `*`        | any         | All verbs (wildcard)         |

### Role Examples

**Read-only role for pods and logs:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
```

**Developer role — deploy apps but not delete:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: dev
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
```

**Secret reader (specific resource names):**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["db-password", "api-key"]  # ← only these specific secrets
  verbs: ["get"]
```

---

## 9. ClusterRole — Cluster Scoped

A **ClusterRole** is like a Role, but works **across all namespaces** and can also access **cluster-level resources** (nodes, persistent volumes, namespaces).

### Basic ClusterRole Syntax

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader       # no namespace field — it's cluster-wide
rules:
- apiGroups: [""]
  resources: ["nodes"]    # cluster-scoped resource
  verbs: ["get", "list", "watch"]
```

### ClusterRole Examples

**Monitoring role — read everything cluster-wide:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-viewer
rules:
- apiGroups: [""]
  resources: ["pods", "nodes", "services", "endpoints",
              "namespaces", "persistentvolumes"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "daemonsets", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "watch"]
```

**CI/CD deploy role — deploy to any namespace:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployer
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["services", "configmaps"]
  verbs: ["get", "list", "create", "update", "patch"]
```

**Wildcard — full access (use carefully):**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: super-admin
rules:
- apiGroups: ["*"]    # all API groups
  resources: ["*"]   # all resources
  verbs: ["*"]       # all actions
```

---

## 10. RoleBinding — Attach Role to User

A **RoleBinding** connects a Role (or ClusterRole) to a Subject (user/group/ServiceAccount) **within a namespace**.

### Basic RoleBinding Syntax

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-pod-reader-binding
  namespace: dev               # ← this binding lives in namespace "dev"
subjects:
- kind: User                   # User, Group, or ServiceAccount
  name: alice                  # the actual name
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role                   # Role or ClusterRole
  name: pod-reader             # must exist in same namespace
  apiGroup: rbac.authorization.k8s.io
```

### RoleBinding — Multiple Subjects

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-binding
  namespace: dev
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
- kind: User
  name: bob
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: dev-team          # ← entire group gets access
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

### RoleBinding using a ClusterRole (Scoped Down!)

This is powerful — reuse a ClusterRole but restrict it to one namespace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: monitoring-dev-only
  namespace: dev              # ← restricts to "dev" namespace only
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole           # ← referencing a ClusterRole...
  name: monitoring-viewer     # ...but binding scopes it to namespace "dev"
  apiGroup: rbac.authorization.k8s.io
```

```
ClusterRole "monitoring-viewer" (cluster-wide definition)
        +
RoleBinding in namespace "dev"
        =
alice can only monitor resources IN namespace "dev"
```

---

## 11. ClusterRoleBinding — Attach ClusterRole to User

A **ClusterRoleBinding** connects a ClusterRole to a Subject with **no namespace restriction** — they get access everywhere.

### Basic ClusterRoleBinding Syntax

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: carol-admin-binding   # no namespace — cluster-wide
subjects:
- kind: User
  name: carol
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin          # built-in ClusterRole
  apiGroup: rbac.authorization.k8s.io
```

### Comparison Table — All 4 Binding Combinations

```
┌──────────────────┬────────────────────┬─────────────────────────────────┐
│ Role Type        │ Binding Type       │ Effect                          │
├──────────────────┼────────────────────┼─────────────────────────────────┤
│ Role             │ RoleBinding        │ Access in ONE namespace          │
│ ClusterRole      │ RoleBinding        │ Access in ONE namespace (scoped) │
│ ClusterRole      │ ClusterRoleBinding │ Access in ALL namespaces         │
│ Role             │ ClusterRoleBinding │ ❌ NOT VALID — won't work        │
└──────────────────┴────────────────────┴─────────────────────────────────┘
```

---

## 12. Subjects — Users, Groups, ServiceAccounts

### 12.1 User

Individual humans authenticating to the cluster.

```yaml
subjects:
- kind: User
  name: alice               # must match the CN in their certificate
  apiGroup: rbac.authorization.k8s.io
```

> ⚠️ Kubernetes does NOT manage users natively. Users come from external systems: certs, OIDC (Google/GitHub), AWS IAM, etc.

### 12.2 Group

A collection of users. Useful for team-level access.

```yaml
subjects:
- kind: Group
  name: dev-team            # group name from cert's "O" field or OIDC
  apiGroup: rbac.authorization.k8s.io
```

**Special built-in groups:**

| Group                              | Who it includes              |
|------------------------------------|------------------------------|
| `system:authenticated`             | All logged-in users          |
| `system:unauthenticated`           | Anonymous users              |
| `system:masters`                   | Cluster admins               |
| `system:serviceaccounts`           | All service accounts         |
| `system:serviceaccounts:<ns>`      | All SAs in a namespace       |

### 12.3 ServiceAccount

An identity for **pods and applications** running inside Kubernetes.

```yaml
subjects:
- kind: ServiceAccount
  name: my-app-sa
  namespace: production     # ← namespace is required for ServiceAccounts
```

---

## 13. Practical Real-World Examples

### Example 1: Developer Access to Their Namespace Only

```yaml
# Step 1: Create the Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: dev
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["pods", "deployments", "services",
              "configmaps", "jobs", "cronjobs"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["pods/log", "pods/exec"]     # exec into pods + view logs
  verbs: ["get", "create"]
---
# Step 2: Bind alice to it
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-developer-binding
  namespace: dev
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

---

### Example 2: Read-Only QA Access to Staging

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: qa-readonly
  namespace: staging
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]   # read only — no create/update/delete
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bob-qa-binding
  namespace: staging
subjects:
- kind: User
  name: bob
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: qa-readonly
  apiGroup: rbac.authorization.k8s.io
```

---

### Example 3: Ops Team — Full Cluster Admin

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ops-team-admin
subjects:
- kind: Group
  name: ops-team            # everyone in ops-team group
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin       # built-in full-access role
  apiGroup: rbac.authorization.k8s.io
```

---

### Example 4: Namespace Admin (Can Manage Everything in One Namespace)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: carol-ns-admin
  namespace: production
subjects:
- kind: User
  name: carol
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin               # built-in namespace admin ClusterRole
  apiGroup: rbac.authorization.k8s.io
```

---

## 14. ServiceAccount + RBAC (Apps Inside K8s)

When a **pod needs to call the Kubernetes API** (e.g., monitoring tools, CI bots, operators), you use a **ServiceAccount** with RBAC.

### Step 1: Create a ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitor-sa
  namespace: monitoring
```

### Step 2: Create a ClusterRole with needed permissions

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-watcher
rules:
- apiGroups: [""]
  resources: ["pods", "nodes"]
  verbs: ["get", "list", "watch"]
```

### Step 3: Bind ServiceAccount to ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitor-sa-binding
subjects:
- kind: ServiceAccount
  name: monitor-sa
  namespace: monitoring        # ← required for ServiceAccounts
roleRef:
  kind: ClusterRole
  name: pod-watcher
  apiGroup: rbac.authorization.k8s.io
```

### Step 4: Use the ServiceAccount in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: monitor-pod
  namespace: monitoring
spec:
  serviceAccountName: monitor-sa    # ← attach the SA here
  containers:
  - name: monitor
    image: my-monitor-image
```

### Flow Inside the Pod

```
Pod starts
  ↓
Kubernetes auto-mounts SA token at:
  /var/run/secrets/kubernetes.io/serviceaccount/token
  ↓
App uses token to authenticate to API server
  ↓
RBAC checks: does monitor-sa have permission?
  ↓
ClusterRoleBinding found → ClusterRole "pod-watcher" → ✅ allowed
```

---

## 15. Built-in ClusterRoles You Should Know

Kubernetes ships with pre-built ClusterRoles. Use them instead of reinventing the wheel.

| ClusterRole        | Access Level         | Use Case                              |
|--------------------|----------------------|---------------------------------------|
| `cluster-admin`    | Full cluster access  | Cluster operators only ⚠️             |
| `admin`            | Namespace admin      | Namespace owners (no quota/policy changes) |
| `edit`             | Read + write         | Developers — can't manage RBAC itself |
| `view`             | Read-only            | QA, auditors, monitoring              |

### Check what a built-in role can do

```bash
kubectl describe clusterrole view
kubectl describe clusterrole edit
kubectl describe clusterrole admin
kubectl describe clusterrole cluster-admin
```

### Use built-in roles via RoleBinding (namespace-scoped)

```yaml
# Give alice "edit" access in namespace "dev" using built-in ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-edit
  namespace: dev
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit                  # built-in
  apiGroup: rbac.authorization.k8s.io
```

---

## 16. Common Mistakes & How to Avoid Them

### ❌ Mistake 1: Giving Everyone cluster-admin

```yaml
# DON'T DO THIS
roleRef:
  kind: ClusterRole
  name: cluster-admin   # ← nuclear option
```

**✅ Fix:** Use `edit` or custom Role with least privilege.

---

### ❌ Mistake 2: Forgetting namespace in ServiceAccount subject

```yaml
# WRONG — missing namespace
subjects:
- kind: ServiceAccount
  name: my-sa
  # apiGroup should NOT be here for SA, namespace is REQUIRED

# CORRECT
subjects:
- kind: ServiceAccount
  name: my-sa
  namespace: my-namespace   # ← always required for ServiceAccount
```

---

### ❌ Mistake 3: Using ClusterRoleBinding when RoleBinding is enough

```yaml
# WRONG — gives access cluster-wide when you only need one namespace
kind: ClusterRoleBinding   # ← overkill

# CORRECT — scope it down
kind: RoleBinding
metadata:
  namespace: dev
```

---

### ❌ Mistake 4: Wrong apiGroup for resources

```yaml
# WRONG — deployments are NOT in core API group
- apiGroups: [""]
  resources: ["deployments"]

# CORRECT
- apiGroups: ["apps"]
  resources: ["deployments"]
```

---

### ❌ Mistake 5: Binding a Role with ClusterRoleBinding

```yaml
# DOES NOT WORK
roleRef:
  kind: Role              # ← Role can't be used in ClusterRoleBinding
  name: my-role
# ClusterRoleBinding only works with ClusterRole
```

---

## 17. Debugging & Troubleshooting

### Check if a User Can Do Something

```bash
# Can alice get pods in namespace dev?
kubectl auth can-i get pods --namespace dev --as alice

# Can alice delete deployments?
kubectl auth can-i delete deployments --namespace dev --as alice

# Can a ServiceAccount list nodes?
kubectl auth can-i list nodes \
  --as system:serviceaccount:monitoring:monitor-sa
```

### List All Roles in a Namespace

```bash
kubectl get roles -n dev
kubectl get rolebindings -n dev
kubectl get clusterroles
kubectl get clusterrolebindings
```

### Describe a Role to See Its Rules

```bash
kubectl describe role developer-role -n dev
kubectl describe clusterrole monitoring-viewer
```

### Find What a User Is Bound To

```bash
# Find all RoleBindings for alice
kubectl get rolebindings -n dev -o wide | grep alice

# Find all ClusterRoleBindings for alice
kubectl get clusterrolebindings -o wide | grep alice
```

### Who Has Access to a Namespace? (audit)

```bash
kubectl get rolebindings -n production -o json | \
  jq '.items[] | {name: .metadata.name, subjects: .subjects, role: .roleRef.name}'
```

### Common Error Messages

| Error | Cause | Fix |
|-------|-------|-----|
| `403 Forbidden` | No matching RBAC rule | Create Role + Binding |
| `User "alice" cannot get resource "pods"` | Missing verb | Add `get` to role verbs |
| `no kind "Role" is registered` | Wrong apiVersion | Use `rbac.authorization.k8s.io/v1` |
| `RoleBinding references non-existing role` | Role doesn't exist yet | Create Role before RoleBinding |
| `cannot escalate privileges` | Trying to grant more than you have | Use cluster-admin to create high-privilege roles |

---

## 18. Quick Reference Cheat Sheet

```bash
# ── CREATE ────────────────────────────────────────────────
kubectl apply -f role.yaml
kubectl apply -f clusterrole.yaml
kubectl apply -f rolebinding.yaml
kubectl apply -f clusterrolebinding.yaml

# ── VIEW ──────────────────────────────────────────────────
kubectl get roles -n <namespace>
kubectl get clusterroles
kubectl get rolebindings -n <namespace>
kubectl get clusterrolebindings

kubectl describe role <name> -n <namespace>
kubectl describe clusterrole <name>
kubectl describe rolebinding <name> -n <namespace>

# ── TEST PERMISSIONS ──────────────────────────────────────
kubectl auth can-i <verb> <resource> --as <user>
kubectl auth can-i <verb> <resource> --as <user> -n <namespace>
kubectl auth can-i <verb> <resource> \
  --as system:serviceaccount:<namespace>:<sa-name>

# ── LIST ALL PERMISSIONS FOR A SUBJECT ────────────────────
kubectl auth can-i --list --as <user> -n <namespace>

# ── QUICK CREATE (imperative) ─────────────────────────────
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n dev

kubectl create rolebinding alice-pod-reader \
  --role=pod-reader \
  --user=alice \
  -n dev

kubectl create clusterrole node-reader \
  --verb=get,list,watch \
  --resource=nodes

kubectl create clusterrolebinding carol-admin \
  --clusterrole=cluster-admin \
  --user=carol

# ── DELETE ────────────────────────────────────────────────
kubectl delete role <name> -n <namespace>
kubectl delete clusterrole <name>
kubectl delete rolebinding <name> -n <namespace>
kubectl delete clusterrolebinding <name>
```

---

## Summary — The Mental Model

```
STEP 1: Define WHAT is allowed
        Role (namespace) or ClusterRole (cluster)
        → apiGroups + resources + verbs

STEP 2: Define WHO gets it
        Subject: User | Group | ServiceAccount

STEP 3: Connect them
        RoleBinding (namespace-scoped)
        ClusterRoleBinding (cluster-wide)

STEP 4: Kubernetes checks on every API call:
        Does a Binding exist for this Subject?
        Does the bound Role allow this verb on this resource?
        YES → ✅  |  NO → ❌ 403
```

```
The Golden Rule of RBAC:
┌─────────────────────────────────────────────┐
│  Grant the MINIMUM permissions needed.      │
│  Start with "view", escalate only if needed.│
│  Never use cluster-admin for apps/pipelines.│
└─────────────────────────────────────────────┘
```

---

*Practice tip: Spin up a local cluster with `minikube` or `kind`, create users with certificates, and test `kubectl auth can-i` to see RBAC in action!*
