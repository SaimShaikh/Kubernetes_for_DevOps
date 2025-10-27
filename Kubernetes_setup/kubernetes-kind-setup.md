# Kubernetes Cluster Setup using KIND on AWS EC2

This guide walks you through setting up a multi-node Kubernetes cluster using KIND (Kubernetes IN Docker) on an AWS EC2 instance.

## Prerequisites

- **AWS EC2 Instance**: t2.medium or higher
- **Operating System**: Ubuntu (20.04 LTS or newer recommended)
- **Required Tools**: Docker, kubectl, KIND
- **Access**: SSH access to the EC2 instance with sudo privileges

## Architecture

This setup creates a Kubernetes cluster with:
- 1 Control Plane Node
- 2 Worker Nodes

All nodes run as Docker containers managed by KIND.

---

## Step 1: Install Docker

Install Docker on your Ubuntu EC2 instance:

```bash
sudo apt-get update
sudo apt-get install docker.io -y
```

Add your user to the docker group to run Docker commands without sudo:

```bash
sudo usermod -aG docker $USER && newgrp docker
```

Reboot the system to apply changes:

```bash
sudo reboot
```

After reboot, verify Docker installation:

```bash
docker --version
docker ps
```

---

## Step 2: Install kubectl

kubectl is the Kubernetes command-line tool that allows you to run commands against Kubernetes clusters.

Download the latest stable version:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

Make the kubectl binary executable:

```bash
chmod +x kubectl
```

Move kubectl to your PATH:

```bash
sudo mv kubectl /usr/local/bin/
```

Verify the installation:

```bash
kubectl version --client
```

---

## Step 3: Install KIND

KIND (Kubernetes IN Docker) is a tool for running local Kubernetes clusters using Docker containers as nodes.

Download KIND v0.22.0:

```bash
curl -Lo ./kind "https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64"
```

Make the KIND binary executable:

```bash
chmod +x kind
```

Move KIND to your PATH:

```bash
sudo mv kind /usr/local/bin/
```

Verify KIND installation:

```bash
kind version
```

---

## Step 4: Create Kubernetes Cluster Configuration

Create a directory for Kubernetes files:

```bash
mkdir Kubernetes_file
cd Kubernetes_file
```

Create a configuration file for your cluster:

```bash
vim config.yml
```

Add the following configuration to `config.yml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.32.1
- role: worker
  image: kindest/node:v1.32.1
- role: worker
  image: kindest/node:v1.32.1
```

**Configuration Explanation:**
- **kind: Cluster** - Defines this as a KIND cluster configuration
- **apiVersion** - Specifies the KIND API version
- **nodes** - List of nodes in the cluster
  - **role: control-plane** - Creates a master/control plane node
  - **role: worker** - Creates worker nodes (2 in this configuration)
  - **image** - Specifies the Kubernetes version (v1.32.1)

---

## Step 5: Create the Kubernetes Cluster

Create the cluster using the configuration file:

```bash
kind create cluster --config config.yml --name my-cluster
```

Replace `my-cluster` with any name you prefer for your cluster.

**Expected Output:**
```
Creating cluster "my-cluster" ...
 ✓ Ensuring node image (kindest/node:v1.32.1) 🖼
 ✓ Preparing nodes 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-my-cluster"
You can now use your cluster with:

kubectl cluster-info --context kind-my-cluster
```

---

## Step 6: Verify Cluster Status

Check that all nodes are running and ready:

```bash
kubectl get nodes
```

**Expected Output:**
```
NAME                       STATUS   ROLES           AGE   VERSION
my-cluster-control-plane   Ready    control-plane   2m    v1.32.1
my-cluster-worker          Ready    <none>          2m    v1.32.1
my-cluster-worker2         Ready    <none>          2m    v1.32.1
```

View cluster information:

```bash
kubectl cluster-info --context kind-my-cluster
```

Check all pods in the cluster:

```bash
kubectl get pods --all-namespaces
```

---

## Useful Commands

### Cluster Management

List all KIND clusters:
```bash
kind get clusters
```

Get cluster details:
```bash
kubectl cluster-info
```

Delete the cluster:
```bash
kind delete cluster --name my-cluster
```

### Node Management

View node details:
```bash
kubectl describe nodes
```

Check node resource usage:
```bash
kubectl top nodes
```

### Context Management

View current context:
```bash
kubectl config current-context
```

Switch between contexts:
```bash
kubectl config use-context kind-my-cluster
```

---

## Troubleshooting

### Issue: Docker permission denied
**Solution:** Ensure your user is added to the docker group and reboot:
```bash
sudo usermod -aG docker $USER
sudo reboot
```

### Issue: Nodes not ready
**Solution:** Wait a few minutes for all pods to start, then check pod status:
```bash
kubectl get pods -n kube-system
```

### Issue: KIND cluster creation fails
**Solution:** Check Docker is running and you have sufficient resources:
```bash
sudo systemctl status docker
docker ps
```

### Issue: Cannot connect to cluster
**Solution:** Verify kubectl context is set correctly:
```bash
kubectl config get-contexts
kubectl config use-context kind-my-cluster
```

---

## Next Steps

Now that your Kubernetes cluster is running, you can:

1. Deploy applications using `kubectl apply -f deployment.yaml`
2. Create services to expose your applications
3. Set up Ingress controllers for external access
4. Practice with Kubernetes manifests and workloads
5. Explore Helm charts for package management

---

## Additional Resources

- [KIND Official Documentation](https://kind.sigs.k8s.io/)
- [Kubernetes Official Documentation](https://kubernetes.io/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

---

## System Requirements

**Minimum EC2 Instance Specifications:**
- **Instance Type**: t2.medium (2 vCPUs, 4 GB RAM)
- **Storage**: 20 GB minimum
- **OS**: Ubuntu 20.04 LTS or newer

**For Production Workloads:**
- **Instance Type**: t2.large or higher recommended
- **Storage**: 50 GB or more

---

## Security Considerations

- Ensure your EC2 security group allows necessary traffic
- Keep Docker and Kubernetes versions updated
- Use IAM roles for AWS resource access
- Implement network policies for pod-to-pod communication
- Regular cluster backups and disaster recovery planning

---

## License

This guide is provided as-is for educational purposes.

## Contributing

Feel free to submit issues or pull requests to improve this guide.

---

**Author**: DevOps Engineer  
**Last Updated**: October 2025  
**Kubernetes Version**: v1.32.1  
**KIND Version**: v0.22.0