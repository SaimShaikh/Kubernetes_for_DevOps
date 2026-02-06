## Q1. What is Kubernetes and what does it do?

Kubernetes is an open-source platform that manages and automates containerized applications.
It handles deployment, scaling, load balancing, and self-healing of containers across multiple servers. If a container crashes or traffic increases, Kubernetes automatically fixes or scales the application, ensuring high availability with minimal manual effort.

---

## Q2. List the main components of Kubernetes architecture
A Kubernetes is Divided into 2 part (1 Master node (control plan) and 2 is Woker node)

- Master node (Control node Components) - This is where decisions are made.
1. API Server - API Server is the component that receives and handles all Kubernetes requests.
2. etcd - etcd is a key-value database that stores the entire state and configuration of a Kubernetes cluster.
3. Scheduler - The Scheduler is responsible for selecting the most suitable node for a pod and assigning it based on resource availability and scheduling rules.
4. Controller Manager - Controller Manager continuously monitors the cluster and ensures the actual state matches the desired state.

- Worker Node - This is where applications run.

  1. kubelet - kubelet is a node-level agent that ensures pods and containers are running as defined in the cluster.
  2. kube-proxy - kube-proxy manages network rules to allow services to route traffic to the correct pods.
  3.  Pod - A Pod is the smallest unit in Kubernetes that runs one or more containers sharing the same network and storage. and Pods are temporary. Don’t treat them like servers.

- Out of this
  1. CNI (Container Network Interface) - CNI is the networking system used by Kubernetes to connect pods to the network.CNI is responsible for: Assigning IP addresses to pods , Enabling pod-to-pod communication Making sure pods can talk across nodes , Setting up network routes and rules
  2. Runs containers inside Pods. Responsibilities: Pulls container images. , Starts and stops containers. , Reports container status to Kubelet. , Common runtimes: containerd, CRI-O (Docker used to be, now deprecated).

---

## Q3. You have an application deployed onKubernetes that is experiencing increased traffic. How would you scale the application to handle the increased load?
First, I would identify the bottleneck by analyzing resource usage such as CPU, memory, and network to understand what is limiting the application’s performance.

If the bottleneck is CPU or memory, I would go for horizontal scaling by increasing the number of pod replicas using Horizontal Pod Autoscaler (HPA). This helps distribute the traffic across multiple pods.

If the issue is resource-specific and cannot be solved by adding more pods, I would use vertical scaling by increasing the resource limits and requests for each pod.

After scaling, I would continuously monitor the application to make sure performance has improved and the system remains stable without overloading the cluster.


---

## Q4. While troubleshooting a networking issue in the cluster, you noticed kube-proxy in the logs. What is the role of kube-proxy in Cluster?
kube-proxy is a Kubernetes component that runs on every node and manages network communication between services and pods.
It handles TCP and UDP traffic forwarding by maintaining network rules (like iptables or IPVS) that route incoming service traffic to the correct backend pods.

---

## Q5. Your team is planning a high-availability Kubernetes cluster. Describe the Process and Considerations for Designing a High-Availability Kubernetes Cluster
To design a high-availability Kubernetes cluster, the main goal is to remove single points of failure

- Multi-Master Setup
We deployed multiple control plane (master) nodes across three availability zones. This ensures redundancy and prevents the cluster from going down if one master or zone fails.

- etcd Distribution
We distributed etcd members across the same availability zones as the master nodes. This provides data redundancy and ensures the cluster state remains available even during zone-level failures.

- API Server Load Balancing
We configured a TCP load balancer in front of the API servers to distribute requests evenly. This avoids overloading a single API server and ensures continuous access to the cluster.

- Node Auto-Repair
We enabled node auto-repair, which automatically detects unhealthy nodes and replaces them. This helps maintain cluster health and availability without manual intervention.


---

## Q6. In your Kubernetes environment, a master or worker node suddenly fails. What happens when the master or the worker node fails?

1. Master (Control Plane) Node Failure

If the master node fails, the running applications are not affected.
All existing pods on worker nodes continue to run normally.
However, cluster management is impacted, which means:

- No new pods can be scheduled

- No scaling or deployment changes can be made

