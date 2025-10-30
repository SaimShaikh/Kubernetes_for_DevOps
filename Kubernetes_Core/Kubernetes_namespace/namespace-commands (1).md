# Kubernetes Namespace Commands - Complete Guide

## What Are Namespaces (Quick Recap)

**Namespaces are like folders or departments in your Kubernetes cluster.** They divide your cluster into separate spaces so different teams, projects, or environments don't interfere with each other.

```
Kubernetes Cluster
├── default (comes by default)
├── kube-system (Kubernetes system stuff)
├── kube-public (public information)
├── kube-node-lease (node heartbeat)
├── my-app (your app here)
├── production (production stuff)
├── testing (testing stuff)
└── staging (staging stuff)
```

---

## Part 1: View and List Namespaces

### 1. Get All Namespaces
```bash
kubectl get namespaces
```
**Purpose:** See all namespaces in your cluster
**Output:**
```
NAME              STATUS   AGE
default           Active   30d
kube-node-lease   Active   30d
kube-public       Active   30d
kube-system       Active   30d
my-app            Active   25d
```

### 2. Get Namespaces (Short Form)
```bash
kubectl get ns
```
**Purpose:** Same as above but shorter command
**Tip:** `ns` is short for `namespaces`

### 3. Describe a Specific Namespace
```bash
kubectl describe namespace my-app
```
**Purpose:** Get detailed information about a specific namespace
**Output shows:**
- Name
- Labels
- Annotations
- Resource Quotas
- Network Policies
- Status

### 4. Get Namespace in YAML Format
```bash
kubectl get namespace my-app -o yaml
```
**Purpose:** See the namespace configuration in YAML format
**Useful for:** Understanding the complete setup

### 5. Get All Namespaces with Details
```bash
kubectl get namespaces --show-labels
```
**Purpose:** List namespaces and show their labels
**Output:**
```
NAME              STATUS   AGE     LABELS
default           Active   30d     <none>
my-app            Active   25d     app=myapp,env=prod
production        Active   10d     env=production
testing           Active   15d     env=testing
```

---

## Part 2: Create Namespaces

### 1. Create Namespace with Command
```bash
kubectl create namespace my-app
```
**Purpose:** Create a new namespace called "my-app"
**Output:** `namespace/my-app created`

### 2. Create Namespace with Labels
```bash
kubectl create namespace production --labels=env=production,team=platform
```
**Purpose:** Create namespace with labels attached
**Use when:** You want to organize namespaces by environment or team

### 3. Create Namespace from YAML File
```bash
kubectl apply -f namespace.yaml
```
**Purpose:** Create namespace from a YAML file
**Example YAML file:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  labels:
    env: production
    team: backend
```

### 4. Create Namespace and Set Resource Quota
```bash
kubectl create namespace my-app
kubectl apply -f resource-quota.yaml
```
**Purpose:** Create namespace with resource limits
**Example YAML:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: my-app
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
```

---

## Part 3: Manage Namespaces

### 1. Switch to a Different Namespace
```bash
kubectl config set-context --current --namespace=my-app
```
**Purpose:** Set default namespace so you don't have to type `-n my-app` every time
**After this:** All commands run in `my-app` namespace by default

### 2. Check Current Namespace
```bash
kubectl config view --minify | grep namespace
```
**Purpose:** See which namespace you're currently in
**Output:** `namespace: my-app`

### 3. Delete a Namespace
```bash
kubectl delete namespace my-app
```
**Purpose:** Delete a namespace (also deletes all Pods/Services inside it)
**Warning:** This is destructive! All resources inside get deleted too

### 4. Delete Namespace with Confirmation
```bash
kubectl delete namespace my-app --ignore-not-found
```
**Purpose:** Delete namespace but don't error if it doesn't exist
**Use when:** Running scripts that might not need the namespace

### 5. Force Delete a Namespace
```bash
kubectl delete namespace my-app --grace-period=0 --force
```
**Purpose:** Immediately delete namespace without waiting
**Warning:** Use only when namespace is stuck

---

## Part 4: Get Resources in Namespaces

### 1. Get Pods in a Specific Namespace
```bash
kubectl get pods -n my-app
```
**Purpose:** See all Pods in the "my-app" namespace
**Output:**
```
NAME                    READY   STATUS    RESTARTS   AGE
nginx-1234              1/1     Running   0          5d
api-5678                1/1     Running   0          3d
```

### 2. Get All Resources in a Namespace
```bash
kubectl get all -n my-app
```
**Purpose:** See Pods, Services, Deployments, ReplicaSets, etc. in one command
**Useful for:** Quick overview of what's running

