# Kubernetes Pod Troubleshooting Guide

A beginner-friendly guide to understand common Kubernetes pod issues, when they happen, and how to fix them.

---

## Table of Contents
1. [Pending Issues](#pending-issues)
2. [Configuration Issues](#configuration-issues)
3. [Crash Loop Issues](#crash-loop-issues)
4. [Networking Issues](#networking-issues)

---

## Pending Issues

### Issue 1: Insufficient Resources (CPU or Memory)

**What is it?**
Your pod wants to run, but the Kubernetes cluster doesn't have enough CPU or memory available on any node to host it.

**When does it happen?**
- You request too much CPU or memory in your pod specification
- Other pods are already using most of the resources on all nodes
- Your cluster doesn't have enough nodes to handle all the pods

**How to identify it?**
Run this command:
```bash
kubectl describe pod <pod-name>
```
Look at the **Events** section. You will see messages like:
- "Insufficient CPU"
- "Insufficient Memory"

**How to fix it?**

Option 1: Reduce resource requests in your pod YAML
```yaml
resources:
  requests:
    cpu: 100m        # Reduce this value
    memory: 128Mi     # Reduce this value
  limits:
    cpu: 500m
    memory: 512Mi
```

Option 2: Check how much resources are currently used
```bash
kubectl top nodes         # See node resource usage
kubectl top pods          # See pod resource usage
```

Option 3: Add more nodes to your cluster (scale up infrastructure)

Option 4: Delete unnecessary pods to free up resources
```bash
kubectl delete pod <pod-name>
```

---

### Issue 2: Node Affinity Mismatch

**What is it?**
Your pod requires a specific node label (tag) that doesn't exist on any available node.

**When does it happen?**
- You set a node affinity rule that requires pods to run on specific nodes
- Those specific nodes don't exist in your cluster, or the labels don't match
- You want to run a pod on a node with special hardware (like GPU), but no such node is available

**Example of node affinity:**
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disk-type
          operator: In
          values:
          - ssd          # Pod requires node with "ssd" label
```

**How to identify it?**
```bash
kubectl describe pod <pod-name>
```
Look for messages like:
- "No matching node found for affinity"
- "Pod unable to find matching nodes"

**How to fix it?**

Option 1: Label your nodes with the required labels
```bash
kubectl label nodes <node-name> disk-type=ssd
```

Option 2: Check existing node labels
```bash
kubectl get nodes --show-labels
```

Option 3: Remove the affinity requirement and let Kubernetes place the pod anywhere
```yaml
# Remove or comment out the affinity section
affinity: {}
```

Option 4: Change affinity to "preferred" instead of "required" (soft constraint)
```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:    # Changed from "required"
      - weight: 100
        preference:
          matchExpressions:
          - key: disk-type
            operator: In
            values:
            - ssd
```

---

### Issue 3: Unbound Persistent Volume (Storage Issue)

**What is it?**
Your pod needs persistent storage (PersistentVolumeClaim), but no available storage volume matches what you requested.

**When does it happen?**
- The storage class doesn't exist
- No PersistentVolume (PV) available with the size you requested
- The PVC is waiting to be bound to a PV
- Dynamic provisioning failed

**Example:**
Your pod asks for 100GB storage, but only 50GB volumes are available.

**How to identify it?**
```bash
kubectl describe pod <pod-name>
# Look for: "Pod has unbound PersistentVolumeClaims"

kubectl get pvc
# Check if STATUS is "Pending" instead of "Bound"
```

**How to fix it?**

Option 1: Create a PersistentVolume matching your claim
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 100Gi      # Match the size requested in PVC
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  # Add storage backend details (NFS, cloud storage, etc.)
```

Option 2: Change your PVC to request less storage
```yaml
resources:
  requests:
    storage: 50Gi       # Reduce to match available PV
```

Option 3: Check available PVs
```bash
kubectl get pv
```

Option 4: Verify your StorageClass exists
```bash
kubectl get storageclass
```

---

### Issue 4: Node Taint Mismatch

**What is it?**
The target node has a "taint" (restriction) that your pod doesn't have permission to tolerate.

**When does it happen?**
- A node is marked as tainted (e.g., "maintenance needed" or "reserved for special workloads")
- Your pod doesn't have a matching toleration
- Only specific pods can run on that node

**How to identify it?**
```bash
kubectl describe pod <pod-name>
# Look for: "Pod has unmatched taints"

kubectl describe node <node-name>
# Look for: Taints section showing applied taints
```

**How to fix it?**

Option 1: Add toleration to your pod
```yaml
tolerations:
- key: maintenance          # Match the taint key
  operator: Equal
  value: needed             # Match the taint value
  effect: NoSchedule        # Match the taint effect
```

Option 2: Check what taints exist on nodes
```bash
kubectl describe node <node-name> | grep Taints
```

Option 3: Remove the taint from the node (if not needed)
```bash
kubectl taint nodes <node-name> maintenance:NoSchedule-
# The minus (-) at the end removes the taint
```

Option 4: Add a taint to a node
```bash
kubectl taint nodes <node-name> maintenance=needed:NoSchedule
```

---

## Configuration Issues

### Issue 5: Missing ConfigMap

**What is it?**
Your container tries to use a ConfigMap (configuration file) that doesn't exist.

**When does it happen?**
- You reference a ConfigMap name in your pod YAML
- That ConfigMap was never created
- The ConfigMap was deleted
- The ConfigMap is in a different namespace

**Error message:**
Status shows "CreateContainerConfigError"

**How to identify it?**
```bash
kubectl describe pod <pod-name>
# Look for: "couldn't find key <key-name> in ConfigMap"

kubectl get configmap
# Check if your ConfigMap exists
```

**How to fix it?**

Option 1: Create the missing ConfigMap
```bash
kubectl create configmap my-config --from-literal=key=value
```

Option 2: Create ConfigMap from a file
```bash
kubectl create configmap my-config --from-file=config.txt
```

Option 3: Define ConfigMap in YAML
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  database_url: "postgresql://localhost"
  debug_mode: "false"
```

Option 4: Verify the ConfigMap name in your pod YAML matches exactly
```yaml
containers:
- name: myapp
  envFrom:
  - configMapRef:
      name: my-config    # Must match ConfigMap name exactly
```

---

### Issue 6: Missing Secret

**What is it?**
Your container tries to use a Secret (sensitive data like passwords) that doesn't exist.

**When does it happen?**
- You reference a Secret name in your pod YAML
- That Secret was never created
- The Secret was deleted
- The Secret is in a different namespace

**Error message:**
Status shows "CreateContainerConfigError"

**How to identify it?**
```bash
kubectl describe pod <pod-name>
# Look for: "couldn't find secret"

kubectl get secret
# Check if your Secret exists
```

**How to fix it?**

Option 1: Create a Secret from literal values
```bash
kubectl create secret generic my-secret --from-literal=username=admin --from-literal=password=secret123
```

Option 2: Create a Secret from a file
```bash
kubectl create secret generic my-secret --from-file=.dockercfg
```

Option 3: Define Secret in YAML
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: YWRtaW4=           # base64 encoded "admin"
  password: c2VjcmV0MTIz=     # base64 encoded "secret123"
```

Option 4: Use the Secret in your pod
```yaml
containers:
- name: myapp
  env:
  - name: USERNAME
    valueFrom:
      secretKeyRef:
        name: my-secret       # Must match Secret name
        key: username
```

---

### Issue 7: Resource Quota Exceeded

**What is it?**
You want to create more pods, but a resource quota (limit) in your namespace prevents it.

**When does it happen?**
- Your namespace has a ResourceQuota set
- The total number of pods exceeds the quota
- Total CPU or memory requests exceed the quota limit
- You've reached the maximum allowed pods in the namespace

**Error message:**
"Pod exceeded quota"

**How to identify it?**
```bash
kubectl get resourcequota
# Shows quotas in current namespace

kubectl describe resourcequota <quota-name>
# Shows current usage vs limits
```

**How to fix it?**

Option 1: Check current quota usage
```bash
kubectl get resourcequota
kubectl describe resourcequota <quota-name>
```

Option 2: Delete unnecessary pods to free up quota
```bash
kubectl delete pod <pod-name>
```

Option 3: Increase the resource quota (if you have admin access)
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: my-quota
spec:
  hard:
    pods: "100"           # Increase pod limit
    requests.cpu: "100"   # Increase CPU limit
    requests.memory: "200Gi"  # Increase memory limit
```

Option 4: Reduce resource requests in your pods to use less quota
```yaml
resources:
  requests:
    cpu: 50m          # Use less CPU
    memory: 64Mi      # Use less memory
```

---

## Crash Loop Issues

### Issue 8: Image Pull Backoff

**What is it?**
Your pod can't start because Kubernetes can't download the container image.

**When does it happen?**
- Image name has a typo (e.g., "ngin" instead of "nginx")
- Image tag is wrong or doesn't exist
- Image repository is private and you don't have credentials
- Image repository doesn't exist
- Network connectivity issue preventing download

**Error message:**
Status shows "ImagePullBackOff"

**How to identify it?**
```bash
kubectl describe pod <pod-name>
# Look for: "Failed to pull image"
# Look for: "Repository does not exist" or "Tag not found"
```

**How to fix it?**

Option 1: Verify and correct the image name
```bash
# First, test pulling the image locally
docker pull nginx:1.14.2

# Then update your pod YAML with correct name/tag
containers:
- name: myapp
  image: nginx:1.14.2    # Correct image name and tag
```

Option 2: Check available image tags
```bash
# Visit Docker Hub or your registry to verify correct tag
# Example: https://hub.docker.com/r/library/nginx/tags
```

Option 3: Use image digest instead of tag (more reliable)
```yaml
containers:
- name: myapp
  image: nginx@sha256:abc123...  # Use digest for exact version
```

Option 4: For private images, create an image pull secret
```bash
kubectl create secret docker-registry my-secret \
  --docker-server=myregistry.com \
  --docker-username=myuser \
  --docker-password=mypass \
  --docker-email=my@email.com
```

Then use it in your pod:
```yaml
imagePullSecrets:
- name: my-secret    # Reference the secret
```

---

### Issue 9: Crash Loop Backoff (Out of Memory - OOM)

**What is it?**
Your application keeps running out of memory, causing Kubernetes to restart the container repeatedly.

**When does it happen?**
- Your application uses more memory than the limit allows
- Memory leak in your application
- Limit is set too low for your application needs
- Sudden spike in data processing

**Error message:**
- Status shows "CrashLoopBackOff"
- Exit code shows "137" (OOMKilled)

**How to identify it?**
```bash
kubectl describe pod <pod-name>
# Look for: "OOMKilled" or "Exit Code 137"

kubectl logs <pod-name> --previous
# Shows logs from previous crashed container
```

**How to fix it?**

Option 1: Increase memory limit
```yaml
resources:
  limits:
    memory: 512Mi       # Increase from current value
  requests:
    memory: 256Mi
```

Option 2: Analyze and reduce application memory usage
```bash
# Monitor memory usage while running
kubectl top pod <pod-name>

# Check memory usage trends
kubectl logs <pod-name> | grep memory
```

Option 3: Add memory request (reserve memory upfront)
```yaml
resources:
  requests:
    memory: 256Mi       # Reserve memory in advance
  limits:
    memory: 512Mi
```

Option 4: Optimize your application
- Fix memory leaks in your code
- Use smaller base images
- Enable memory caching properly
- Process data in batches instead of all at once

---

### Issue 10: Crash Loop Backoff (Health Check Failure)

**What is it?**
Your container keeps crashing because the liveness or readiness probe is failing repeatedly.

**When does it happen?**
- Health check endpoint is not responding correctly
- Health check is too strict (fails too easily)
- Application needs more time to start
- Health check is checking wrong endpoint

**Error message:**
- Status shows "CrashLoopBackOff"
- Events show "Probe failed"

**How to identify it?**
```bash
kubectl describe pod <pod-name>
# Look for: "Liveness probe failed" or "Readiness probe failed"

kubectl logs <pod-name>
# Check application logs for errors
```

**How to fix it?**

Option 1: Increase initial delay (give app time to start)
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30    # Wait 30 seconds before first check
  periodSeconds: 10          # Check every 10 seconds
  failureThreshold: 3        # Allow 3 failures before restart

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 15    # Wait 15 seconds before first check
  periodSeconds: 5           # Check every 5 seconds
```

Option 2: Verify health check endpoint exists
```bash
kubectl exec <pod-name> -- curl http://localhost:8080/health
# Should return a successful response
```

Option 3: Relax probe settings temporarily to debug
```yaml
livenessProbe:
  initialDelaySeconds: 60    # Give more time
  failureThreshold: 5        # Allow more failures
```

Option 4: Check application logs
```bash
kubectl logs <pod-name>
kubectl logs <pod-name> --previous    # Previous crashed instance
```

Option 5: Use a simpler health check
```yaml
livenessProbe:
  tcpSocket:         # Just check if port is open
    port: 8080
  initialDelaySeconds: 20
```

---

### Issue 11: Crash Loop Backoff (Init Container Failure)

**What is it?**
Your pod's init container (startup container) is failing, preventing the main application container from starting.

**When does it happen?**
- Init container command has an error or typo
- Init container references missing files or resources
- Init container network connectivity issues
- Init container doesn't complete successfully

**Error message:**
- Status shows "CrashLoopBackOff"
- Events show "Init container failed"

**How to identify it?**
```bash
kubectl describe pod <pod-name>
# Look for: "Init container failed" in events

kubectl logs <pod-name> -c init-container-name
# Shows logs from init container
```

**How to fix it?**

Option 1: Check init container logs
```bash
kubectl logs <pod-name> -c init-container-name
# Understand why it's failing
```

Option 2: Verify init container command is correct
```yaml
initContainers:
- name: init
  image: alpine
  command: ['sh', '-c', 'echo "Hello" > /data/file.txt']
  # Make sure command syntax is correct
```

Option 3: Ensure required files/volumes exist
```yaml
initContainers:
- name: init
  image: alpine
  volumeMounts:
  - name: data
    mountPath: /data
volumes:
- name: data
  emptyDir: {}
```

Option 4: Allow init container more time if it's slow
```yaml
initContainers:
- name: init
  image: alpine
  # No direct timeout, but overall pod startup time matters
```

Option 5: Simplify init container for debugging
```yaml
initContainers:
- name: init
  image: alpine
  command: ['echo', 'Init starting']  # Simple debug command
```

---

### Issue 12: Crash Loop Backoff (Application Runtime Error)

**What is it?**
Your application crashes because of an error in the code or missing dependencies (like database connection).

**When does it happen?**
- Application code has a bug
- Application can't connect to database
- Required service is not available
- Environment variables are missing
- Configuration is incorrect

**Error message:**
- Status shows "CrashLoopBackOff"
- Exit code shows "1" or other non-zero number

**How to identify it?**
```bash
kubectl describe pod <pod-name>
# Look at events for error messages

kubectl logs <pod-name>
# See application error messages

kubectl logs <pod-name> --previous
# See logs from previous crashed container
```

**How to fix it?**

Option 1: Check application logs for the actual error
```bash
kubectl logs <pod-name>
# Read the error message carefully
```

Option 2: Verify environment variables are set
```yaml
env:
- name: DATABASE_URL
  value: "postgresql://db.example.com"
- name: DATABASE_USER
  value: "admin"
# Add all required environment variables
```

Option 3: Use ConfigMap or Secret for configuration
```yaml
envFrom:
- configMapRef:
    name: app-config
- secretRef:
    name: app-secrets
```

Option 4: Check if required services are available
```bash
kubectl get svc
# Verify dependent services are running
```

Option 5: Add resource limits if app is crashing due to resource issues
```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

Option 6: Increase liveness probe delay to give app time to initialize
```yaml
livenessProbe:
  initialDelaySeconds: 60    # Wait longer for app to start
```

---

## Networking Issues

### Issue 13: Application Not Accessible (Service/Networking Issue)

**What is it?**
Your pod is running fine, but users can't access the application. Usually caused by service selector mismatch or networking configuration issues.

**When does it happen?**
- Service selector labels don't match pod labels
- Pod doesn't have the required labels
- Service port doesn't match container port
- Network policies are blocking traffic
- Firewall rules blocking access

**How to identify it?**
```bash
kubectl get pods --show-labels
# Check if pods have required labels

kubectl get svc
# Check services exist

kubectl describe svc <service-name>
# Check service configuration
```

**Example of the problem:**

Pod labels:
```yaml
labels:
  app: myapp
  version: v1
```

Service selector (doesn't match!):
```yaml
selector:
  app: webapp    # Wrong label name!
  version: v1
```

**How to fix it?**

Option 1: Fix service selector to match pod labels
```bash
# First, check pod labels
kubectl get pods --show-labels

# Then, verify service selector matches
kubectl describe svc <service-name> | grep Selector
```

Update service YAML:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: myapp         # Must match pod label
    version: v1        # Must match pod label
  ports:
  - protocol: TCP
    port: 80           # External port
    targetPort: 8080   # Container port
```

Option 2: Add missing labels to pods
```bash
kubectl label pod <pod-name> app=myapp version=v1
```

Or in deployment YAML:
```yaml
template:
  metadata:
    labels:
      app: myapp         # Add required labels
      version: v1
```

Option 3: Verify port configuration
```yaml
containers:
- name: myapp
  image: myapp:1.0
  ports:
  - containerPort: 8080    # Container listens on this port

---
spec:
  selector:
    app: myapp
  ports:
  - port: 80               # Service exposes this port
    targetPort: 8080       # Forwards to container port
```

Option 4: Check network policies
```bash
kubectl get networkpolicy
# See if network policies are restricting traffic

kubectl describe networkpolicy <policy-name>
# Check policy rules
```

If blocking traffic, create allowing policy:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-traffic
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}      # Allow from any pod
```

Option 5: Test connectivity from another pod
```bash
kubectl exec <test-pod> -- curl http://<service-name>:80
# Test if service is accessible internally
```

Option 6: Check service endpoints
```bash
kubectl get endpoints <service-name>
# Should show pod IP addresses
# If empty, selector is not matching pods
```

---

## Quick Troubleshooting Checklist

When your pod has issues, follow this checklist:

**Step 1: Get pod status**
```bash
kubectl get pods
kubectl describe pod <pod-name>
```

**Step 2: Check events**
Look at the "Events" section in describe output. This tells you what's wrong.

**Step 3: Check logs**
```bash
kubectl logs <pod-name>
kubectl logs <pod-name> --previous   # Previous instance if crashed
```

**Step 4: Identify the issue type**
- **Pending?** → Check resources, affinity, taints, PVC, or quotas
- **CrashLoopBackOff?** → Check logs, memory, health probes, init containers
- **ImagePullBackOff?** → Check image name, tag, or credentials
- **CreateContainerConfigError?** → Check ConfigMap or Secret
- **Not accessible?** → Check service selector and pod labels

**Step 5: Apply the fix**
Use the appropriate section above to fix the identified issue.

**Step 6: Verify**
```bash
kubectl get pods              # Check pod status
kubectl describe pod <name>   # Check details
```

---

## Useful Debugging Commands

```bash
# View all pods in current namespace
kubectl get pods

# Get detailed pod information
kubectl describe pod <pod-name>

# View pod logs
kubectl logs <pod-name>

# View logs from previous container
kubectl logs <pod-name> --previous

# Execute command inside pod
kubectl exec -it <pod-name> -- bash

# Check resource usage
kubectl top nodes
kubectl top pods

# View events in cluster
kubectl get events

# Check service details
kubectl describe svc <service-name>

# View pod labels
kubectl get pods --show-labels

# Check ConfigMaps
kubectl get configmap
kubectl describe configmap <name>

# Check Secrets
kubectl get secret
kubectl describe secret <name>

# Check PersistentVolumes
kubectl get pv
kubectl get pvc

# Check ResourceQuotas
kubectl get resourcequota

# Check Taints
kubectl describe node <node-name> | grep Taints
```

---

## Summary

Most Kubernetes pod issues fall into these categories:

1. **Scheduling Issues** → Pod can't get scheduled (Pending)
2. **Configuration Issues** → Missing ConfigMap/Secret
3. **Resource Issues** → Not enough CPU/Memory or exceeded quota
4. **Image Issues** → Can't download container image
5. **Application Issues** → App crashes or health check fails
6. **Networking Issues** → Service selector mismatch

Always start by checking pod status and events. The error messages are usually very helpful in identifying the root cause. Don't skip reading the logs – they often tell you exactly what's wrong!

Remember: Use `kubectl describe` and `kubectl logs` as your first debugging tools.