- kubectl and API operations may not work

2. Worker Node Failure

When a worker node fails, Kubernetes automatically detects it.

What happens step by step:

- The master marks the node as NotReady

- Pods running on that node become unavailable

- Kubernetes evicts those pods

- The scheduler recreates the pods on healthy worker nodes

This process usually happens within 1 to 7 minutes, depending on configuration.

---

## Q7. How does ingress help in Kubernetes?

Ingress in Kubernetes is a resource used to expose services to external traffic.
It defines rules that control how external users access applications running inside the cluster.

Ingress helps Kubernetes by allowing you to:

- Expose multiple services using a single IP address or domain name

- Route traffic based on host name or URL path (for example /app1, /app2)

- Provide load balancing across backend services

- Handle SSL/TLS termination for secure HTTPS traffic

- Support name-based virtual hosting

- Simplify service access and configuration


---

## Q8. List the different types of services in Kubernetes

Kubernetes Services are used to expose applications (pods) inside or outside the cluster.

1. ClusterIP (Default)

Exposes the service inside the cluster only.

Accessible only within Kubernetes

Used for internal communication between services

Example use:
Backend service accessed by a frontend inside the cluster

2. NodePort

Exposes the service on a static port on each worker node.

Accessible using <NodeIP>:<NodePort>

Used for basic external access

Example use:
Accessing an app directly for testing or demo

3. LoadBalancer

Exposes the service using a cloud provider’s load balancer.

Automatically creates an external load balancer

Common in AWS, Azure, GCP

Example use:
Production applications needing public access

4. ExternalName

Maps a service to an external DNS name.

No selector or pods

Used to access services outside the cluster

Example use:
Connecting to an external database or API


___

## Q9 What is a Headless Service and what is its use?
A Headless Service in Kubernetes is a service without a ClusterIP.
Instead of routing traffic through a single service IP, gives pod IPs directly using DNS, no load balancing required .

In a database cluster, each database pod needs its own address. A headless service allows clients to connect directly to a specific database pod.

---

## Q10. Your manager has instructed you to run several scripts before starting the main application in your Kubernetes pod, and suggested using init containers. What is the init container?

An init container is a special type of container in Kubernetes that runs before the main application containers in a pod.
Its main purpose is to perform in
itialization or setup tasks that the main application depends on.
Init containers always run to completion, and only after they succeed do the main containers start.

- Downloading configuration files or secrets

- Waiting for another service (like a database) to be ready

- Initializing a database schema

- Setting up network or permissions

- Running pre-start scripts or checks

---


## Q11 A Critical application running on one of nodes is not working properly. How do you monitor applications in Kubernetes?
I first check the pod status and logs to identify the issue, then use monitoring and alerting tools to understand resource usage and performance.

---

## Q12 Company is very concerned about Securing Clusters. List some security measures that you can take while using Kubernetes.

1. RBAC: Restrict access based on user roles.
2. Network Policies: Control pod-to-pod and external communication.
3. Container Security: Ensure secure runtimes and image scanning.
4. Secrets Management: Safeguard sensitive data with Kubernetes secrets.
5. Audit Logging: Track activities for security monitoring.
6. Update and Patching: Maintain components and nodes current version.

--- 

## Q13 Explain Kubernetes RBAC

RBAC (Role-Based Access Control) in Kubernetes restricts access to resources based on
user roles.
You define Roles with specific permissions (e.g., create, delete) and bind them to users or
groups using RoleBindings.
RBAC ensures users and applications have access only to necessary resources, following
the principle of least privilege and enhancing security

---

## Q14. How do you perform maintenance on the K8 node?
To perform maintenance on a Kubernetes node, 
- I first make the node unschedulable using kubectl cordon so that no new pods are placed on it.
- Then I safely move the running pods to other healthy nodes using kubectl drain --ignore-daemonsets, which ensures applications remain available.
- After draining the node, I perform the required maintenance tasks such as OS updates, patches, or configuration changes.
- If needed, I reboot the node.
- Once maintenance is completed, I mark the node schedulable again using kubectl uncordon.
Finally, I verify the node status using kubectl get nodes to ensure it is ready and functioning properly.

---

