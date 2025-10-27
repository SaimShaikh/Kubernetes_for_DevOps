# 📚 Kubernetes CLI Cheatsheet (All-in-One)



---



## kubectl — Global Syntax & Power Flags

| Command/Flag                     | Description                          |
|---------------------------------|------------------------------------|
| `kubectl <verb> <resource> [name]` | Basic command form                |
| `-n <ns> / --namespace <ns>`    | Target a specific namespace          |
| `-o wide`                       | Show extended information            |
| `-l key=value`                  | Label selector                      |
| `--field-selector key=value`    | Field selector                      |
| `-w / --watch`                  | Stream resource changes              |
| `--dry-run=client/server`        | Preview changes without applying    |
| `-f <file.yaml> / -k <dir>`     | Apply manifest files or kustomize   |
| `--context <ctx>`               | Use specific kubeconfig context      |
| `--as <user> --as-group <grp>`  | Impersonate user/group (RBAC test)  |
| `--server-side`                 | Use server-side apply (SSA)          |
| `--sort-by=<field>`             | Sort output by field                  |

---

## API Discovery & Resources

| Command                 | Purpose                                  |
|-------------------------|------------------------------------------|
| `kubectl api-resources` | List all available API resources         |
| `kubectl api-versions`  | Show supported API versions               |
| `kubectl explain <resource>` | Show fields/details of a resource     |

---

## Resource Kinds & Short Names

| Kind        | Short Name | Scope    | Description                |
|-------------|------------|----------|----------------------------|
| pod         | po         | namespace | Workload unit              |
| deployment  | deploy     | namespace | Manages ReplicaSets        |
| statefulset | sts        | namespace | Stable IDs for pods        |
| daemonset   | ds         | namespace | Run one pod per node       |
| service     | svc        | namespace | Exposes workloads          |
| ingress     | ing        | namespace | HTTP routing               |
| configmap   | cm         | namespace | Config data                |
| secret      | secret     | namespace | Sensitive data             |
| node        | no         | cluster   | Node in cluster            |
| namespace   | ns         | cluster   | Isolation boundary         |
| pvc         | pvc        | namespace | Storage request            |
| pv          | pv         | cluster   | Persistent volume          |
| job         | job        | namespace | Batch workload             |
| cronjob     | cj         | namespace | Scheduled jobs             |

---

## List & Inspect

| Command                                      | Purpose                           |
|----------------------------------------------|---------------------------------|
| `kubectl get all -n <ns>`                    | List all resources in namespace   |
| `kubectl describe po <pod> -n <ns>`          | Detailed pod info                 |
| `kubectl get events --sort-by=.lastTimestamp -n <ns>` | List sorted events          |
| `kubectl get nodes -o wide`                   | Node details                     |

---

## Create / Apply / Patch / Delete

| Command                             | Description                   |
|-----------------------------------|-------------------------------|
| `kubectl create -f x.yaml`         | Create resources from file     |
| `kubectl apply -f x.yaml`          | Apply changes from file        |
| `kubectl replace -f x.yaml`        | Replace existing resources     |
| `kubectl patch deploy <name> -p '<json>'` | Patch resource            |
| `kubectl delete -f x.yaml`         | Delete resources from file     |

---

## Deployments & Rollouts

| Command                          | Purpose                        |
|---------------------------------|--------------------------------|
| `kubectl create deploy web --image=nginx -n <ns>` | Create deployment     |
| `kubectl scale deploy web --replicas=5 -n <ns>`   | Scale deployment      |
| `kubectl rollout status deploy/web -n <ns>`       | Check rollout status  |
| `kubectl rollout undo deploy/web -n <ns>`         | Undo deployment       |
| `kubectl rollout restart deployment/<name>`       | Restart deployment    |

---

## Debugging, Logs, Exec, Port-Forward, Copy

| Command                                      | Purpose                         |
|----------------------------------------------|--------------------------------|
| `kubectl logs <pod> -n <ns>`                  | View pod logs                  |
| `kubectl exec -it <pod> -n <ns> -- sh`        | Execute shell in pod           |
| `kubectl port-forward svc/web 8080:80 -n <ns>` | Forward local port            |
| `kubectl cp <ns>/<pod>:/path ./local`          | Copy files from pod            |

---

## Services, Endpoints & Ingress

| Command                                      | Purpose                              |
|----------------------------------------------|------------------------------------|
| `kubectl expose deploy web --port=80 --target-port=8080 -n <ns>` | Expose service as ClusterIP  |
| `kubectl expose deploy web --type=NodePort --port=80 -n <ns>`    | Expose as NodePort            |
| `kubectl get ing -n <ns>`                     | List Ingress resources              |

---

## ConfigMaps & Secrets

