# Kubeadm Installation Guide - Complete Setup for Kubernetes Cluster

A comprehensive step-by-step guide to set up a production-ready Kubernetes cluster using kubeadm on AWS EC2 instances running Ubuntu.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [AWS Setup & Security Group Configuration](#aws-setup--security-group-configuration)
3. [Common Setup for All Nodes](#common-setup-for-all-nodes)
4. [Master Node Setup](#master-node-setup)
5. [Worker Node Setup](#worker-node-setup)
6. [Cluster Verification](#cluster-verification)
7. [Troubleshooting](#troubleshooting)

---

## Prerequisites

Before beginning the Kubernetes cluster setup, ensure the following requirements are met:

**System Requirements:**
- Operating System: Ubuntu 20.04 LTS or later (Focal, Jammy, or Noble)
- Instance Type: AWS EC2 t2.medium or higher
- vCPU: Minimum 2 cores per instance
- RAM: Minimum 2GB per instance (4GB recommended)
- Storage: Minimum 20GB per instance
- Network: All instances must be able to communicate with each other

**AWS Requirements:**
- AWS account with EC2 access
- All instances in the same VPC
- Security group with proper port configurations
- Internet access for downloading packages

---

## AWS Setup & Security Group Configuration

### Step 1: Create or Identify a Security Group

**Method: AWS Management Console**

1. Log in to the **AWS Management Console**
2. Navigate to the **EC2 Dashboard**
3. In the left sidebar, find **Network & Security** section
4. Click on **Security Groups**

### Step 2: Create a New Security Group

1. Click the **Create Security Group** button
2. Fill in the following details:

| Field | Value |
|-------|-------|
| Security group name | `Kubernetes-Cluster-SG` |
| Description | Security group for Kubernetes cluster management |
| VPC | Select your default VPC or appropriate VPC |

### Step 3: Add Inbound Rules to Security Group

Add the following inbound rules to allow necessary traffic:

**Rule 1: SSH Access**
- **Type**: SSH
- **Protocol**: TCP
- **Port Range**: 22
- **Source**: 0.0.0.0/0 (or your specific IP for better security)
- **Description**: Allow SSH access for cluster management

**Rule 2: Kubernetes API Server**
- **Type**: Custom TCP
- **Protocol**: TCP
- **Port Range**: 6443
- **Source**: 0.0.0.0/0 (or security group itself)
- **Description**: Allow Kubernetes API traffic from worker nodes

**Rule 3: Allow Internal Communication**
- **Type**: All traffic
- **Source**: Select the security group itself (sg-xxxxxxxxx)
- **Description**: Allow all traffic within the security group

**Rule 4: Kubelet API**
- **Type**: Custom TCP
- **Port Range**: 10250
- **Source**: 0.0.0.0/0
- **Description**: Kubelet API port

**Rule 5: etcd Server Client API**
- **Type**: Custom TCP
- **Port Range**: 2379-2380
- **Source**: 0.0.0.0/0
- **Description**: etcd server communication

### Step 4: Create Instances with Security Group

When launching EC2 instances:

1. Proceed through the instance creation wizard
2. At the **Configure Security Group** step, select **Select an existing security group**
3. Choose **Kubernetes-Cluster-SG** from the dropdown
4. Complete the instance creation

**Recommended Instance Configuration:**
- **Master Node**: 1 instance (t2.medium)
- **Worker Nodes**: 2-3 instances (t2.medium or higher)

---

## Common Setup for All Nodes

Execute the following commands on **all instances** (master and worker nodes) to prepare them for Kubernetes installation.

### 1. Disable Swap Memory

Kubernetes requires swap to be disabled for proper functionality:

```bash
sudo swapoff -a
```

Verify swap is disabled:

```bash
free -h
```

The swap row should show 0 bytes.

### 2. Load Necessary Kernel Modules

Create and load required kernel modules for Kubernetes networking:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

Load the modules immediately:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Verify modules are loaded:

```bash
lsmod | grep br_netfilter
lsmod | grep overlay
```

Both should return output confirming the modules are loaded.

### 3. Configure Sysctl Parameters

Configure kernel parameters for Kubernetes networking:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

Apply the settings:

```bash
sudo sysctl --system
```

Verify the settings:

```bash
sudo sysctl -a | grep net.ipv4.ip_forward
sudo sysctl -a | grep net.bridge.bridge-nf-call-iptables
```

### 4. Install Containerd Container Runtime

Update system packages:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
```

Set up Docker GPG key:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add Docker repository:

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install containerd:

```bash
sudo apt-get update
sudo apt-get install -y containerd.io
```

Configure containerd with SystemdCgroup driver:

```bash
containerd config default | sed -e 's/SystemdCgroup = false/SystemdCgroup = true/' -e 's/sandbox_image = "registry.k8s.io\/pause:3.6"/sandbox_image = "registry.k8s.io\/pause:3.9"/' | sudo tee /etc/containerd/config.toml
```

Start and enable containerd service:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd
```

Verify containerd is running:

```bash
sudo ctr version
```

### 5. Install Kubernetes Components (kubeadm, kubelet, kubectl)

Update system packages:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

Add Kubernetes GPG key:

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add Kubernetes repository:

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Install Kubernetes components:

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
```

Prevent automatic updates of Kubernetes packages:

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

Enable kubelet service:

```bash
sudo systemctl enable kubelet
```

Verify installation:

```bash
kubeadm version
kubectl version --client
kubelet --version
```

---

## Master Node Setup

Execute the following commands **only on the master node** (control plane).

### 1. Initialize the Kubernetes Cluster

Initialize the cluster with kubeadm:

```bash
sudo kubeadm init
```

**Expected Output:**
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join <master-ip>:6443 --token <token> \
        --discovery-token-ca-cert-hash sha256:<hash>
```

**Save the join command** - you'll need it for worker nodes.

### 2. Set Up Local kubeconfig

Configure kubectl for the current user:

```bash
mkdir -p "$HOME"/.kube
sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config
```

Verify kubectl is configured correctly:

```bash
kubectl cluster-info
```

### 3. Install Network Plugin (Calico)

Deploy Calico network plugin for pod-to-pod communication:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml
```

Wait for Calico pods to be running:

```bash
kubectl get pods -n calico-system
```

All pods should be in "Running" state.

### 4. Generate Worker Node Join Command

Generate the token and join command for worker nodes:

```bash
kubeadm token create --print-join-command
```

**Expected Output:**
```
kubeadm join 10.0.1.100:6443 --token a1b2c3.d4e5f6g7h8i9j0k1 --discovery-token-ca-cert-hash sha256:abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890
```

Save this command for each worker node.

---

## Worker Node Setup

Execute the following commands on **each worker node**.

### 1. Pre-flight Checks

Reset kubeadm pre-flight checks:

```bash
sudo kubeadm reset pre-flight checks
```

### 2. Join the Cluster

Paste and execute the join command from the master node. The command format is:

```bash
sudo kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --cri-socket unix:///run/containerd/containerd.sock -v=5
```

**Replace the placeholders:**
- `<master-ip>`: Private IP address of the master node
- `<token>`: Token from the master node
- `<hash>`: Hash from the master node

**Example:**
```bash
sudo kubeadm join 10.0.1.100:6443 --token a1b2c3.d4e5f6g7h8i9j0k1 --discovery-token-ca-cert-hash sha256:abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890 --cri-socket unix:///run/containerd/containerd.sock -v=5
```

**Expected Output:**
```
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FlattenedLocalEtcdUpstreamToDefaultDown: Flattened a local etcd upstream configuration to the default upstream
[kubelet-start] Writing kubelet environment file with flags to file /var/lib/kubelet/kubeadm-flags.env
[kubelet-start] Writing kubelet configuration to file /var/lib/kubelet/config.yaml
[kubelet-start] Starting the kubelet
[kubeadm] Waiting for the kubelet to boot up the control plane as static pods from directory /etc/kubernetes/manifests.
[kubeadm] This might take a moment, because the kubelet is stopping the component-based kubelet from earlier in this configuration.
[kubeadm] The "kubelet" is now running.
[kubeadm] The control plane instance for this control plane is now running.

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a request was approved.
* The Kubelet has registered itself with the Master.
* The control plane has marked this node as a member of the cluster and assigned it a role.

Run 'kubectl get nodes' on the control-plane to see this node register. Give it a minute to fully register and appear as 'ready'.
```

### 3. Verify Node Joined Successfully

From the master node, verify all nodes have joined:

```bash
kubectl get nodes
```

All nodes should show "Ready" status after a minute or two.

---

## Cluster Verification

### Check Node Status

On the master node, verify all nodes are ready:

```bash
kubectl get nodes
```

**Expected Output:**
```
NAME               STATUS   ROLES           AGE     VERSION
master-node        Ready    control-plane   5m10s   v1.29.x
worker-node-1      Ready    <none>          2m35s   v1.29.x
worker-node-2      Ready    <none>          2m15s   v1.29.x
```

### Check Cluster Information

```bash
kubectl cluster-info
```

### Check All Pods

```bash
kubectl get pods --all-namespaces
```

### Check etcd Status

```bash
kubectl get pods -n kube-system | grep etcd
```

### Check Calico Network Plugin

```bash
kubectl get pods -n calico-system
```

All should be in "Running" state.

### Test Cluster Connectivity

Create a test pod:

```bash
kubectl run test-pod --image=nginx --port=80
```

Check pod status:

```bash
kubectl get pods
```

Delete test pod:

```bash
kubectl delete pod test-pod
```

---

## Troubleshooting

### Issue: Nodes stuck in "NotReady" state

**Check kubelet status:**
```bash
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 50
```

**Check containerd status:**
```bash
sudo systemctl status containerd
```

### Issue: Worker node join fails

**Verify network connectivity:**
```bash
ping <master-ip>
```

**Check if ports are open:**
```bash
telnet <master-ip> 6443
```

**Verify join command:**
- Ensure you copied the entire command correctly
- Ensure you have sudo privileges
- Check the token hasn't expired (tokens expire after 24 hours)

**Generate new token if expired:**
```bash
kubeadm token create --print-join-command
```

### Issue: Pods cannot communicate

**Check CNI plugin status:**
```bash
kubectl get pods -n calico-system
kubectl get daemonset -n calico-system
```

**Check network policies:**
```bash
kubectl get networkpolicies --all-namespaces
```

### Issue: Certificate expiration

**Check certificate expiration:**
```bash
sudo kubeadm certs check-expiration
```

**Renew certificates:**
```bash
sudo kubeadm certs renew all
```

### Issue: etcd issues

**Check etcd health:**
```bash
kubectl get pods -n kube-system etcd-$(hostname)
```

**View etcd logs:**
```bash
kubectl logs -n kube-system etcd-$(hostname)
```

---

## Essential Commands Reference

### Cluster Management

```bash
# Get cluster info
kubectl cluster-info

# Get nodes
kubectl get nodes

# Describe a node
kubectl describe node <node-name>

# Check cluster events
kubectl get events
```

### Pod Management

```bash
# Get all pods
kubectl get pods --all-namespaces

# Get pods in a namespace
kubectl get pods -n <namespace>

# Describe a pod
kubectl describe pod <pod-name> -n <namespace>

# View pod logs
kubectl logs <pod-name> -n <namespace>

# Execute command in pod
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash
```

### Node Management

```bash
# Drain a node for maintenance
kubectl drain <node-name> --ignore-daemonsets

# Uncordon a node
kubectl uncordon <node-name>

# Cordon a node (prevent new pods)
kubectl cordon <node-name>
```

### Kubeadm Commands

```bash
# Check certificate expiration
sudo kubeadm certs check-expiration

# Renew all certificates
sudo kubeadm certs renew all

# Reset kubeadm on a node
sudo kubeadm reset

# Generate new token
kubeadm token create --print-join-command
```

---

## Post-Installation Configuration

### Enable kubectl Autocompletion

```bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc
source ~/.bashrc
```

### Configure kubectl Alias

```bash
echo 'alias k=kubectl' >> ~/.bashrc
source ~/.bashrc
```

### View kubeconfig

```bash
kubectl config view
```

### Switch Cluster Context

```bash
kubectl config use-context <context-name>
```

---

## Security Best Practices

1. **Restrict Security Group Access**: Instead of 0.0.0.0/0, use specific IP ranges
2. **Use Strong Credentials**: Protect kubeconfig files carefully
3. **Enable RBAC**: Implement Role-Based Access Control
4. **Network Policies**: Define network policies for pod communication
5. **Regular Updates**: Keep Kubernetes and components updated
6. **Monitoring**: Implement cluster monitoring and logging
7. **Secrets Management**: Use Kubernetes Secrets for sensitive data

---

## Next Steps

After successful cluster setup:

1. **Deploy Applications**: Use kubectl to deploy containerized applications
2. **Configure Storage**: Set up Persistent Volumes for data persistence
3. **Set up Ingress**: Configure ingress for external traffic routing
4. **Implement Monitoring**: Deploy Prometheus and Grafana for monitoring
5. **Configure Logging**: Set up centralized logging with ELK stack
6. **Security Hardening**: Implement network policies and RBAC
7. **Backup Strategy**: Plan and implement etcd backups

---

## Useful Resources

- [Kubernetes Official Documentation](https://kubernetes.io/docs/)
- [kubeadm Official Documentation](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)
- [Calico Documentation](https://docs.tigera.io/calico/latest/)
- [containerd Documentation](https://containerd.io/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

---

## Troubleshooting Checklist

- ✅ All instances have swap disabled
- ✅ Kernel modules (overlay, br_netfilter) are loaded
- ✅ Sysctl parameters are configured
- ✅ containerd is installed and running
- ✅ All Kubernetes components installed correctly
- ✅ Master node initialized successfully
- ✅ Network plugin (Calico) deployed
- ✅ All nodes showing "Ready" status
- ✅ All system pods are "Running"
- ✅ Security group rules are configured correctly
- ✅ Network connectivity verified between nodes

---

## Summary

This guide provides a complete setup for a production-ready Kubernetes cluster using kubeadm. By following these steps carefully, you'll have:

- A functional Kubernetes cluster with 1 master and 2+ worker nodes
- Container runtime (containerd) properly configured
- Network plugin (Calico) for pod communication
- All necessary components installed and configured
- A secure and scalable foundation for container orchestration

---

**Author**: DevOps Engineer  
**Last Updated**: October 2025  
**Kubernetes Version**: v1.29  
**Container Runtime**: containerd  
**Network Plugin**: Calico