## Q15. What is a ConfigMap in Kubernetes?

A ConfigMap in Kubernetes is used to store non-confidential configuration data separately from application code.
It allows us to manage configuration like environment variables, application settings, or config files without rebuilding container images.

What can be stored in a ConfigMap

- Environment variables

- Application properties

- Config files (YAML, JSON, .conf files)

- Command-line arguments

---

## Q16. What is a Secret in Kubernetes?

A Secret in Kubernetes is used to store sensitive information securely, such as passwords, API keys, tokens, or certificates.

---

## Q17. What is the purpose of Operators

A Kubernetes operator is a method of packaging, deploying, and managing a Kubernetes
application. An operator uses the Kubernetes API to automate tasks such as deployment,
scaling, implementing custom controllers, enabling self-healing systems, and supporting
declarative management practices


---

## Q18. How can you run a pod on a specific node?
A pod can be run on a specific node in Kubernetes by using nodeSelector, node affinity, or taints and tolerations.

first I will assign the lables to nodes using

``` bash
kubectl label nodes node-01 nodelocation= <any name>
```
Then, in the pod specification, specify either node affinity or node selector to ensure the pod is scheduled on that node:

``` bash
apiVersion: v1
kind: Pod
metadata:
 name: mypod
spec:
 containers:
 - name: mycontainer
 image: nginx
 affinity:
 nodeAffinity:
 requiredDuringSchedulingIgnoredDuringExecution:
 nodeSelectorTerms:
 - matchExpressions:
 - key: nodelocation
 operator: In
 values:
 - <node name >
```
This ensures the pod ( mypod ) runs on the node labeled with nodelocation= <Node name>

---

## Q19. Suppose a pod exceeds its memory limit. What signal will be sent to the process? How do you solve a pod exceeding its memory limit?

When a pod exceeds its memory limit and gets OOMKilled killer may send the SIGKILL signal to the process, terminating it immediately, the solution is to identify the cause and fix memory usage.

Steps to solve the issue
1. Check pod memory usage

First, I check how much memory the pod is actually using.

Review pod metrics

Check logs for OOMKilled events

2. Update memory limits

If the limit is too low, I increase the memory limit based on real usage.

Set proper requests and limits to avoid frequent restarts.

3. Fix memory leaks

If the application has a memory leak, I work with the dev team to fix it.

Improve garbage collection

Optimize code or dependencies

---

## Q20. Can you schedule the pods to the node if the node is tainted?

By default, Kubernetes does not schedule pods on a tainted node.
However, a pod can be scheduled on a tainted node if it has a matching toleration for that taint.

Apply a taint to a node:
```kubectl taint nodes node1 key=value:NoSchedul```
Apply toleration to a pod:

```bash

spec:
tolerations:
- key: "key"
operator: "Equal"
value: "value"
effect: "NoSchedule"

```
---

## Q21. ReplicaSet. Vs ReplicaSet Controller

A ReplicaSet defines the desired number of Pod replicas, while the ReplicaSet Controller
ensures that this desired state is always maintained.
If Pods are deleted or fail, the controller automatically creates new Pods to match the
ReplicaSet specification.

---
## Q22. DaemonSet in Kubernetes 
A DaemonSet in Kubernetes ensures that a specific Pod runs on every node in the cluster.
It is mainly used for system-level services like logging, monitoring, and networking agents
that must be present on all nodes.

---

## Q23. stateful set 

A StatefulSet in Kubernetes is used to manage stateful applications that require stable Pod
identities and persistent storage.
It ensures that Pods have fixed names, dedicated storage, and are created or deleted in an
ordered manner, making it suitable for databases and distributed systems.

---

## Q24. What are Labels and Selectors in Kubernetes?

Labels in Kubernetes are key–value pairs used to identify and group resources like Pods
and Services.
Selectors are used to select resources based on these labels.
Together, labels and selectors allow Kubernetes to manage, group, and connect resources efficiently.

---

## Q25. What is a Service Account in Kubernetes?

A Service Account in Kubernetes is used to provide an identity to pods so they can securely interact with the Kubernetes API or other cluster resources.
It is mainly used for applications running inside the cluster, not for human users.