### 3. Get Pods Across All Namespaces
```bash
kubectl get pods --all-namespaces
```
**Purpose:** See all Pods in entire cluster
**Output shows:** Pod name and their namespace

### 4. Get Pods Across All Namespaces (Short Form)
```bash
kubectl get pods -A
```
**Purpose:** Same as above but shorter
**Tip:** `-A` is short for `--all-namespaces`

### 5. Get Specific Resource Type Across All Namespaces
```bash
kubectl get services -A
```
**Purpose:** See all Services in all namespaces
**Also works with:** `deployments`, `configmaps`, `secrets`, etc.

### 6. Get Resources with Namespace Column
```bash
kubectl get pods -A -o wide
```
**Purpose:** See Pods with extra details like Node they're running on
**Columns shown:** NAME, READY, STATUS, RESTARTS, AGE, IP, NODE, NAMESPACE

---

## Part 5: Describe Resources in Namespaces

### 1. Describe Pod in Specific Namespace
```bash
kubectl describe pod nginx-1234 -n my-app
```
**Purpose:** Get detailed information about a specific Pod in a namespace
**Shows:** IP, Node, Labels, Events, Containers, etc.

### 2. Describe All Pods in Namespace
```bash
kubectl describe pods -n my-app
```
**Purpose:** Get details of all Pods in the namespace
**Useful for:** Debugging issues in that namespace

### 3. Describe Service in Specific Namespace
```bash
kubectl describe service my-service -n my-app
```
**Purpose:** Get details about a Service in a namespace
**Shows:** IP, Port, Endpoints, Selectors, etc.

### 4. Describe Namespace Itself
```bash
kubectl describe namespace my-app
```
**Purpose:** Get full details about the namespace
**Shows:** Labels, Quotas, Status, Events

---

## Part 6: Edit Namespaces and Resources

### 1. Edit a Namespace
```bash
kubectl edit namespace my-app
```
**Purpose:** Open namespace YAML in your default editor for editing
**Use when:** You want to add/modify labels or annotations

### 2. Label a Namespace
```bash
kubectl label namespace my-app env=production
```
**Purpose:** Add a label to an existing namespace
**Useful for:** Organizing namespaces

### 3. Unlabel a Namespace
```bash
kubectl label namespace my-app env-
```
**Purpose:** Remove a label from a namespace
**Note:** The `-` at the end removes the label

### 4. Overwrite Namespace Label
```bash
kubectl label namespace my-app env=testing --overwrite
```
**Purpose:** Change an existing label value
**Use when:** You need to update the namespace label

---

## Part 7: Delete Resources in Namespaces

### 1. Delete All Pods in a Namespace
```bash
kubectl delete pods -n my-app
```
**Purpose:** Delete every Pod in the namespace (useful for cleanup)
**Warning:** All Pods will be deleted

### 2. Delete All Pods with Specific Label
```bash
kubectl delete pods -n my-app -l app=web
```
**Purpose:** Delete only Pods matching the label
**Useful for:** Selective cleanup

### 3. Delete Specific Pod in Namespace
```bash
kubectl delete pod nginx-1234 -n my-app
```
**Purpose:** Delete a specific Pod from a namespace
**Output:** `pod "nginx-1234" deleted`

### 4. Delete All Resources in Namespace
```bash
kubectl delete all -n my-app
```
**Purpose:** Delete Pods, Services, Deployments, etc. but keep the namespace
**Useful for:** Cleaning up everything except the namespace itself

### 5. Delete Resource with Confirmation
```bash
kubectl delete pod nginx-1234 -n my-app --ignore-not-found
```
**Purpose:** Delete Pod but don't error if it doesn't exist
**Use when:** Running cleanup scripts

---

## Part 8: Run Commands in Namespaces

### 1. Run a Command Inside a Pod (Specific Namespace)
```bash
kubectl exec -it nginx-1234 -n my-app -- /bin/bash
```
**Purpose:** Open a shell inside a Pod in a specific namespace
**After running:** You're inside the container, can run commands

### 2. Run Command Inside Pod (Get Logs)
```bash
kubectl logs nginx-1234 -n my-app
```
**Purpose:** See what the Pod is printing (logs)
**Useful for:** Debugging problems

### 3. Run Command Inside Pod (Follow Logs)
```bash
kubectl logs -f nginx-1234 -n my-app
```
**Purpose:** See logs in real-time (live stream)
**Stop with:** Press `Ctrl+C`

### 4. Copy Files to/from Pod
```bash
kubectl cp my-app/nginx-1234:/etc/nginx/nginx.conf ./nginx.conf
```
**Purpose:** Copy a file from Pod to your computer
**Format:** `kubectl cp namespace/pod:/path/to/file ./local/path`

