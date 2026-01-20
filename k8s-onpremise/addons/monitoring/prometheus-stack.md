## Prometheus Stack (prometheus, grafana, alertManager)
```
helm repo add https://github.com/prometheus-community/helm-charts
helm repo update
```

```
values-prometheus-stack.yaml

prometheusSpec:
  storageSpec:
    volumeClaimTemplate:
      spec:
        storageClassName: nfs-storage
        resources:
          requests:
            storage: 50Gi

grafana:
  adminUser: admin
  adminPassword: eyb!3GEy


```

```
helm repo install kube-prometheus-stack https://prometheus-community.github.io/helm-charts
```