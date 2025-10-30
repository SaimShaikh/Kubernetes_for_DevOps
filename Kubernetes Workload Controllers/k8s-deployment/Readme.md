```bash
# ------------------------------------------------------------
# Deployment (manages Pods, handles rolling updates)
# ------------------------------------------------------------
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx        # MUST MATCH -> spec.template.metadata.labels.app (below)

  template:
    metadata:
      labels:
        app: nginx      # MUST MATCH -> spec.selector.matchLabels.app (above)
    spec:
      containers:
        - name: nginx
          image: nginx:1.14.2
          ports:
            - containerPort: 80  # MUST MATCH -> Service.spec.ports[*].targetPort (below)


```
