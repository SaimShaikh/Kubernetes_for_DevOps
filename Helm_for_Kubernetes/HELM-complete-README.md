# Kubernetes Helm - Complete Beginner's Guide

## What is Helm?

**Helm** = Package Manager for Kubernetes

Think of Helm like this:
- **Package Manager** (like npm for Node.js, pip for Python)
- **Kubernetes** uses YAML files to deploy apps
- **Helm** makes it easy to manage multiple YAML files
- **Instead of managing 10 YAML files**, Helm packages them together

**In simple terms:** Helm = Make Kubernetes deployment easy! ✔️

---

## Real-World Analogy

### Without Helm (Doing it manually):
```
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f ingress.yaml
kubectl apply -f pvc.yaml
... and many more files
```

### With Helm (One command):
```
helm install my-app ./my-chart
# Done! Everything deployed!
```

---

## Why Use Helm?

| Problem | Solution |
|---------|----------|
| Too many YAML files | Package them together |
| Hard to manage versions | Helm handles versioning |
| Manual deployments | Automated deployment |
| Difficult to update | Easy helm upgrade |
| Difficult to delete | helm uninstall removes everything |
| Hard to reuse configs | Helm templates |
| Difficult for teams | Standardized packages |
| No rollback | helm rollback goes back |

---

## Benefits of Helm

✅ **Easy Installation** - One command to deploy entire apps  
✅ **Version Control** - Install specific versions  
✅ **Updates** - Simple helm upgrade  
✅ **Rollback** - Go back to previous version  
✅ **Reusable** - Use same chart for different environments  
✅ **Templates** - Dynamic configuration  
✅ **Standardization** - Teams use same format  
✅ **Community Charts** - 1000+ pre-built charts  
✅ **Package Management** - Dependency management  
✅ **Easy Cleanup** - helm uninstall removes everything  

---

## Key Concepts

### 1. Chart
A **Chart** is a package that contains all Kubernetes manifests.

Think: A folder with all YAML files organized

### 2. Release
A **Release** is when you install a Chart.

Think: An instance of the chart running in your cluster

### 3. Repository (Repo)
A **Repository** is where Charts are stored.

Think: Like GitHub for Helm Charts

### 4. Values
**Values** are variables that customize the chart.

Think: Configuration file for the chart

---

## Helm vs kubectl

| Aspect | kubectl | Helm |
|--------|---------|------|
| **What** | Deploy one resource | Deploy entire app |
| **Command** | `kubectl apply -f file.yaml` | `helm install name chart` |
| **Files** | One file | Multiple files in chart |
| **Versioning** | Manual | Automatic |
| **Upgrade** | Replace file, reapply | `helm upgrade` |
| **Rollback** | Manual | `helm rollback` |
| **Deletion** | Delete each resource | `helm uninstall` |
| **Reusability** | Low | High |

---

# 📦 How Helm Works

## Step 1: Find a Chart

```bash
helm repo add myrepo https://charts.example.com
helm search repo wordpress
```

## Step 2: Install the Chart

```bash
helm install my-wordpress myrepo/wordpress
```

## Step 3: Release is Running

```bash
helm list
# Shows: my-wordpress is DEPLOYED
```

## Step 4: Update if Needed

```bash
helm upgrade my-wordpress myrepo/wordpress
```

## Step 5: Rollback if Something Wrong

```bash
helm rollback my-wordpress
```

## Step 6: Delete Everything

```bash
helm uninstall my-wordpress
```

---

# 🎯 Helm Chart Structure

A Helm Chart folder looks like this:

```
my-app-chart/
├── Chart.yaml              # Chart metadata
├── values.yaml             # Default values
├── templates/
│   ├── deployment.yaml     # Deployment template
│   ├── service.yaml        # Service template
│   ├── configmap.yaml      # ConfigMap template
│   ├── _helpers.tpl        # Helper functions
│   └── NOTES.txt           # Post-install notes
├── values/
│   ├── dev.yaml            # Dev environment values
│   ├── staging.yaml        # Staging values
│   └── prod.yaml           # Production values
└── README.md               # Documentation
```