### 5. Port Forward to Pod
```bash
kubectl port-forward nginx-1234 8080:80 -n my-app
```
**Purpose:** Forward local port 8080 to Pod's port 80
**Use when:** You want to access Pod from your computer

---

## Part 9: Apply Configuration to Namespaces

### 1. Apply YAML to Specific Namespace
```bash
kubectl apply -f deployment.yaml -n my-app
```
**Purpose:** Deploy a resource to a specific namespace
**Instead of:** Changing the namespace in YAML file

### 2. Apply YAML and Create Namespace if Doesn't Exist
```bash
kubectl apply -f deployment.yaml --namespace=my-app
```
**Purpose:** Apply resource, namespace flag also works
**Both forms:** `-n` and `--namespace` do the same thing

### 3. Apply Multiple Files to a Namespace
```bash
kubectl apply -f ./manifests/ -n my-app
```
**Purpose:** Apply all YAML files in a folder to a namespace
**Useful for:** Deploying multiple resources at once

### 4. Apply with Dry Run (Preview)
```bash
kubectl apply -f deployment.yaml -n my-app --dry-run=client -o yaml
```
**Purpose:** See what will be created without actually creating it
**Use when:** You want to preview before deploying

---

## Part 10: Set Default Namespace

### 1. Set Current Context's Default Namespace
```bash
kubectl config set-context --current --namespace=my-app
```
**Purpose:** Make `my-app` the default namespace
**After this:** `kubectl get pods` runs in `my-app` automatically

### 2. Create New Context with Default Namespace
```bash
kubectl config set-context my-app-ctx --cluster=minikube --user=minikube --namespace=my-app
```
**Purpose:** Create a new context with a specific namespace
**Then use:** `kubectl config use-context my-app-ctx`

### 3. Switch Context
```bash
kubectl config use-context my-app-ctx
```
**Purpose:** Switch to the context you just created
**After this:** All commands use that context's settings

### 4. View Current Context
```bash
kubectl config current-context
```
**Purpose:** See which context you're currently using
**Output:** `minikube` or `my-app-ctx`

---

## Part 11: Watch Namespace Changes

### 1. Watch Pods in Real-Time
```bash
kubectl get pods -n my-app -w
```
**Purpose:** Live monitor all Pods in the namespace
**Shows:** Changes as they happen
**Stop with:** Press `Ctrl+C`

### 2. Watch Deployments in Real-Time
```bash
kubectl get deployments -n my-app -w
```
**Purpose:** Watch Deployment status changes
**Useful for:** Monitoring rollouts

### 3. Watch Events in Namespace
```bash
kubectl get events -n my-app
```
**Purpose:** See what happened in the namespace (events log)
**Shows:** Errors, warnings, normal events

### 4. Watch Events in Real-Time
```bash
kubectl get events -n my-app -w
```
**Purpose:** Live stream of namespace events
**Useful for:** Debugging in real-time

---

## Part 12: Export and Backup Namespaces

### 1. Export All Resources from Namespace
```bash
kubectl get all -n my-app -o yaml > backup.yaml
```
**Purpose:** Save all resources from namespace to a file
**Use when:** You want to backup everything

### 2. Export Specific Resource Type
```bash
kubectl get pods -n my-app -o yaml > pods-backup.yaml
```
**Purpose:** Backup only Pods from a namespace
**Also works with:** deployments, services, etc.

### 3. Export with Custom Format
```bash
kubectl get pods -n my-app -o json > pods-backup.json
```
**Purpose:** Export in JSON format instead of YAML

### 4. Export and Pretty Print
```bash
kubectl get pods -n my-app -o yaml | head -50
```
**Purpose:** See first 50 lines of the export
**Useful for:** Quick preview before saving

---

## Part 13: Common Namespace Workflows

### Workflow 1: Create New Project Namespace
```bash
# Step 1: Create namespace
kubectl create namespace project-alpha

# Step 2: Add labels
kubectl label namespace project-alpha team=alpha env=dev

# Step 3: Set it as default
kubectl config set-context --current --namespace=project-alpha

# Step 4: Deploy your app
kubectl apply -f deployment.yaml

# Step 5: Check status
kubectl get all -n project-alpha
```

### Workflow 2: Backup and Restore Namespace
```bash
# Backup
kubectl get all -n my-app -o yaml > my-app-backup.yaml

# Delete (if needed)
kubectl delete namespace my-app

# Restore
kubectl apply -f my-app-backup.yaml
```

