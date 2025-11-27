<img width="1676" height="769" alt="image" src="https://github.com/user-attachments/assets/cd557535-cc17-40f6-aa33-f639a11243b0" />



---


# 🧠 Kubernetes Architecture Explained in Depth

This repository provides a **complete breakdown of Kubernetes architecture**, including the **Control Plane**, **Worker Node**, and **Networking components**, along with **roles and responsibilities** for each element.

---

## ⚙️ Control Plane Components

The **Control Plane** is the brain of Kubernetes. It makes global decisions about the cluster (like scheduling) and detects/responds to cluster events (like scaling).

### 1️⃣ API Server

**Role:** The front-end of the Kubernetes Control Plane.
**Responsibilities:**

* Acts as the **central communication hub** for all Kubernetes components.
* Validates and processes **REST requests** (kubectl, client SDKs, or components).
* Exposes the Kubernetes API.
* Stores the cluster state in `etcd`.

📍 **Key Command:**

```bash
kubectl get --raw /api
```

---

### 2️⃣ etcd

**Role:** The distributed **key-value store** for all cluster data.
**Responsibilities:**

* Stores cluster configuration and state persistently.
* Provides **consensus** across all nodes using the **Raft protocol**.
* Acts as the **single source of truth** for Kubernetes objects.

📍 **Stored Data Examples:**

* Pods, Deployments, ConfigMaps, Secrets, and Node info.

---

### 3️⃣ Scheduler

**Role:** Assigns newly created Pods to available Worker Nodes.
**Responsibilities:**

* Watches for **unscheduled Pods**.
* Selects the best node based on **resource availability**, **affinity rules**, and **taints/tolerations**.
* Optimizes load across the cluster.

📍 **Example Criteria:**

* CPU & memory usage
* Node labels
* Pod affinity/anti-affinity
* NodeSelector constraints

---

### 4️⃣ Controller Manager

**Role:** Ensures the desired cluster state is always maintained.
**Responsibilities:**

* Runs background **controllers** (loops) that regulate the system.
* Common controllers:

  * Node Controller – monitors node health.
  * ReplicaSet Controller – maintains Pod replicas.
  * Endpoint Controller – maintains Service endpoints.
  * Job Controller – manages batch jobs.

📍 **Concept:**
If something goes wrong, controllers act automatically to bring the system back to the desired state.

📍 **Node Controller Configuration Parameters:**

* **Node Monitor Period:** 5 seconds (default) — the frequency at which the node controller checks node status.
* **Node Monitor Grace Period:** 40 seconds (default) — the grace period after which a node is considered unhealthy if no status is received.
* **Pod Eviction Timeout:** 5 minutes (default) — the time after which Pods on an unreachable node are deleted and recreated on a healthy node.

---

### 5️⃣ Kubectl

**Role:** The command-line tool for interacting with the Kubernetes API.
**Responsibilities:**

* Acts as the **primary interface** for users and DevOps engineers to manage Kubernetes clusters.
* Sends commands to the **API Server**, which then performs the requested operations.
* Supports CRUD operations for all Kubernetes objects (Pods, Deployments, Services, etc.).
* Provides debugging, scaling, and monitoring capabilities.

📍 **Common Commands:**

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl apply -f deployment.yaml
kubectl delete -f deployment.yaml
```

📍 **Usage Example:**

```bash
kubectl get nodes -o wide
kubectl get deployments -n kube-system
```

---

## 🧩 Worker Node Components

Each **Worker Node** runs actual workloads (containers) and communicates with the Control Plane.

### 1️⃣ Kubelet

**Role:** Kubelet manages the containers that are
scheduled to run on that node. It ensures that
the containers are running and healthy, and that
the resources they need are available.
Kubelet communicates with the Kubernetes API
server to get information about the containers
that should be running on the node, and then
starts and stops the containers as needed to
maintain the desired state. It also monitors the
containers to ensure that they are running
correctly, and restarts them if necessary.
**Responsibilities:**

* Registers the node with the cluster.
* Ensures that Pods are running as defined in PodSpecs.
* Reports node and Pod status to the API Server.
* Restarts failed Pods via the container runtime.

📍 **Command Example:**

```bash
systemctl status kubelet
```

---

### 2️⃣ Kube-Proxy (Service Proxy)

**Role:** Manages network rules to allow communication inside/outside the cluster.
**Responsibilities:**

* Maintains **network routing** and **load balancing** for Services.
* Handles cluster-wide IP routing (ClusterIP, NodePort, LoadBalancer).
* Works at both **L4 (TCP/UDP)** and **L7 (HTTP)** levels.
* Kube-proxy works by maintaining a set of network rules on each node in the cluster, which are updated dynamically as services are added or removed. When a client sends a request to a service, the request is intercepted by kube-proxy on the node where it was received. Kube-proxy then looks up the destination endpoint for the service and routes the request accordingly. Kube-proxy is an essential component of a Kubernetes cluster, as it ensures that
services can communicate with each other.

📍 **Modes:**

* `iptables` mode
* `ipvs` mode

---

### 3️⃣ Container Runtime

**Role:** Runs containers inside Pods.
**Responsibilities:**

* Pulls container images.
* Starts and stops containers.
* Reports container status to Kubelet.
* Common runtimes: **containerd**, **CRI-O** (Docker used to be, now deprecated).

📍 **Command Example:**

```bash
crictl ps
```

---

### 4️⃣ Pod

**Role:** The smallest deployable unit in Kubernetes.
**Responsibilities:**

* Encapsulates one or more containers.
* Shares the same network namespace (localhost) and storage volumes.
* Represents a single instance of an application.

📍 **Command Example:**

```bash
kubectl get pods -o wide
```

---

## 🌐 Networking Component

### CNI (Container Network Interface)

**Role:** Connects containers and nodes within the cluster.
**Responsibilities:**

* Assigns IPs to Pods.
* Enables inter-Pod communication.
* Manages network policies.
* Integrates with plugins like Calico, Flannel, or WeaveNet.

📍 **Command Example:**

```bash
kubectl get pods -n kube-system | grep cni
```

---

## 🧭 Summary Table

| Component          | Layer              | Role           | Key Function                     |
| ------------------ | ------------------ | -------------- | -------------------------------- |
| API Server         | Control Plane      | Entry point    | Manages API requests             |
| etcd               | Control Plane      | Database       | Stores cluster state             |
| Scheduler          | Control Plane      | Decision maker | Assigns Pods to nodes            |
| Controller Manager | Control Plane      | Enforcer       | Maintains desired state          |
| Kubectl            | Control Plane Tool | CLI Interface  | Sends commands to API Server     |
| Kubelet            | Worker Node        | Agent          | Runs and reports Pods            |
| Kube-Proxy         | Worker Node        | Network        | Routes and load balances traffic |
| Container Runtime  | Worker Node        | Executor       | Runs container images            |
| Pod                | Worker Node        | Workload       | Smallest deployable unit         |
| CNI                | Network            | Connector      | Manages Pod networking           |

---




## ✨ Author

**Saime Shaikh**
DevOps Enthusiast | AWS | Kubernetes | Docker | Terraform | Jenkins | ArgoCD