---

## Chart.yaml - Metadata

```yaml
apiVersion: v2
name: my-app
description: My application Helm chart
type: application
version: 1.0.0              # Chart version
appVersion: "1.0"           # App version
author: Your Name
home: https://example.com
sources:
  - https://github.com/example/my-app
keywords:
  - app
  - kubernetes
```

---

## values.yaml - Configuration

```yaml
# Default image
image:
  repository: my-app
  tag: "1.0"
  pullPolicy: IfNotPresent

# Replicas
replicaCount: 3

# Resources
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

# Service
service:
  type: ClusterIP
  port: 80

# Ingress
ingress:
  enabled: true
  host: app.example.com

# Database
database:
  host: db.example.com
  port: 5432
  user: admin
```

---

## templates/deployment.yaml - Template with Variables

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
      - name: app
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.port }}
        resources:
          limits:
            cpu: {{ .Values.resources.limits.cpu }}
            memory: {{ .Values.resources.limits.memory }}
```

---

## Understanding Template Variables

### `{{ .Release.Name }}`
The name you gave when installing the chart

```bash
helm install my-app ./chart
# {{ .Release.Name }} = my-app
```

### `{{ .Values.replicaCount }}`
Value from values.yaml

```yaml
replicaCount: 3
# {{ .Values.replicaCount }} = 3
```

### `{{ .Values.image.repository }}`
Nested value from values.yaml

```yaml
image:
  repository: nginx
# {{ .Values.image.repository }} = nginx
```

---

# 🎯 How to Install Helm

## Step 1: Download Helm

```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Windows
choco install kubernetes-helm
```

## Step 2: Verify Installation

```bash
helm version
# Output: version.BuildInfo{Version:"v3.12.0",...}
```

## Step 3: You're Ready!

```bash
helm --help
```

---

# 🚀 Using Helm - Step by Step

## Step 1: Add a Repository

```bash
# Add Bitnami repository (popular Helm charts)
helm repo add bitnami https://charts.bitnami.com/bitnami

# Update repositories
helm repo update
```

## Step 2: Search for Charts

```bash
# Search for WordPress
helm search repo wordpress

# Output:
# NAME                    CHART VERSION   APP VERSION   DESCRIPTION
# bitnami/wordpress       15.0.0          6.0           ...
```

## Step 3: Install a Chart

```bash
# Install WordPress
helm install my-wordpress bitnami/wordpress

# With custom values
helm install my-wordpress bitnami/wordpress \
  --set wordpressUsername=admin \
  --set wordpressPassword=mypassword
```

## Step 4: Check Installation

```bash
# List installed releases
helm list

# Get details
helm status my-wordpress

# Get values used
helm get values my-wordpress
```

## Step 5: Update Installation

```bash
# Upgrade to new version
helm upgrade my-wordpress bitnami/wordpress

# Upgrade with new values
helm upgrade my-wordpress bitnami/wordpress \
  --set replicaCount=5
```

## Step 6: Rollback if Needed

```bash
# Go back to previous version
helm rollback my-wordpress

# Go back 2 versions
helm rollback my-wordpress 2
```

## Step 7: Uninstall

```bash
# Remove everything
helm uninstall my-wordpress
```

---

# 🎓 Hands-On Exercise: Create Your First Helm Chart

## Exercise 1: Create a Simple Helm Chart

### Step 1: Create Chart Directory

```bash
mkdir my-first-chart
cd my-first-chart
```

### Step 2: Create Chart.yaml

Save as `Chart.yaml`:

```yaml
apiVersion: v2
name: my-first-app
description: My first Helm chart
type: application
version: 1.0.0
appVersion: "1.0"
```

### Step 3: Create values.yaml

Save as `values.yaml`:

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "latest"

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  host: myapp.example.com
```

