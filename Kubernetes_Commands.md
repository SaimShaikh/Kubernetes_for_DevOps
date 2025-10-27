# 📚 Kubernetes CLI Cheatsheet (All-in-One)

> Ready for GitHub as `CHEATSHEET.md`. Includes **kubectl**, **kubeadm**, **krew/plugins**, **container-runtime CLIs**, and **Helm**. Replace `<...>` with your actual values.

---



## 🔸 kubectl — Global Syntax & Power Flags

| Topic             | Command/Flag                            | Notes                    |               |       |               |
| ----------------- | --------------------------------------- | ------------------------ | ------------- | ----- | ------------- |
| Basic form        | `kubectl <verb> <resource> [name]`      | e.g., `kubectl get pods` |               |       |               |
| Namespace         | `-n <ns>` / `--namespace <ns>`          | Target a namespace       |               |       |               |
| Output            | `-o wide                                | yaml                     | json          | name` | Format output |
| Labels            | `-l key=value`                          | Label selector           |               |       |               |
| Fields            | `--field-selector key=value`            | Field selector           |               |       |               |
| Watch             | `-w` / `--watch`                        | Stream changes           |               |       |               |
| Dry run           | `--dry-run=client                       | server`                  | Validate only |       |               |
| Filename          | `-f file.yaml` / `-k dir/`              | File / Kustomize dir     |               |       |               |
| Context           | `--context <ctx>`                       | Use specific context     |               |       |               |
| Impersonate       | `--as <user>` `--as-group <grp>`        | RBAC tests               |               |       |               |
| Server-side apply | `--server-side`                         | SSA                      |               |       |               |
| Sort              | `--sort-by=.metadata.creationTimestamp` | Sort by fields           |               |       |               |

---

## 🔸 API Discovery & Resources

| Goal           | Command                                |
| -------------- | -------------------------------------- |
| API resources  | `kubectl api-resources`                |
| API versions   | `kubectl api-versions`                 |
| Explain fields | `kubectl explain deploy.spec.strategy` |

---

## 🔸 Resource Kinds & Short Names

| Kind        | Short  | Scoped  | Purpose            |
| ----------- | ------ | ------- | ------------------ |
| pod         | po     | ns      | Workload unit      |
| deployment  | deploy | ns      | Manage ReplicaSets |
| statefulset | sts    | ns      | Stable IDs         |
| daemonset   | ds     | ns      | One per node       |
| service     | svc    | ns      | Expose workload    |
| ingress     | ing    | ns      | HTTP routing       |
| configmap   | cm     | ns      | Config data        |
| secret      | secret | ns      | Sensitive data     |
| node        | no     | cluster | Node               |
| namespace   | ns     | cluster | Isolation boundary |
| pvc         | pvc    | ns      | Request storage    |
| pv          | pv     | cluster | Persistent storage |
| job         | job    | ns      | Batch workload     |
| cronjob     | cj     | ns      | Scheduled job      |

> Run `kubectl api-resources` for the full list.

---

## 🔸 List & Inspect

| Goal         | Command                                               |
| ------------ | ----------------------------------------------------- |
| List all     | `kubectl get all -n <ns>`                             |
| Describe pod | `kubectl describe po <pod> -n <ns>`                   |
| Events       | `kubectl get events --sort-by=.lastTimestamp -n <ns>` |
| Node info    | `kubectl get nodes -o wide`                           |

---

## 🔸 Create / Apply / Replace / Patch / Delete

| Action  | Command                                   |
| ------- | ----------------------------------------- |
| Create  | `kubectl create -f x.yaml`                |
| Apply   | `kubectl apply -f x.yaml`                 |
| Replace | `kubectl replace -f x.yaml`               |
| Patch   | `kubectl patch deploy <name> -p '<json>'` |
| Delete  | `kubectl delete -f x.yaml`                |

---

## 🔸 Deployments & Rollouts

| Goal           | Command                                           |
| -------------- | ------------------------------------------------- |
| Create         | `kubectl create deploy web --image=nginx -n <ns>` |
| Scale          | `kubectl scale deploy web --replicas=5 -n <ns>`   |
| Rollout status | `kubectl rollout status deploy/web -n <ns>`       |
| Undo           | `kubectl rollout undo deploy/web -n <ns>`         |

---

## 🔸 Debugging, Logs, Exec, Port-Forward, Copy

| Action       | Command                                        |
| ------------ | ---------------------------------------------- |
| Logs         | `kubectl logs <pod> -n <ns>`                   |
| Exec shell   | `kubectl exec -it <pod> -n <ns> -- sh`         |
| Port-forward | `kubectl port-forward svc/web 8080:80 -n <ns>` |
| Copy files   | `kubectl cp <ns>/<pod>:/path ./local`          |

---

## 🔸 Services, Endpoints & Ingress

| Action              | Command                                                          |
| ------------------- | ---------------------------------------------------------------- |
| Expose as ClusterIP | `kubectl expose deploy web --port=80 --target-port=8080 -n <ns>` |
| NodePort            | `kubectl expose deploy web --type=NodePort --port=80 -n <ns>`    |
| Ingress list        | `kubectl get ing -n <ns>`                                        |

---

## 🔸 ConfigMaps & Secrets

