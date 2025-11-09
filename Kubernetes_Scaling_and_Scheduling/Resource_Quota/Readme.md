


```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3                    # Number of Pods to create
  selector:
    matchLabels:
      app: nginx                 # Select Pods with this label
  template:                      # Pod template
    metadata:
      labels:
        app: nginx               # Label for Pods
    spec:
      containers:
      - name: nginx
        image: nginx:1.21        # Container image
        ports:
        - containerPort: 80      # Port exposed by container
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"

```
## Apply 
```bash
kubectl apply -f nginx-deployment.yaml
kubectl get pods -o wide
kubectl describe pod <pod-name>
``` 