### Step 4: Create templates folder

```bash
mkdir templates
```

### Step 5: Create Deployment Template

Save as `templates/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
      - name: app
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.port }}
```

### Step 6: Create Service Template

Save as `templates/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-service
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Release.Name }}
  ports:
  - port: {{ .Values.service.port }}
    targetPort: {{ .Values.service.port }}
```

### Step 7: Validate Your Chart

```bash
helm lint ./my-first-chart
# Output: 1 chart(s) linted, 0 error(s)
```

### Step 8: Test Your Chart (Dry Run)

```bash
helm install test-release ./my-first-chart --dry-run --debug

# Shows what YAML will be created without actually deploying
```

### Step 9: Install Your Chart

```bash
helm install my-app ./my-first-chart
```

### Step 10: Verify Installation

```bash
# Check release
helm list

# Check pods
kubectl get pods

# Check services
kubectl get svc
```

### Step 11: Update Your Chart

```bash
# Edit values.yaml
# Change replicaCount: 3

helm upgrade my-app ./my-first-chart
```

### Step 12: Verify Update

```bash
kubectl get pods
# Should show 3 pods now
```

### Step 13: Rollback

```bash
helm rollback my-app

kubectl get pods
# Back to 2 pods
```

### Step 14: Uninstall

```bash
helm uninstall my-app
```

✅ **You created your first Helm chart!**

---

# 📚 Helm Commands Reference

## Repository Commands

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo list
helm repo update
helm repo remove bitnami
helm search repo wordpress
helm search hub nginx
```

## Chart Commands

```bash
helm create my-chart              # Create new chart
helm lint my-chart                # Validate chart
helm package my-chart             # Create .tgz file
helm template my-chart            # Render templates
helm dependency list my-chart     # List dependencies
```

## Installation Commands

```bash
helm install my-app ./chart              # Install chart
helm install my-app ./chart -n myspace   # Install in namespace
helm install my-app ./chart -f values.yaml
helm install my-app ./chart --dry-run    # Test without installing
helm list                                 # List all releases
helm status my-app                       # Check status
```

## Update Commands

```bash
helm upgrade my-app ./chart               # Upgrade release
helm upgrade my-app ./chart --values prod-values.yaml
helm rollback my-app                     # Rollback to previous
helm rollback my-app 2                   # Rollback to version 2
helm history my-app                      # See all versions
```

## Inspection Commands

```bash
helm get values my-app                   # Get values used
helm get manifest my-app                 # Get rendered YAML
helm get notes my-app                    # Get installation notes
helm get all my-app                      # Get everything
```

## Deletion Commands

```bash
helm uninstall my-app                    # Delete release
helm uninstall my-app -n myspace         # Delete from namespace
helm uninstall my-app --keep-history     # Keep release history
```

---

# 🐛 Common Helm Mistakes

## ❌ Mistake 1: Typo in Chart Name

```bash
# WRONG
helm install myapp bitnami/wrodpress

# CORRECT
helm install myapp bitnami/wordpress
```

## ❌ Mistake 2: Missing Namespace Flag

```bash
# WRONG - Installs in default namespace
helm install myapp ./chart

# CORRECT - Installs in specific namespace
helm install myapp ./chart -n production
```

## ❌ Mistake 3: Wrong Values Syntax

```bash
# WRONG
helm install myapp ./chart --set replicas=3

# CORRECT
helm install myapp ./chart --set replicaCount=3
```

## ❌ Mistake 4: Not Updating Repos

```bash
# WRONG - Uses old chart
helm install myapp bitnami/wordpress

# CORRECT - Update first
helm repo update
helm install myapp bitnami/wordpress
```

## ❌ Mistake 5: Installing Without Validation

```bash
# WRONG - May have errors
helm install myapp ./chart

# CORRECT - Validate first
helm lint ./chart
helm install myapp ./chart
```

## ❌ Mistake 6: Upgrading Without Backup

```bash
# WRONG - Can't go back if something breaks
helm upgrade myapp ./chart

