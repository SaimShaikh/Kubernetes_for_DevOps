# Kubernetes Cluster Setup using Minikube on AWS EC2

This comprehensive guide walks you through setting up a single-node Kubernetes cluster using Minikube on an AWS EC2 instance running Ubuntu.

## What is Minikube?

**Minikube** is a lightweight Kubernetes implementation that creates a VM or container on your local machine and deploys a simple cluster containing only one node. It's perfect for learning Kubernetes, development, and testing purposes without the complexity of a multi-node production cluster.

## Prerequisites

- **AWS EC2 Instance**: t2.medium or higher (minimum 2 CPUs, 4GB RAM)
- **Operating System**: Ubuntu 20.04 LTS or newer (Ubuntu 22.04/24.04 recommended)
- **Required Tools**: Docker, kubectl, Minikube
- **Storage**: Minimum 20GB of free disk space
- **Access**: SSH access to the EC2 instance with sudo privileges
- **Network**: Internet connection for downloading packages

## Minikube vs KIND Comparison

| Feature | Minikube | KIND |
|---------|----------|------|
| **Primary Use Case** | Local development, learning Kubernetes | CI/CD pipelines, testing |
| **Deployment Method** | VM or Container | Container-based only |
| **Startup Time** | Slower (especially VM-based) | Faster |
| **Resource Usage** | Higher (2GB+ RAM) | Lower, more lightweight |
| **Multi-node Support** | Limited | Excellent |
| **Built-in Addons** | Extensive (dashboard, ingress, metrics) | Manual installation required |
| **Production-like** | More comprehensive | Minimal |
| **Best For** | Beginners, full-featured testing | Automated testing, multi-node scenarios |

---

## Step 1: Launch AWS EC2 Instance

Create an EC2 instance with the following specifications:

**Instance Configuration:**
- **AMI**: Ubuntu Server 20.04 LTS (HVM), 64-bit (x86) or Ubuntu 22.04/24.04
- **Instance Type**: t2.medium (2 vCPUs, 4GB RAM minimum)
- **Storage**: 20GB or more
- **Security Group**: 
  - Allow SSH (port 22) from your IP
  - Allow All Traffic (or specific ports as needed) for testing

**Create or Select a Key Pair** for SSH access and launch the instance.

---

## Step 2: Connect to Your EC2 Instance

SSH into your EC2 instance:

```bash
ssh -i your-key.pem ubuntu@your-ec2-public-ip
```

Update the system packages:

```bash
sudo apt update
sudo apt upgrade -y
```

---

## Step 3: Install Docker

Docker is required as Minikube's default driver on Linux.

Install Docker:

```bash
sudo apt-get update
sudo apt-get install docker.io -y
```

Start and enable Docker service:

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

Add your user to the docker group to run Docker without sudo:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Verify Docker installation:

```bash
docker --version
docker ps
```

**Note:** If the `newgrp` command doesn't work immediately, you may need to log out and log back in, or reboot the instance:

```bash
sudo reboot
```

---

## Step 4: Install kubectl

kubectl is the Kubernetes command-line tool for interacting with Kubernetes clusters.

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

Verify kubectl installation:

```bash
kubectl version --client
```

**Expected Output:**
```
Client Version: v1.XX.X
```

---

## Step 5: Install Minikube

Download the latest Minikube binary:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
```

Make the binary executable and move it to your PATH:

```bash
chmod +x minikube-linux-amd64
sudo mv minikube-linux-amd64 /usr/local/bin/minikube
```

**Alternative Installation Methods:**

**Using Debian Package:**
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
```

**Using install command:**
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

Verify Minikube installation:

```bash
minikube version
```

**Expected Output:**
```
minikube version: v1.XX.X
commit: xxxxxxxx
```

---

## Step 6: Start Minikube Cluster

Start Minikube with the Docker driver:

```bash
minikube start --driver=docker
```

**For EC2 instances, you can specify additional resources:**

```bash
minikube start --driver=docker --cpus=2 --memory=4096
```

**Alternative Drivers:**

If you prefer to use the none driver (runs Kubernetes directly on the host without virtualization):

```bash
sudo minikube start --driver=none
```

**Note:** The `none` driver requires root privileges and is recommended for CI/CD environments.

**Expected Output:**
```
😄  minikube v1.XX.X on Ubuntu 22.04
✨  Using the docker driver based on user configuration
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
🔥  Creating docker container (CPUs=2, Memory=4096MB) ...
🐳  Preparing Kubernetes v1.XX.X on Docker XX.XX.X ...
🔎  Verifying Kubernetes components...
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster
```

---

## Step 7: Verify Cluster Status

Check Minikube status:

```bash
minikube status
```