| Goal      | Command                                                                 |
| --------- | ----------------------------------------------------------------------- |
| ConfigMap | `kubectl create cm app-cfg --from-literal=KEY=VALUE -n <ns>`            |
| Secret    | `kubectl create secret generic app-sec --from-literal=PASS=123 -n <ns>` |

---

## 🔸 Nodes & Maintenance

| Goal            | Command                                                           |
| --------------- | ----------------------------------------------------------------- |
| Cordon/Uncordon | `kubectl cordon <node>` / `kubectl uncordon <node>`               |
| Drain           | `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data` |

---

## 🔸 Namespaces, Labels & Annotations

| Goal      | Command                                   |
| --------- | ----------------------------------------- |
| Create ns | `kubectl create ns <ns>`                  |
| Label     | `kubectl label po <pod> env=prod -n <ns>` |

---

## 🔸 RBAC & Auth

| Goal                 | Command                               |
| -------------------- | ------------------------------------- |
| Whoami               | `kubectl auth whoami`                 |
| Can I?               | `kubectl auth can-i get pods -n <ns>` |
| ServiceAccount token | `kubectl create token <sa> -n <ns>`   |

---

## 🔸 Cluster Info & Health

| Goal    | Command                      |
| ------- | ---------------------------- |
| Version | `kubectl version --short`    |
| Info    | `kubectl cluster-info`       |
| Health  | `kubectl get --raw /healthz` |

---

## 🔸 Metrics

| Goal      | Command               |
| --------- | --------------------- |
| Top nodes | `kubectl top nodes`   |
| Top pods  | `kubectl top pods -A` |

---

## 🔸 Jobs & CronJobs

| Goal           | Command                                                                                    |
| -------------- | ------------------------------------------------------------------------------------------ |
| Create job     | `kubectl create job pi --image=perl -- perl -Mbignum=bpi -wle 'print bpi(2000)' -n <ns>`   |
| Create cronjob | `kubectl create cronjob hello --image=busybox --schedule="*/1 * * * *" -- echo hi -n <ns>` |

---

## 🔸 Storage

| Goal          | Command                               |
| ------------- | ------------------------------------- |
| Get PV/PVC/SC | `kubectl get pv,pvc,sc`               |
| Describe PVC  | `kubectl describe pvc <name> -n <ns>` |

---

## 🔸 Contexts & Kubeconfig

| Goal           | Command                                                 |
| -------------- | ------------------------------------------------------- |
| List contexts  | `kubectl config get-contexts`                           |
| Use context    | `kubectl config use-context <ctx>`                      |
| Set default ns | `kubectl config set-context --current --namespace=<ns>` |

---

## 🔸 Kustomize

| Goal   | Command                             |
| ------ | ----------------------------------- |
| Render | `kubectl kustomize ./overlays/prod` |
| Apply  | `kubectl apply -k ./overlays/prod`  |

---

## 🔸 kubeadm — Cluster Lifecycle

| Action            | Command                                                                                     |
| ----------------- | ------------------------------------------------------------------------------------------- |
| Init cluster      | `sudo kubeadm init --pod-network-cidr=<cidr>`                                               |
| Join node         | `sudo kubeadm join <api:6443> --token <token> --discovery-token-ca-cert-hash sha256:<hash>` |
| Generate join cmd | `sudo kubeadm token create --print-join-command`                                            |
| Reset node        | `sudo kubeadm reset -f`                                                                     |
| Upgrade           | `sudo kubeadm upgrade apply v<version>`                                                     |
| Renew certs       | `sudo kubeadm certs renew all`                                                              |

---

## 🔸 krew — Plugin Manager

| Task           | Command                                                                               |
| -------------- | ------------------------------------------------------------------------------------- |
| Install krew   | [See docs: krew.sigs.k8s.io](https://krew.sigs.k8s.io/docs/user-guide/setup/install/) |
| List plugins   | `kubectl krew list`                                                                   |
| Install plugin | `kubectl krew install <plugin>`                                                       |
| Popular        | `kubectx`, `kubens`, `neat`, `tree`, `who-can`, `view-utilization`                    |

---

## 🔸 Container Runtime CLIs

| Runtime | Command                                                         | Example              |
| ------- | --------------------------------------------------------------- | -------------------- |
| crictl  | `sudo crictl ps`, `sudo crictl images`, `sudo crictl logs <id>` | CRI-compatible       |
| nerdctl | `sudo nerdctl ps`, `sudo nerdctl exec -it <ctr> sh`             | containerd CLI       |
| ctr     | `sudo ctr -n k8s.io c ls`                                       | low-level containerd |

---

## 🔸 Helm — Chart Manager

| Goal            | Command                                            |
| --------------- | -------------------------------------------------- |
| Add repo        | `helm repo add <name> <url>`                       |
| Update repos    | `helm repo update`                                 |
| Install/upgrade | `helm upgrade --install <release> <chart> -n <ns>` |
| List releases   | `helm list -A`                                     |
| Uninstall       | `helm uninstall <release> -n <ns>`                 |

---

## 🔸 Pro Tips

* Use **label selectors** everywhere: `-l app=<name>`.
* Prefer **server-side apply** for shared clusters.
* Keep kubeconfig tidy using `kubectx`/`kubens`.
* For help: `kubectl --help`, `kubeadm --help`, `helm --help`.

---

**Created by Saime Shaikh (Boss)**
DevOps Enthusiast | Kubernetes | AWS | Docker | Terraform | Jenkins | ArgoCD
