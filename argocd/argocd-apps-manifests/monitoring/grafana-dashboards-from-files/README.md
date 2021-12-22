To apply manually run:

```
kubectl kustomize ./ > tst.yaml && kubectl apply -f tst.yaml
```

Example ArgoCD app:

```
project: default
source:
  repoURL: 'https://github.com/shalb/monitoring-examples.git'
  path: argocd/argocd-apps-manifests/monitoring/grafana-dashboards-from-files
  targetRevision: HEAD
destination:
  server: 'https://kubernetes.default.svc'
  namespace: monitoring
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```