Why Service Accounts are used

- Pods need permission to access Kubernetes resources

- Avoid using admin credentials inside applications

- Follow least privilege principle

I- mprove cluster security

How Service Accounts work

- Each pod is associated with a service account

- Kubernetes automatically mounts a token inside the pod

- The token is used to authenticate with the Kubernetes API

- Permissions are controlled using RBAC

---

## Q26. Kubernetes Deployment Strategies. 

A deployment strategy in Kubernetes defines how traffic is shifted from the old version of an application to the new version during an update.
Kubernetes supports multiple deployment strategies to ensure smooth releases with minimal risk and downtime.

1. Rolling Update (Default Strategy)

Rolling Update gradually replaces old pods with new ones.

Pods are updated one by one

No downtime

Default strategy in Kubernetes

Use case:

Most production applications

2. Recreate Strategy

Recreate stops all old pods before starting new pods.

Causes downtime

Simple but risky for production

Use case:

Non-critical applications or dev environments

3. Blue-Green Deployment

Blue-Green maintains two identical environments:

Blue → current live version

Green → new version

Traffic is switched instantly after testing.

Zero downtime

Easy rollback

Use case:

Critical applications needing safe releases

4. Canary Deployment

Canary releases the new version to a small percentage of users first.

Reduces risk

Gradual rollout

Monitor before full release

Use case:

High-risk or high-traffic applications


---

## Q27. What is deployment in Kubernetes
A Deployment in Kubernetes is a resource used to manage and control Pods for an application.
It ensures that the desired number of pod replicas are always running and handles updates, scaling, and self-healing automatically.

What a Deployment does

- Creates and manages ReplicaSets

- Keeps the desired number of pods running

- Automatically restarts pods if they fail

- Supports rolling updates and rollbacks

- Allows scaling up or down easily

---

## Q28 Why should we use custom namespaces ?

By creating custom namespaces, you can logically group your resources based on your
needs, such as separating production and development environments or separating
applications by team or department. This makes it easier to manage and maintain your
resources within the cluster, and also provides better security and resource isolation.


---

## Q29 What you will do to upgrade a Kubernetes cluster?

To upgrade a Kubernetes cluster, I follow a planned and step-by-step approach to avoid downtime and issues.
I upgrade the control plane first, then the worker nodes, while ensuring applications remain available.

Steps to upgrade a Kubernetes cluster
1. Check Kubernetes version compatibility

First, I check the Kubernetes release notes and make sure the new version is compatible with:

Current cluster version

Add-ons (CNI, CoreDNS, kube-proxy)

Applications

2. Take backups

Before upgrading, I take backups of:

etcd (cluster state)

Important manifests and configurations

This helps in recovery if something goes wrong.

3. Upgrade the Control Plane (Master Node)

I upgrade the control plane components first:

kube-apiserver

kube-controller-manager

kube-scheduler

Applications continue running during this step.

4. Upgrade Worker Nodes (One by One)

For worker nodes, I:

Cordon the node

Drain workloads safely

Upgrade kubelet and kube-proxy

Bring the node back (uncordon)

This ensures zero or minimal downtime.

5. Upgrade Cluster Add-ons

After nodes are upgraded, I update:

CNI plugin

CoreDNS

kube-proxy

6. Verify the Cluster

Finally, I verify:

Node status

Pod health

Application availability

---

## Q30. How does Kubernetes perform load balancing?

Kubernetes performs load balancing mainly using Services.
A Service provides a single stable IP and DNS name and automatically distributes traffic across multiple Pods running the application.

How it works (simple explanation)

Pods are dynamic

Pods can restart, scale up/down, or get new IPs.

Clients should not directly talk to Pods.

Service sits in front of Pods

A Service selects Pods using labels.

It exposes a stable virtual IP called ClusterIP.

Traffic distribution

When a request comes to the Service IP, Kubernetes:

Uses kube-proxy

Forwards traffic to one of the healthy Pods

Uses round-robin or similar algorithms

Types of load balancing in Kubernetes
1. Internal Load Balancing

Done using Service (ClusterIP)

Used for Pod-to-Pod communication inside the cluster

2. External Load Balancing