### Workflow 3: Copy Namespace to Another Cluster
```bash
# Export from source cluster
kubectl get all -n my-app -o yaml > export.yaml

# Import to target cluster
kubectl config use-context target-cluster
kubectl create namespace my-app
kubectl apply -f export.yaml -n my-app
```

### Workflow 4: Monitor Multiple Namespaces
```bash
# Watch Pods in all namespaces
kubectl get pods -A -w

# Watch Pods in specific namespaces only
kubectl get pods -n my-app -n production -w
```

---

## Part 14: Namespace Best Practices

### ✅ DO:
- Create separate namespaces for different teams
- Use labels to organize namespaces
- Set resource quotas on production namespaces
- Use specific namespace in commands (`-n namespace-name`)
- Backup important namespaces regularly

### ❌ DON'T:
- Use `default` namespace for production
- Share namespaces between unrelated projects
- Forget to specify namespace in scripts
- Delete namespaces without backup
- Mix environment types in same namespace

---

## Part 15: Namespace Commands Quick Reference

| Command | Purpose |
|---------|---------|
| `kubectl get ns` | List all namespaces |
| `kubectl create namespace my-app` | Create new namespace |
| `kubectl delete namespace my-app` | Delete namespace |
| `kubectl config set-context --current --namespace=my-app` | Set default namespace |
| `kubectl get pods -n my-app` | Get Pods in specific namespace |
| `kubectl get pods -A` | Get Pods from all namespaces |
| `kubectl describe namespace my-app` | Get namespace details |
| `kubectl label namespace my-app env=prod` | Add label to namespace |
| `kubectl edit namespace my-app` | Edit namespace |
| `kubectl delete all -n my-app` | Delete all resources in namespace |
| `kubectl get events -n my-app` | See namespace events |
| `kubectl apply -f file.yaml -n my-app` | Apply resource to namespace |

---

## Part 16: Complete Command Index (Alphabetical)

### A - Apply
- `kubectl apply -f file.yaml -n my-app` - Apply resource to namespace
- `kubectl apply -f file.yaml --namespace=my-app` - Alternative syntax

### C - Create / Config
- `kubectl create namespace my-app` - Create namespace
- `kubectl config set-context --current --namespace=my-app` - Set default namespace
- `kubectl config use-context my-app-ctx` - Switch context
- `kubectl config current-context` - View current context
- `kubectl copy file pod:/path` - Copy file to pod

### D - Delete / Describe
- `kubectl delete namespace my-app` - Delete namespace
- `kubectl delete pods -n my-app` - Delete all Pods in namespace
- `kubectl delete pod pod-name -n my-app` - Delete specific Pod
- `kubectl describe namespace my-app` - Describe namespace
- `kubectl describe pods -n my-app` - Describe all Pods

### E - Edit / Exec
- `kubectl edit namespace my-app` - Edit namespace
- `kubectl exec -it pod-name -n my-app -- /bin/bash` - Shell into Pod

### G - Get
- `kubectl get ns` - List all namespaces
- `kubectl get pods -n my-app` - Get Pods in namespace
- `kubectl get pods -A` - Get Pods from all namespaces
- `kubectl get all -n my-app` - Get all resources in namespace
- `kubectl get events -n my-app` - Get events in namespace

### L - Label / Logs
- `kubectl label namespace my-app env=prod` - Add label
- `kubectl logs pod-name -n my-app` - Get Pod logs
- `kubectl logs -f pod-name -n my-app` - Follow Pod logs

### P - Port-forward
- `kubectl port-forward pod-name 8080:80 -n my-app` - Forward port

### W - Watch
- `kubectl get pods -n my-app -w` - Watch Pods in real-time
- `kubectl get events -n my-app -w` - Watch events in real-time

---

## TL;DR (Boss Summary)

- **`kubectl get ns`** → See all namespaces
- **`kubectl create namespace my-app`** → Create namespace
- **`kubectl get pods -n my-app`** → Get Pods in specific namespace
- **`kubectl get pods -A`** → Get Pods from all namespaces
- **`kubectl config set-context --current --namespace=my-app`** → Set default namespace
- **`kubectl delete namespace my-app`** → Delete namespace
- **`kubectl apply -f file.yaml -n my-app`** → Deploy to namespace
- **`kubectl delete all -n my-app`** → Clean up namespace

**Remember:** Always specify namespace with `-n` flag unless you've set it as default!

---

## Resources to Learn More

- Kubernetes Namespaces Official Docs: https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/
- Kubectl Reference: https://kubernetes.io/docs/reference/kubectl/
- Practice: Try creating namespaces and deploying apps to different ones
