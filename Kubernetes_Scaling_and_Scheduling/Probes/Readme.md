## Practice 

```bash
# 🏷️ Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: nginx-space
```
## 🐋 Deployment
```bash 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: nginx-space
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80

        # 🧠 Health Checks
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
          failureThreshold: 3

        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
---
```
##  🌐 Service (NodePort)
```bash
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: nginx-space
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80          # Service port
      targetPort: 80    # Container port
      nodePort: 30080   # Port accessible from browser (KillerKoda supports 30000–32767)
```
```bash
kubectl apply -f nginx-full.yaml
kubectl get all -n nginx-space
```
```bash
NAME                                   READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-xxxxx             1/1     Running   0          10s

NAME                      TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/nginx-service     NodePort   10.xxx.xxx.x   <none>        80:30080/TCP   10s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   3/3     3            3           10s
```

## Create YAML files and 🌍 Access in Browser
