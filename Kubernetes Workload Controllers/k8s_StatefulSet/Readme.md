## Practice 

### service.yml

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

### stateful.yml

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
            - name: MYSQL_ROOT_PASSWORD
              value: "root"
            - name: MYSQL_DATABASE
              value: "devops"
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
kubectl create namespace mysql
kubectl apply -f mysql-service.yaml
kubectl apply -f mysql-statefulset.yaml

kubectl get svc -n mysql
kubectl get sts -n mysql
kubectl get pods -n mysql
kubectl get pvc -n mysql
kubectl describe sts/mysql-statefulset -n mysql

```