Service type LoadBalancer

Cloud provider creates an external load balancer (AWS ELB, GCP LB, etc.)

Traffic → LoadBalancer → Service → Pods

3. HTTP/HTTPS Load Balancing

Done using Ingress

Provides:

Path-based routing

Host-based routing

SSL termination

---

## Q31. A service is not reaching the correct Pods. How do you debug ?

Step-by-step debugging approach
1. Check Pods and their labels

First, I verify that the Pods are running and check their labels.

`kubectl get pods --show-labels


I make sure the Pod labels match the Service selector.

2. Check the Service configuration

Then I check the Service details.

`kubectl get svc <service-name>`
`kubectl describe svc <service-name>`


I verify:

Selector labels

Port and targetPort

Service type

3. Check Service Endpoints (Very Important)

Next, I check whether the Service has endpoints.

`kubectl get endpoints <service-name>`


If endpoints are empty, it means the Service is not selecting any Pods.

This usually indicates a label mismatch.

4. Verify Pod readiness

I ensure the Pods are in Ready state.
Unready Pods will not receive traffic.

`kubectl get pods`

5. Test connectivity inside the cluster

I exec into a Pod and test the Service DNS or ClusterIP.

`kubectl exec -it <pod-name> -- curl http://<service-name>:<port>`


This helps confirm internal connectivity.

---

## Q32. How do you scale a Kubernetes deployment


If I need quick scaling, I manually change the replica count.

`kubectl scale deployment <deployment-name> --replicas=5`

--- 

## Q33. Difference between HPA and VPA in Kubernetes

HPA – Horizontal Pod Autoscaler

What it does:

HPA increases or decreases the number of Pods based on load.

How it scales:

Scales out / in

Adds more Pods when traffic increases

Removes Pods when traffic decreases

Metrics used:

CPU

Memory

Custom metrics (Prometheus)

Example:

If CPU usage goes above 70%, HPA creates more Pods.

Best use case:

Web applications

APIs

Microservices

Traffic-based workloads

VPA – Vertical Pod Autoscaler

What it does:

VPA increases or decreases the CPU and memory of a Pod.

How it scales:

Scales up / down

Changes resource requests and limits

May restart Pods to apply new values

Metrics used:

Historical CPU and memory usage

Example:

If a Pod consistently needs more memory, VPA increases its memory request.

Best use case:

Batch jobs

Background workers

Applications with unpredictable memory usage

---

## Q34. What is Cluster Autoscaler in Kubernetes?

Cluster Autoscaler automatically adds or removes worker nodes in a Kubernetes cluster based on pod scheduling needs.

Why Cluster Autoscaler is needed

Sometimes Pods cannot be scheduled because:

Not enough CPU or memory on existing nodes

All nodes are full

Cluster Autoscaler solves this by scaling the cluster itself, not the Pods.

How Cluster Autoscaler works
Scale Up

When a Pod is in Pending state due to insufficient resources:

Cluster Autoscaler detects it

Adds a new node

Pod gets scheduled on the new node

Scale Down

When nodes are underutilized:

Pods are moved to other nodes

Empty nodes are removed safely

What it works with

Cloud providers:

AWS (Auto Scaling Groups)

GCP

Azure

Works alongside:

HPA (scales Pods)

VPA (scales Pod resources)

---

## Q35. How does Kubernetes handle rolling updates and rollbacks

Kubernetes handles rolling updates and rollbacks using Deployments.
It gradually replaces old Pods with new ones while keeping the application available, and if something goes wrong, it can quickly roll back to the previous stable version.

How rolling update works

- During a rolling update:

- Kubernetes creates new Pods with the new version

- Old Pods are terminated gradually

- Traffic is always routed to healthy Pods

This avoids downtime.

How rollback works

Kubernetes stores Deployment revision history.
You can roll back to the last working version easily.

How Kubernetes detects failures

- Readiness probes

- Liveness probes

- Pod crash status

Unhealthy Pods don’t receive traffic.


---

## Q36. CrashLoopBackOff explanation and How do you debug CrashLoopBackOff?

CrashLoopBackOff means a container keeps starting, crashing, and restarting repeatedly, and Kubernetes is backing off before restarting it again.

