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

## Q15. 
