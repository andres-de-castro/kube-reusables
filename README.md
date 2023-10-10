# Delete Resources older than X amount of time

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pvc-deletion-sa
  namespace: default
--- 
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pvc-deletion-role
  namespace: default
rules:
- apiGroups:
  - ""
  resources:
  - persistentvolumeclaims
  verbs:
  - get
  - list
  - delete
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pvc-deletion-rolebinding
  namespace: default
subjects:
- kind: ServiceAccount
  name: pvc-deletion-sa
  namespace: default
roleRef:
  kind: ClusterRole
  name: pvc-deletion-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: pvc-deletion-cronjob
  namespace: default
spec:
  schedule: "*/15 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: pvc-deletion-sa
          containers:
          - name: pvc-deletion-container
            resources:
              requests:
                cpu: 500m
                memory: 500Mi
              limits:
                cpu: 500m
                memory: 500Mi                              
            image: bitnami/kubectl
            command:
              - "bash"
              - "-c"
              - kubectl get pvc -o go-template --template '{{range .items}}{{.metadata.name}} {{.metadata.creationTimestamp}}{{"\n"}}{{end}}' | awk '$2 <= "'$(date -d'now-6 hours' -Ins --utc | sed 's/+0000/Z/')'" { print $1 }' | xargs --no-run-if-empty kubectl delete pvc
          restartPolicy: OnFailure
```
