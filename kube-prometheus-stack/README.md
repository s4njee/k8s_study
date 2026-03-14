# kube-prometheus-stack

This folder contains the local Helm values file for the `prometheus-community/kube-prometheus-stack` chart.

Install it with:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install monitoring-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --version 82.10.3 \
  -f kube-prometheus-stack/values.yaml
```

The default values here assume:

- Traefik is your ingress controller
- Grafana should be reachable at `grafana.example.com`
- Prometheus should be reachable at `prometheus.example.com`
- Grafana and Prometheus both need persistent storage

Update the hostnames and storage sizes before installing.