Common reasons:

- Application error or bug

- Wrong command or entrypoint

- Missing environment variables

- Wrong config or secrets

- App trying to connect to a service that is not available

- Port already in use

- Permission issues

How do you debug CrashLoopBackOff?

---

## Q37. A Deployment update caused downtime. How do you prevent this?

Deployment downtime usually happens when old Pods are terminated before new Pods become ready. To prevent this, I make sure deployments are done using a Rolling Update strategy with proper health checks and configuration.

How I prevent downtime in Kubernetes deployments
Use Rolling Update strategy (not Recreate)

---
## Q38.  What is a Persistent Volume (PV) and Persistent Volume Claim (PVC)?

In Kubernetes, Pods are temporary, so any data stored inside a Pod is lost when the Pod restarts or is deleted. To store data permanently, Kubernetes provides Persistent Volumes (PV) and Persistent Volume Claims (PVC).

Persistent Volume (PV)

A Persistent Volume is a piece of storage in the cluster that is provisioned by the admin or dynamically by Kubernetes.

It represents real storage like:

- EBS

- NFS

- Azure Disk

It exists independently of Pods

Even if the Pod is deleted, the data remains

Persistent Volume Claim (PVC)

A Persistent Volume Claim is a request for storage made by the application or developer.

It defines:

Required storage size

Access mode (ReadWriteOnce, etc.)

Kubernetes automatically binds the PVC to a matching PV

Pods use PVCs, not PVs directly


---

## Q39. PVC stuck in Pending

Persistent Volume Claim (PVC) stuck in "Pending" status typically indicates Kubernetes cannot bind the PVC to a suitable Persistent Volume (PV).

Common Causes and Fixes
Missing or Incorrect StorageClass:
Ensure the PVC specifies a valid storageClassName that exists in your cluster:
`kubectl get storageclass`

If no suitable StorageClass exists, create one or update the PVC to use an existing one. For dynamic provisioning, the StorageClass must have a provisioner (e.g., kubernetes.io/aws-ebs, rancher.io/local-path). 
No Available PVs Matching PVC Requirements:
Check for existing PVs:
`kubectl get pv`

Ensure at least one PV is in Available state with matching storage, accessModes, and storageClassName. 
Storage Provisioner Issues:
If using dynamic provisioning, verify the provisioner pod is running and healthy:

```bash

kubectl get pods -n <provisioner-namespace>
kubectl logs -n <provisioner-namespace> <provisioner-pod-name>
```

Look for errors related to API calls, authentication, or backend storage. 
Resource Quotas or Limits:
Check for quota violations:
```bash
kubectl get resourcequotas
kubectl describe pvc <pvc-name>
```

Events may show exceeded quota errors (e.g., too many services, persistent volumes). 


---

## Q40 How do you set up autoscaling in Kubernetes

In Kubernetes, autoscaling is mainly handled using HPA (Horizontal Pod Autoscaler) for pods and Cluster Autoscaler for nodes. I usually set it up in layers.

1. Horizontal Pod Autoscaler (HPA)

HPA automatically increases or decreases the number of pod replicas based on metrics like CPU or memory.

Steps to set up HPA:

Install Metrics Server (required)

`kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml`


Set resource requests & limits in Deployment

```bash
resources:
  requests:
    cpu: "200m"
  limits:
    cpu: "500m"
```

Create HPA

```bash
kubectl autoscale deployment my-app \
  --cpu-percent=70 \
  --min=2 \
  --max=10
 ```


Now Kubernetes automatically scales pods based on CPU usage.

2. Cluster Autoscaler (Node Autoscaling)

HPA scales pods, but if nodes don’t have enough capacity, Cluster Autoscaler adds or removes worker nodes.

How it works:

If pods are Pending due to lack of resources, Cluster Autoscaler adds nodes

If nodes are underutilized, it removes them safely

In cloud environments like EKS, AKS, or GKE, Cluster Autoscaler is integrated with the cloud provider’s autoscaling groups.

3. Vertical Pod Autoscaler (Optional)

VPA adjusts CPU and memory requests for pods automatically.
Usually not used together with HPA on CPU.


---