| Command                                       | Description                    |
|-----------------------------------------------|-------------------------------|
| `kubectl create cm app-cfg --from-literal=KEY=VALUE -n <ns>`  | Create ConfigMap       |
| `kubectl create secret generic app-sec --from-literal=PASS=123 -n <ns>` | Create Secret        |

---

## Nodes & Maintenance

| Command                                   | Description                     |
|-------------------------------------------|--------------------------------|
| `kubectl cordon <node>` / `uncordon <node>` | Mark node unschedulable/schedulable |
| `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data` | Drain node for maintenance |

---

## Namespaces, Labels & Annotations

| Command                          | Description                  |
|---------------------------------|------------------------------|
| `kubectl create ns <ns>`          | Create namespace             |
| `kubectl label po <pod> env=prod -n <ns>` | Add label to pod     |

---

## RBAC & Auth

| Command                                    | Use Case                    |
|--------------------------------------------|-----------------------------|
| `kubectl auth whoami`                      | Show current user           |
| `kubectl auth can-i get pods -n <ns>`     | Check permissions           |
| `kubectl create token <sa> -n <ns>`       | Create ServiceAccount token |

---

## Cluster Info & Health

| Command                       | Purpose                 |
|-------------------------------|-------------------------|
| `kubectl version --short`      | Show client/server versions |
| `kubectl cluster-info`         | Display cluster info    |
| `kubectl get --raw /healthz`   | Cluster health endpoint |

---

## Metrics

| Command                | Description                |
|------------------------|----------------------------|
| `kubectl top nodes`    | Show resource usage of nodes |
| `kubectl top pods -A`  | Show resource usage of all pods |

---

## Jobs & CronJobs

| Command | Purpose |
|---------|---------|
| `kubectl create job pi --image=perl -- perl -Mbignum=bpi -wle 'print bpi(2000)' -n <ns>` | Create job |
| `kubectl create cronjob hello --image=busybox --schedule="*/1 * * * *" -- echo hi -n <ns>` | Create cronjob |

---

## Storage

| Command                      | Description                      |
|------------------------------|--------------------------------|
| `kubectl get pv,pvc,sc`      | List PersistentVolumes, Claims, StorageClasses |
| `kubectl describe pvc <name> -n <ns>` | Describe PVC              |

---

## Contexts & Kubeconfig

| Command                         | Purpose                   |
|---------------------------------|----------------------------|
| `kubectl config get-contexts`   | List contexts              |
| `kubectl config use-context <ctx>` | Switch context          |
| `kubectl config set-context --current --namespace=<ns>` | Set default namespace |

---

## Kustomize

| Command                 | Purpose                |
|-------------------------|------------------------|
| `kubectl kustomize ./overlays/prod` | Render manifests    |
| `kubectl apply -k ./overlays/prod`   | Apply overlays       |

---

## kubeadm — Cluster Lifecycle

| Command                              | Purpose                  |
|------------------------------------|--------------------------|
| `sudo kubeadm init --pod-network-cidr=<cidr>` | Initialize cluster      |
| `sudo kubeadm join <api:6443> --token <token> --discovery-token-ca-cert-hash sha256:<hash>` | Join node |
| `sudo kubeadm token create --print-join-command` | Generate join command   |
| `sudo kubeadm reset -f`             | Reset node               |
| `sudo kubeadm upgrade apply v<version>` | Upgrade cluster version |
| `sudo kubeadm certs renew all`      | Renew certificates       |

---

## krew — Plugin Manager

| Command                         | Purpose                  |
|---------------------------------|--------------------------|
| Install krew                   | [See docs](https://krew.sigs.k8s.io/docs/user-guide/setup/install/) |
| `kubectl krew list`             | List installed plugins    |
| `kubectl krew install <plugin>` | Install a plugin          |
| Popular plugins: kubectx, kubens, neat, tree, who-can, view-utilization |

---

## Container Runtimes (crictl, nerdctl, ctr)

| Runtime | Example Usage                      | Description            |
|---------|----------------------------------|------------------------|
| `crictl` | `sudo crictl ps`, `sudo crictl logs <id>` | CRI-compatible CLI     |
| `nerdctl` | `sudo nerdctl ps`, `sudo nerdctl exec -it <ctr> sh` | Containerd CLI        |
| `ctr`   | `sudo ctr -n k8s.io c ls`         | Low-level containerd CLI |

---

## Helm — Chart Manager

| Command                          | Purpose                   |
|---------------------------------|---------------------------|
| `helm repo add <name> <url>`    | Add Helm repo             |
| `helm repo update`              | Update repos              |
| `helm upgrade --install <release> <chart> -n <ns>` | Install or upgrade helm release |
| `helm list -A`                  | List all Helm releases    |
| `helm uninstall <release> -n <ns>` | Uninstall Helm release |

---


Created by Saime Shaikh  
DevOps Enthusiast 

