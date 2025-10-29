<img width="1754" height="429" alt="image" src="https://github.com/user-attachments/assets/4a8f173c-1e49-4711-b6e5-6fee2b682d8c" />

---

# Kubernetes Workflow: Behind the Scenes

This document explains the steps that occur when a user interacts with a Kubernetes cluster using `kubectl`.

---

## 1. User with Kubectl

- The process begins when a user sends a request to the Kubernetes cluster using the `kubectl` command-line tool.
- Example: `kubectl apply -f pod.yaml` creates a Pod in the cluster.

## 2. Kubernetes API Server

- The request first reaches the **Kubernetes API Server**, the central hub of the Kubernetes control plane.
- All interactions—whether from users or other internal components—pass through the API Server.

## 3. Authenticate

- The API Server checks your identity (authentication).
- Authentication methods include certificates, bearer tokens, or cloud provider IAM roles.

## 4. Authorize

- The server then checks what this user is allowed to do (authorization).
- Authorization policies (like RBAC or ABAC) determine if the user can perform the requested action.

## 5. Validate

- The server validates your request for correct syntax and required fields.
- If there are errors, the request is rejected and feedback is sent to the user.

## 6. etcd: Persistent Storage

- Validated resource definitions are stored in **etcd**, a distributed key-value store.
- etcd maintains the cluster state, acting as Kubernetes' source of truth.

## 7. Scheduler

- The **Scheduler** selects the most appropriate node on which to run the new Pod or workload.
- Scheduling decisions consider resource availability, policies, and workload requirements.

## 8. Pod Creation

- The selected node (via the Kubelet agent) launches the defined Pod.
- The Pod is now active and managed according to the user's original request.

---

## Workflow Summary Table

| Step               | Description                                        |
|--------------------|---------------------------------------------------|
| User with kubectl  | Sends request to Kubernetes cluster                |
| API Server         | Entry point for all cluster requests               |
| Authenticate       | Verifies user identity                             |
| Authorize          | Checks user permissions                            |
| Validate           | Ensures correctness of the request                 |
| etcd               | Stores cluster configuration and state             |
| Scheduler          | Matches Pod to suitable node                       |
| Pod                | Container(s) created on chosen node                |

---

*Made by Saime Shaikh - Visual: Kubernetes workflow behind the scenes*