## Q41. What is a Network Policy in Kubernetes?

A Network Policy in Kubernetes is used to control how pods communicate with each other and with external networks.
By default, all pods can talk to each other, but Network Policies allow us to restrict traffic for better security.

What it controls

Network Policy defines:

Ingress rules – which pods can receive traffic

Egress rules – which pods can send traffic

Traffic based on pod labels, namespaces, and ports


---

## Q42. How do you expose a Kubernetes application externally?

By default, Kubernetes pods are internal and cannot be accessed from outside the cluster.
To expose an application externally, Kubernetes provides multiple options depending on the use case.

1. NodePort Service

Exposes the application on a static port on each node

Accessed using NodeIP:NodePort

Mostly used for testing or small setups

👉 Not recommended for production

2. LoadBalancer Service

Creates a cloud-managed external load balancer

Assigns a public IP automatically

Commonly used in AWS, Azure, GCP

👉 Simple and reliable for production in cloud

3. Ingress Controller (Most Preferred)

Exposes multiple services using:

Single IP

Domain-based routing

Path-based routing

Supports SSL/TLS termination

Works with controllers like NGINX, ALB, Traefik

👉 Best practice for production environments

4. External DNS / Service Mesh (Advanced)

Used in large-scale systems

Provides:

Advanced traffic routing

Security

Observability

---

## Q43. How does Kubernetes handle node failures? 

Kubernetes handles node failures using its built-in self-healing mechanisms to ensure applications remain available.

1. Node Failure Detection

Each node sends a heartbeat to the control plane.

If the control plane doesn’t receive it within a timeout, the node is marked as NotReady.

2. Pod Rescheduling

Pods running on the failed node are marked as unavailable.

If the pods are part of a Deployment / ReplicaSet / StatefulSet, Kubernetes automatically reschedules new pods on healthy nodes.

3. Self-Healing with ReplicaSets

ReplicaSets ensure the desired number of pod replicas is always running.

If a pod is lost due to node failure, a new one is created automatically.

4. Persistent Volume Handling

If pods use Persistent Volumes, Kubernetes re-attaches the volume to the new pod (depending on storage type).

This ensures data is not lost.

5. Service Continuity

Kubernetes Services automatically stop sending traffic to failed pods.

Traffic is routed only to healthy pods, ensuring minimal disruption.

---

## Q44. What are Custom Resource Definitions (CRDs)

Custom Resource Definitions, or CRDs, allow us to extend Kubernetes by creating our own custom resources, just like built-in resources such as Pods, Services, or Deployments.

---

## Q45. What is a Memory Leak? How do you troubleshoot a memory leak in Kubernetes?

1. Check Pod memory usage

First, I check if the pod’s memory usage keeps increasing.

kubectl top pod <pod-name>


If memory keeps growing and never comes down, it’s a strong sign of a memory leak.

2. Check Pod events

Then I describe the pod to see if it was killed due to memory issues.

kubectl describe pod <pod-name>


If I see OOMKilled, it confirms the container exceeded its memory limit.

3. Check container logs

Next, I check application logs for errors or unusual behavior.

kubectl logs <pod-name>


This helps identify issues like infinite loops, unclosed connections, or cache problems.

4. Verify resource limits

I check whether proper memory requests and limits are set.

```bash
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"

```
Without limits, a leaking application can consume all node memory.

---

## Q46 . Your Kubernetes cluster is running slow. How do you optimize it?

1. Check resource usage

First, I check CPU and memory usage of nodes and pods.

kubectl top nodes
kubectl top pods


This helps me see if any pod or node is overloaded or underutilized.

2. Fix resource requests and limits

Many performance issues happen due to incorrect resource settings.

If requests are too high → pods don’t get scheduled

If limits are missing → pods may consume excessive resources

So I tune CPU and memory requests/limits properly.

3. Scale workloads

If the application is under heavy load:

I scale pods using HPA (Horizontal Pod Autoscaler)

If nodes are insufficient, I enable Cluster Autoscaler

This ensures enough capacity when traffic increases.

4. Remove unused resources

I clean up:

Unused pods

Old deployments

CrashLoopBackOff pods

Unused PVCs