# CORRECT - Can rollback if needed
helm upgrade myapp ./chart
helm rollback myapp    # If needed
```

## ❌ Mistake 7: Using Old Values After Upgrade

```bash
# WRONG - Old values not applied
helm upgrade myapp ./chart

# CORRECT - Specify values
helm upgrade myapp ./chart -f values.yaml
```

## ❌ Mistake 8: Not Checking Deployment Status

```bash
# WRONG - Install and assume it worked
helm install myapp ./chart

# CORRECT - Check status
helm status myapp
kubectl get pods
```

## ❌ Mistake 9: Complex Helm Charts Without Testing

```bash
# WRONG - Deploy directly to production
helm install prod-app ./complex-chart

# CORRECT - Test first
helm install test-app ./complex-chart --dry-run --debug
helm install test-app ./complex-chart -n test
# Verify it works
helm install prod-app ./complex-chart -n prod
```

## ❌ Mistake 10: Not Using Helm Values Overrides

```yaml
# values.yaml (BAD - hardcoded)
replicaCount: 3
environment: production

# CORRECT - Use defaults, override when needed
replicaCount: 1        # Default
environment: dev       # Default
```

Then use:
```bash
helm install myapp ./chart
helm install myapp ./chart --set environment=prod --set replicaCount=5
```

---

# 📊 Helm vs kubectl

| Task | kubectl | Helm |
|------|---------|------|
| Deploy app | `kubectl apply -f file.yaml` | `helm install myapp chart` |
| Update app | Edit YAML, reapply | `helm upgrade myapp chart` |
| Rollback | Manual recreation | `helm rollback myapp` |
| Delete app | `kubectl delete all --all` | `helm uninstall myapp` |
| Version tracking | Manual | Automatic |
| Dependency management | Manual | Automatic |
| Reuse configs | Copy-paste | Helm templates |

---

# 🎓 Best Practices

✅ **Use helm lint** - Validate charts before using  
✅ **Use --dry-run** - Test before deploying  
✅ **Version your charts** - Track changes  
✅ **Use namespaces** - Organize releases  
✅ **Keep values in separate files** - Reuse charts  
✅ **Use helm rollback** - For quick rollbacks  
✅ **Document your charts** - README.md in chart  
✅ **Use helm hooks** - For lifecycle management  
✅ **Test locally first** - Before production  
✅ **Keep helmfile** - For managing multiple charts  

---

# 💡 When to Use Helm

Use Helm when:
- ✅ Deploying complete applications
- ✅ Need to manage multiple YAML files
- ✅ Want easy version management
- ✅ Need to deploy same app multiple times
- ✅ Working in teams
- ✅ Need rollback capability
- ✅ Reusing configurations

Don't use Helm when:
- ❌ Deploying single resources
- ❌ Simple one-off deployments
- ❌ Learning Kubernetes basics
- ❌ Quick troubleshooting

---

# 📖 Key Takeaways

✅ Helm is a package manager for Kubernetes  
✅ Charts are packages of Kubernetes manifests  
✅ Releases are running instances of charts  
✅ Values customize charts for different environments  
✅ Templates make charts reusable  
✅ helm install deploys, helm upgrade updates  
✅ helm rollback goes back to previous version  
✅ helm uninstall removes everything  
✅ Always validate with helm lint  
✅ Test with --dry-run before deploying  

---

# 🚀 Next Steps

1. ✅ Install Helm on your machine
2. ✅ Complete Exercise 1 (create first chart)
3. ✅ Install a pre-made chart (WordPress, nginx)
4. ✅ Customize values
5. ✅ Practice upgrade and rollback
6. ✅ Create your own complex chart
7. ✅ Push chart to repository
8. ✅ Use Helm in production

---

**Happy using Helm!** 🎉

Version: 1.0  
Date: November 12, 2025  
Status: ✅ Complete & Ready
