# OpenSearch

This folder contains a current Helm-based OpenSearch setup using the official OpenSearch Kubernetes operator charts.

## 2026 install path

The current maintained Kubernetes path is:

1. Install the operator chart
2. Install the cluster chart

As of March 14, 2026:

- Latest non-alpha operator chart: `opensearch-operator` `2.8.2`
- Latest non-alpha cluster chart: `opensearch-cluster` `3.2.2`
- Latest OpenSearch server release: `3.5.0`

The latest operator chart in the repo index is `3.0.0`, but that line is already on the `3.0.0-alpha` operator app version. This folder stays on the latest stable non-alpha operator line.

## 1. Add the Helm repo

```bash
helm repo add opensearch https://opensearch-project.github.io/opensearch-k8s-operator/
helm repo update
```

## 2. Install the operator

```bash
kubectl create namespace opensearch
helm upgrade --install opensearch-operator opensearch/opensearch-operator \
  --namespace opensearch \
  --version 2.8.2
```

## 3. Review the cluster values

Edit `values.yaml` before installing:

- change the Dashboards hostname
- adjust `diskSize`
- adjust memory and CPU for your cluster

The included values file enables:

- one OpenSearch node pool
- one Dashboards replica
- Traefik ingress for Dashboards
- example Dashboards hostname `dashboards.example.com`

## 4. Install the cluster

```bash
helm upgrade --install opensearch-cluster opensearch/opensearch-cluster \
  --namespace opensearch \
  --version 3.2.2 \
  -f opensearch/values.yaml
```

## 5. Verify the install

```bash
helm list -n opensearch
kubectl get pods -n opensearch
kubectl get ingress -n opensearch
```

## Notes

- The previous repo root `cluster.yaml` used the legacy `opensearch.opster.io/v1` API group. The values file here uses the modern `opensearch.org` path through the current cluster chart.
- The local values file pins OpenSearch and Dashboards to `2.8.0`, which matches the stable operator line used here. If you want to move to newer 3.x engine releases, check operator compatibility first.

Upstream docs and sources:

- Operator repo: <https://github.com/opensearch-project/opensearch-k8s-operator>
- Helm repo index: <https://opensearch-project.github.io/opensearch-k8s-operator/>
- OpenSearch releases: <https://github.com/opensearch-project/OpenSearch/releases>
