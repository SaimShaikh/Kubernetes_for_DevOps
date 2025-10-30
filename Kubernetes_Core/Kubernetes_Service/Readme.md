```bash
# ------------------------------------------------------------
# Service (optional but typical—exposes your Pods inside cluster)
# ------------------------------------------------------------
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx          # MUST MATCH -> Pod labels at spec.template.metadata.labels.app (in Deployment)
  ports:
    - port: 80          # Client-facing service port (can be different from targetPort)
      targetPort: 80    # MUST MATCH -> containerPort (in Deployment Pod spec)
```