This frees up cluster resources.

5. Check pod scheduling issues

I look for:

Pods in Pending state

Node taints & tolerations

Node affinity / anti-affinity rules

These can slow down scheduling.

6. Optimize networking

If apps communicate a lot:

I check Service and Ingress configuration

Look for NetworkPolicy issues

Ensure DNS and CNI are healthy

Network bottlenecks can make the cluster feel slow.


---

## Q47. Describe the role of Ingress Controllers and Ingress Resources in Kubernetes networking.

In Kubernetes, Ingress and Ingress Controllers are used to manage external HTTP/HTTPS traffic to services.

An Ingress resource defines routing rules, like mapping a domain name or URL path to a specific service. It is only a configuration.

An Ingress Controller is the actual component, such as NGINX or Traefik, that reads these rules and routes traffic accordingly.

Without an Ingress Controller, Ingress rules don’t work.
Together, they allow us to expose multiple services using a single IP, handle SSL termination, and manage traffic efficiently.

---

## Q48. What are Probes in Kubernetes?

Probes in Kubernetes are mechanisms used to monitor the health and readiness of containers running inside pods.  They enable Kubernetes to automatically detect and respond to unhealthy states, enhancing application reliability through self-healing capabilities. 

There are three primary types of probes:

Liveness Probe: Determines if a container is running. If it fails, Kubernetes restarts the container. 
Readiness Probe: Checks if a container is ready to serve traffic. If it fails, Kubernetes removes the container from service endpoints until it passes. 
Startup Probe: Ensures the application has started successfully before other probes (liveness and readiness) begin. This prevents premature checks during long startup phases. 

---

## Q49. What is the purpose of a Service Mesh in Kubernetes, and how does it work?

A Service Mesh is a dedicated infrastructure layer for managing service-to-service communication within a Kubernetes cluster.
It typically includes features like traffic routing, load balancing, encryption, and observability to enhance reliability and security.

How it works:
A Service Mesh works by deploying a sidecar proxy next to each application pod.
Instead of services talking directly to each other, all network traffic flows through these proxies.

Step by step:

- A request from Service A first goes to its sidecar proxy

- The proxy applies rules like security, retries, timeouts, and routing

- The request then goes to the sidecar proxy of Service B

- Service B receives the request after all policies are applied

- The control plane manages these proxies by pushing configuration, policies, and certificates.


---

## Q50. What is ResourceQuota in Kubernetes?

ResourceQuota in Kubernetes is used to limit how much CPU, memory, and other resources a namespace can consume.
It helps prevent one team or application from using all the cluster resources and ensures fair usage in multi-tenant environments.
When a namespace reaches its quota, Kubernetes rejects any new resource requests that exceed the defined limits.

```bash
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-namespace-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "6"
    limits.memory: 12Gi
    pods: "10"
    services: "5"
    persistentvolumeclaims: "4"
```

---

## Q51. How do you back up etcd in Kubernetes?

To back up etcd in Kubernetes, use the etcdctl command-line tool to create a snapshot of the cluster state. This is critical because etcd stores all cluster configuration, deployments, services, secrets, and object states. 

Prerequisites:
- Access to a control plane node.
- etcdctl installed and available.
- Required certificates: ca.crt, server.crt, and server.key from /etc/kubernetes/pki/etcd/.

Backup Command

```bash
export ETCDCTL_API=3
sudo etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db
```

This command saves a snapshot to a specified location (e.g., /opt/backup/).

Verify the Backup:

`etcdutl snapshot status /opt/backup/etcd-snapshot-*.db --write-out=table   `

---

## Q52. Difference between ResourceQuota vs LimitRange

ResourceQuota and LimitRange are both used to control resources in a Kubernetes namespace, but they serve different purposes.

ResourceQuota

ResourceQuota limits the total amount of resources a namespace can consume.

- Applies at namespace level

- Controls total CPU, memory, pods, services, PVCs, etc.

- Prevents one team from using all cluster resources

LimitRange

LimitRange sets default and maximum/minimum resource values for individual pods or containers.

- Applies at pod/container level

- Sets default requests and limits

- Ensures pods don’t request too much or too little resources

--- 
