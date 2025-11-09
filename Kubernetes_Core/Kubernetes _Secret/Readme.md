## Practice 

``echo "root" | base64``

## secret.yml
```bash
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: mysql
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: cm9vdA==   # base64 of "root"

```
## Headless Service
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
                secretKeyRef:
                  name: mysql-secret
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


```bash
# Create namespace
kubectl create namespace mysql

# Apply all configs
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f service.yaml
kubectl apply -f statefulset.yaml

# Verify
kubectl get all -n mysql
kubectl get pvc -n mysql
kubectl logs mysql-statefulset-0 -n mysql
```

## Output should be 
```bash

NAME                  READY   STATUS    RESTARTS   AGE
mysql-statefulset-0   1/1     Running   0          2m
mysql-statefulset-1   1/1     Running   0          1m
mysql-statefulset-2   1/1     Running   0          1m
```
