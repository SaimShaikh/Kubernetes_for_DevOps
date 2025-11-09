## Practice 
## configmap.yaml
```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config-map
  namespace: mysql
data:
  MYSQL_DATABASE: "<ANY NAME >"
  MYSQL_ROOT_PASSWORD: "root"   # dev/test only
```
## service.yaml
```bash
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
    - name: mysql
      port: 3306
      targetPort: 3306
```
## statefulset.yaml
```bash

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-statefulset
  namespace: mysql
spec:
  serviceName: "mysql-service"
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: mysql
              containerPort: 3306
          env:
            - name: MYSQL_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: mysql-config-map
                  key: MYSQL_DATABASE
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                configMapKeyRef:
                  name: mysql-config-map
                  key: MYSQL_ROOT_PASSWORD
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
      terminationGracePeriodSeconds: 30
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

### How To Apply 
```bash
# 1. Create namespace (if not already)
kubectl create namespace mysql

# 2. Apply ConfigMap
kubectl apply -f configmap.yaml

# 3. Apply headless Service
kubectl apply -f service.yaml

# 4. Apply StatefulSet
kubectl apply -f statefulset.yaml
```

### Verify Deployment

```bash
kubectl get all -n mysql
kubectl get pvc -n mysql
kubectl logs mysql-statefulset-0 -n mysql
```
### Out Should be
```bash
NAME                  READY   STATUS    RESTARTS   AGE
mysql-statefulset-0   1/1     Running   0          2m
mysql-statefulset-1   1/1     Running   0          1m
mysql-statefulset-2   1/1     Running   0          1m
```