**Expected Output:**
```
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

View cluster information:

```bash
kubectl cluster-info
```

Check nodes:

```bash
kubectl get nodes
```

**Expected Output:**
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   2m    v1.XX.X
```

Check all pods in the cluster:

```bash
kubectl get pods --all-namespaces
```

---

## Step 8: Access Kubernetes Dashboard

Minikube comes with a built-in Kubernetes dashboard addon.

Enable the dashboard addon:

```bash
minikube addons enable dashboard
```

Start the dashboard:

```bash
minikube dashboard
```

**For EC2 instances**, since you're accessing remotely, use the following to get the dashboard URL:

```bash
minikube dashboard --url
```

This will return a URL like `http://127.0.0.1:XXXXX/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/`

**To access remotely via SSH tunneling:**

On your local machine, create an SSH tunnel:

```bash
ssh -i your-key.pem -L 8001:localhost:XXXXX ubuntu@your-ec2-public-ip
```

Replace `XXXXX` with the port number from the dashboard URL. Then access `http://localhost:8001` in your browser.

---

## Step 9: Enable Useful Addons

Minikube provides several built-in addons that extend functionality.

### List All Available Addons

```bash
minikube addons list
```

### Enable Metrics Server

For monitoring cluster resource usage:

```bash
minikube addons enable metrics-server
```

Verify metrics are available:

```bash
kubectl top nodes
kubectl top pods --all-namespaces
```

### Enable Ingress Controller

For routing external traffic to services:

```bash
minikube addons enable ingress
```

Verify ingress controller is running:

```bash
kubectl get pods -n ingress-nginx
```

### Enable Registry

For pushing and pulling Docker images locally:

```bash
minikube addons enable registry
```

### View Enabled Addons

```bash
minikube addons list | grep enabled
```

---

## Step 10: Deploy Your First Application

Create a simple nginx deployment:

```bash
kubectl create deployment nginx --image=nginx
```

Expose the deployment as a service:

```bash
kubectl expose deployment nginx --type=NodePort --port=80
```

Get the service URL:

```bash
minikube service nginx --url
```

Access the application:

```bash
curl $(minikube service nginx --url)
```

**For EC2 access**, get the NodePort:

```bash
kubectl get service nginx
```

Note the NodePort (e.g., 30000-32767 range), then access:

```bash
curl $(minikube ip):NODE_PORT
```

---

## Useful Minikube Commands

### Cluster Management

```bash
# Pause Kubernetes without stopping containers
minikube pause

# Unpause the cluster
minikube unpause

# Stop the cluster
minikube stop

# Restart the cluster
minikube start

# Delete the cluster
minikube delete

# Delete all Minikube clusters and profiles
minikube delete --all
```

### Configuration

```bash
# Set default memory limit
minikube config set memory 6144

# Set default CPUs
minikube config set cpus 4

# Set default driver
minikube config set driver docker

# View current configuration
minikube config view
```

### Multiple Clusters (Profiles)

```bash
# Create a second cluster with a different profile
minikube start -p cluster2 --kubernetes-version=v1.30.0

# List all profiles
minikube profile list

# Switch active profile
minikube profile cluster2

# View current profile
minikube profile
```

### SSH into Minikube

```bash
# SSH into the Minikube VM/container
minikube ssh

# Run a command inside Minikube
minikube ssh "docker ps"
```

### Services

```bash
# List all services
minikube service list

# Open a service in browser
minikube service <service-name>

# Get service URL
minikube service <service-name> --url
```

### Logs and Debugging

```bash
# View Minikube logs
minikube logs

# View logs for specific time period
minikube logs --length=100

# Check Minikube IP
minikube ip

# View Kubernetes version
minikube kubectl version
```

---

## Working with kubectl

### Set kubectl Alias

For easier use with Minikube's kubectl:

```bash
alias kubectl="minikube kubectl --"
```

Add to `.bashrc` or `.zshrc` to make permanent:

```bash
echo 'alias kubectl="minikube kubectl --"' >> ~/.bashrc
source ~/.bashrc
```

### Enable kubectl Autocompletion

```bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc
source ~/.bashrc
```

---

## Minikube Drivers Explained

Minikube supports various drivers depending on your environment:

| Driver | Description | Use Case |
|--------|-------------|----------|
| **docker** | Runs Kubernetes in Docker containers | Linux/Mac/Windows - Default, most compatible |
| **none** | Runs directly on host OS | CI/CD, cloud VMs (requires root) |
| **virtualbox** | Uses VirtualBox VM | Cross-platform, stable but slower |
| **kvm2** | Uses KVM hypervisor | Linux with KVM support |
| **hyperv** | Uses Hyper-V | Windows Pro/Enterprise |
| **podman** | Uses Podman containers | Rootless container alternative |

**Specify driver explicitly:**

```bash
minikube start --driver=docker
```

---

## Troubleshooting

### Issue: Docker permission denied

**Solution:** Ensure user is in docker group and reboot:
```bash
sudo usermod -aG docker $USER
sudo reboot
```

### Issue: Minikube won't start

**Solution:** Check Docker is running and resources are sufficient:
```bash
sudo systemctl status docker
docker info
minikube delete
minikube start --driver=docker
```

### Issue: Cannot access dashboard remotely

**Solution:** Use SSH port forwarding as described in Step 8, or use kubectl proxy:
```bash
kubectl proxy --address='0.0.0.0' --accept-hosts='.*'
```

Then access: `http://your-ec2-ip:8001/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/`

**Security Warning:** Only do this for testing. For production, use proper authentication.

### Issue: Pods are not starting

**Solution:** Check pod status and logs:
```bash
kubectl get pods --all-namespaces
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
```

### Issue: Minikube runs out of resources

**Solution:** Increase allocated resources:
```bash
minikube stop
minikube delete
minikube start --cpus=4 --memory=8192 --disk-size=30g
```

### Issue: Connection refused to Minikube IP

**Solution:** For EC2 instances, ensure security group allows traffic:
```bash
# Get Minikube IP
minikube ip

# Test connectivity
ping $(minikube ip)

# Check security group allows traffic on NodePort range (30000-32767)
```

---

## Best Practices

1. **Resource Allocation**: Always allocate sufficient resources based on your workload
   ```bash
   minikube start --cpus=4 --memory=8192
   ```

2. **Use Profiles**: Create separate profiles for different projects
   ```bash
   minikube start -p project-dev
   minikube start -p project-test
   ```

3. **Regular Cleanup**: Delete unused clusters to free resources
   ```bash
   minikube delete --all
   ```

4. **Enable Addons**: Use built-in addons instead of manual installations
   ```bash
   minikube addons enable metrics-server ingress dashboard
   ```

5. **Version Control**: Specify Kubernetes version for consistency
   ```bash
   minikube start --kubernetes-version=v1.30.0
   ```

6. **Save Configuration**: Export cluster configuration for backup
   ```bash
   kubectl config view --raw > kubeconfig-backup.yaml
   ```

---

## Common Minikube Addons

| Addon | Description | Enable Command |
|-------|-------------|----------------|
| dashboard | Kubernetes Dashboard UI | `minikube addons enable dashboard` |
| metrics-server | Resource metrics collection | `minikube addons enable metrics-server` |
| ingress | NGINX Ingress controller | `minikube addons enable ingress` |
| registry | Local Docker registry | `minikube addons enable registry` |
| storage-provisioner | Dynamic volume provisioning | Enabled by default |
| default-storageclass | Default storage class | Enabled by default |
| ingress-dns | DNS resolution for ingress | `minikube addons enable ingress-dns` |
| metallb | LoadBalancer implementation | `minikube addons enable metallb` |

---

## Next Steps

Now that your Minikube cluster is running, you can:

1. **Deploy Applications**: Practice deploying various workloads (Deployments, StatefulSets, DaemonSets)
2. **Work with Services**: Experiment with ClusterIP, NodePort, LoadBalancer service types
3. **Configure Ingress**: Set up ingress rules for routing external traffic
4. **Explore Helm**: Install Helm and deploy packaged applications
5. **Practice kubectl Commands**: Master Kubernetes command-line operations
6. **Learn about ConfigMaps and Secrets**: Manage application configuration
7. **Set up Persistent Storage**: Work with PersistentVolumes and PersistentVolumeClaims
8. **Implement RBAC**: Configure Role-Based Access Control
9. **Monitor with Prometheus**: Install monitoring stack
10. **CI/CD Integration**: Integrate with Jenkins, GitLab CI, or GitHub Actions

---



## Security Considerations

- **EC2 Security Groups**: Restrict SSH access to your IP only
- **Kubernetes RBAC**: Implement Role-Based Access Control for production-like environments
- **Network Policies**: Define network policies to restrict pod-to-pod communication
- **Secrets Management**: Use Kubernetes Secrets for sensitive data, consider external secret managers
- **Image Security**: Scan container images for vulnerabilities before deployment
- **Dashboard Access**: Never expose Kubernetes dashboard publicly without authentication
- **Regular Updates**: Keep Minikube, kubectl, and Kubernetes versions updated

---


**Author**: Saime Shaikh  
**Last Updated**: October 2025  
**Minikube Version**: Latest  
**Kubernetes Version**: v1.30+
