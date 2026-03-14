# Platform Services

This guide covers the add-ons that sit on top of the base cluster: CI runners, monitoring, registry infrastructure, database examples, and optional platform components.

Follow [Cluster bootstrap](cluster-bootstrap.md) first so storage, ingress, and kubeconfig access are already in place.

## Table of contents

- [Install Actions Runner Controller](#install-actions-runner-controller)
- [Install kube-prometheus-stack](#install-kube-prometheus-stack)
- [Install Grafana Dashboards for Kubernetes](#install-grafana-dashboards-for-kubernetes)
- [Set up cert-manager for the Docker Registry certificate](#set-up-cert-manager-for-the-docker-registry-certificate)
- [Set up Docker Registry](#set-up-docker-registry)
- [Set up PostgreSQL](#set-up-postgresql)
- [Optional infrastructure](#optional-infrastructure)
- [Cluster operations](#cluster-operations)

## Install Actions Runner Controller

Actions Runner Controller (ARC) lets you run GitHub Actions CI/CD jobs on your own Kubernetes cluster instead of using GitHub's hosted runners.

GitHub's current recommendation is to use the runner scale set approach, which installs two Helm charts: one for the controller and one for each set of runners.

If you want to pin a tested chart version, the latest upstream release as of March 14, 2026 is `0.13.1` (release tag `gha-runner-scale-set-0.13.1` published on December 23, 2025).

### 1. Install the ARC controller

Use a dedicated namespace for the controller:

```bash
export ARC_CONTROLLER_NS=arc-systems
helm install arc \
  --namespace "${ARC_CONTROLLER_NS}" \
  --create-namespace \
  --version 0.13.1 \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

### 2. Create GitHub credentials in the runner namespace

For organization or repository runners, a GitHub App is the recommended authentication method. Create the secret in the runner namespace:

```bash
export ARC_RUNNERS_NS=arc-runners
kubectl create namespace "${ARC_RUNNERS_NS}"
kubectl create secret generic arc-github-app \
  --namespace "${ARC_RUNNERS_NS}" \
  --from-literal=github_app_id='<app_id>' \
  --from-literal=github_app_installation_id='<installation_id>' \
  --from-file=github_app_private_key=/path/to/private-key.pem
```

If you prefer a token-based setup instead, create:

```bash
kubectl create secret generic arc-github-token \
  --namespace "${ARC_RUNNERS_NS}" \
  --from-literal=github_token='<pat_or_fine_grained_pat>'
```

### 3. Install a runner scale set

Use a separate namespace for runner pods:

```bash
export ARC_RUNNER_SET=arc-runner-set
export GITHUB_CONFIG_URL='https://github.com/<org-or-repo>'
helm install "${ARC_RUNNER_SET}" \
  --namespace "${ARC_RUNNERS_NS}" \
  --create-namespace \
  --version 0.13.1 \
  --set githubConfigUrl="${GITHUB_CONFIG_URL}" \
  --set githubConfigSecret=arc-github-app \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

If you created the token secret instead of the GitHub App secret, change the last command to:

```bash
helm install "${ARC_RUNNER_SET}" \
  --namespace "${ARC_RUNNERS_NS}" \
  --create-namespace \
  --version 0.13.1 \
  --set githubConfigUrl="${GITHUB_CONFIG_URL}" \
  --set githubConfigSecret=arc-github-token \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

### 4. Verify the install

```bash
helm list -A | grep -E 'arc|gha-runner-scale-set'
kubectl get pods -n "${ARC_CONTROLLER_NS}"
kubectl get pods -n "${ARC_RUNNERS_NS}"
```

### 5. Use the runner scale set in a GitHub Actions workflow

In your workflow YAML file, set `runs-on` to the Helm release name you chose for the runner scale set:

```yaml
name: ARC Demo
on:
  workflow_dispatch:

jobs:
  test:
    runs-on: arc-runner-set
    steps:
      - run: echo "Running on ARC"
```

You can watch runner pods spin up and shut down in real time while a workflow runs:

```bash
kubectl get pods -n "${ARC_RUNNERS_NS}" -w
```

Further reading:

- [Quickstart](https://docs.github.com/en/actions/tutorials/use-actions-runner-controller/quickstart)
- [Deploying runner scale sets](https://docs.github.com/en/actions/tutorials/use-actions-runner-controller/deploy-runner-scale-sets)
- [Authentication](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/authenticating-to-the-github-api)

## Install kube-prometheus-stack

This installs a complete monitoring stack in one step: Prometheus, Grafana, Alertmanager, and supporting components like kube-state-metrics and node-exporter.

If you want to pin a tested chart version, the latest release as of March 14, 2026 is `82.10.3` (app version `v0.89.0`, published March 10, 2026).

### 1. Add the Prometheus Community Helm repo

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

You can confirm the chart version with:

```bash
helm search repo prometheus-community/kube-prometheus-stack
```

### 2. Create the monitoring namespace

```bash
kubectl create namespace monitoring
```

### 3. Use the local values file

This repo includes a starter values file at `kube-prometheus-stack/values.yaml` with:

- Grafana enabled with Traefik ingress and persistent storage
- Prometheus enabled with persistent storage and a Traefik ingress
- Alertmanager enabled with persistent storage

### 4. Install the stack

```bash
helm upgrade --install monitoring-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --version 82.10.3 \
  -f kube-prometheus-stack/values.yaml
```

### 5. Verify the install

```bash
helm list -n monitoring
kubectl get all -n monitoring
```

### 6. Get the Grafana admin password

```bash
kubectl get secret --namespace monitoring monitoring-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

### 7. Access Grafana

If ingress is configured in the values file, open the Grafana hostname you configured in `kube-prometheus-stack/values.yaml`.

If you want local access first:

```bash
kubectl port-forward -n monitoring svc/monitoring-stack-grafana 3000:80
```

### 8. Access Prometheus

If ingress is configured, open the Prometheus hostname from `kube-prometheus-stack/values.yaml`.

If you want local access instead:

```bash
kubectl port-forward -n monitoring svc/monitoring-stack-kube-prom-prometheus 9090:9090
```

Further reading:

- [Chart repo](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Values reference](https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/values.yaml)

## Install Grafana Dashboards for Kubernetes

The `grafana-dashboards-kubernetes/` folder contains a set of pre-made Grafana dashboards that show CPU, memory, network, and pod-level metrics for your cluster. They are managed with Kustomize.

This is a clone of the [`dotdc/grafana-dashboards-kubernetes`](https://github.com/dotdc/grafana-dashboards-kubernetes) project.

### Included dashboards

| ConfigMap name | Dashboard file |
|---|---|
| `dashboards-k8s-views-global` | `k8s-views-global.json` |
| `dashboards-k8s-views-namespaces` | `k8s-views-namespaces.json` |
| `dashboards-k8s-views-nodes` | `k8s-views-nodes.json` |
| `dashboards-k8s-views-pods` | `k8s-views-pods.json` |
| `dashboards-k8s-system-api-server` | `k8s-system-api-server.json` |
| `dashboards-k8s-system-coredns` | `k8s-system-coredns.json` |
| `dashboards-k8s-addons-prometheus` | `k8s-addons-prometheus.json` |
| `dashboards-k8s-addons-trivy-operator` | `k8s-addons-trivy-operator.json` |

### Prerequisites

These dashboards are designed to work with the Grafana instance from the [kube-prometheus-stack](#install-kube-prometheus-stack) section above.

### Deploy the dashboards

```bash
kubectl apply -k grafana-dashboards-kubernetes/
```

> [!TIP]
> If your Grafana is in a namespace other than `default`, uncomment and set the `namespace` field in `grafana-dashboards-kubernetes/kustomization.yaml`.

### Verify

```bash
kubectl get configmaps -l grafana_dashboard=1
```

## Set up cert-manager for the Docker Registry certificate

cert-manager automatically obtains and renews TLS certificates. This repo uses cert-manager with Let's Encrypt to issue the HTTPS certificate for the Docker registry.

### 1. Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

For a repeatable guide, prefer a release-tagged URL after you verify the version you want to teach:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/<tested_cert_manager_version>/cert-manager.yaml
```

### 2. Create a Let's Encrypt issuer

For testing, start with staging:

```bash
kubectl apply -f certbox/letsencrypt-staging.yaml
```

For production, use:

```bash
kubectl apply -f certbox/letsencrypt-prod.yaml
```

The production issuer in `certbox/letsencrypt-prod.yaml` creates a `ClusterIssuer` named `letsencrypt-prod`. The certificate request in `docker-registry.helm/certificate.yaml` references this issuer by name.

### 3. Make the registry hostname reachable

Before requesting the certificate:

1. Create a DNS `A` record for your registry hostname.
2. Point it at the public IP or reachable LAN IP for your K3s ingress.
3. Make sure ports `80` and `443` can reach Traefik.

### 4. Issue the Docker registry certificate

The checked-in example uses the `registry` namespace. Update `docker-registry.helm/certificate.yaml` if you want a different namespace, hostname, or issuer, then apply it:

```bash
kubectl apply -f docker-registry.helm/certificate.yaml
```

Check certificate status:

```bash
kubectl describe certificate docker-registry-certificate -n registry
kubectl get challenge,order -A
```

### 5. Install the registry

After the certificate secret exists, install the Docker registry chart.

## Set up Docker Registry

A container image registry is where you store the Docker images that Kubernetes pulls when it starts your apps.

> [!NOTE]
> If you need a full registry platform with UI, RBAC, vulnerability scanning, replication, and robot accounts, Harbor is the more common 2026 choice. This repo keeps CNCF Distribution because it is still the simplest option for a small homelab registry.

### Recommended setup

Use HTTPS and password authentication. The registry uses an `htpasswd` file for authentication. Instead of putting the password directly in `values.yaml`, pass the file to Helm separately with `--set-file` so it stays out of version control.

Create an `htpasswd` file locally:

```bash
mkdir -p auth
docker run --rm --entrypoint htpasswd httpd:2 -Bbn <registry_user> <registry_password> > auth/htpasswd
```

Review and update these files before installing:

- `docker-registry.helm/values.yaml`
- `docker-registry.helm/certificate.yaml`

Then install the chart from this repo:

```bash
helm upgrade --install private-registry ./docker-registry.helm \
  --namespace registry \
  --create-namespace \
  --set persistence.enabled=true \
  --set-file secrets.htpasswd=auth/htpasswd
```

### Log in and push images

```bash
docker login <registry_hostname>
docker tag my-image:latest <registry_hostname>/my-image:latest
docker push <registry_hostname>/my-image:latest
```

### Let Kubernetes pull from the private registry

```bash
kubectl create secret docker-registry regcred \
  --namespace <namespace> \
  --docker-server=<registry_hostname> \
  --docker-username=<registry_user> \
  --docker-password=<registry_password>
```

Then add `imagePullSecrets: [{name: regcred}]` to your Pod, Deployment, or ServiceAccount spec.

### Configure K3s to trust your registry

If your registry uses a certificate from a well-known authority, K3s will trust it automatically.

If your registry uses a self-signed certificate or a private CA, add an entry to `/etc/rancher/k3s/registries.yaml` on each node:

```yaml
mirrors:
  "<registry_hostname>":
    endpoint:
      - "https://<registry_hostname>"
configs:
  "<registry_hostname>":
    auth:
      username: <registry_user>
      password: <registry_password>
    tls:
      ca_file: /etc/rancher/k3s/certs/<registry_ca>.crt
```

> [!IMPORTANT]
> Restart K3s after changing `registries.yaml`.

## Set up PostgreSQL

The `postgres/` folder contains a simple single-instance PostgreSQL deployment for homelab use.

### Files in this folder

| File | Purpose |
|---|---|
| `postgres/namespace.yaml` | Creates the dedicated `postgres` namespace |
| `postgres/pg-config.yaml` | Sets `POSTGRES_DB`, `POSTGRES_USER`, and `PGDATA` |
| `postgres/pg-secret.yaml` | Stores `POSTGRES_PASSWORD` |
| `postgres/pg-deployment.yaml` | Runs the PostgreSQL container and mounts the PVC |
| `postgres/pg-service.yaml` | Exposes PostgreSQL on port `5432` |
| `postgres/pg-loadbalancer.yaml` | Optional `LoadBalancer` service for outside-cluster access |
| `postgres/pg-pv-nfs.yaml` | Static NFS CSI-backed `PersistentVolume` |
| `postgres/pg-pv-claim.yaml` | `PersistentVolumeClaim` used by the deployment |
| `postgres/pg-pv-local.yaml` | Alternative local-disk PV/PVC example for a single node |

### Storage options

This repo currently defaults to the NFS CSI path:

- `postgres/pg-pv-nfs.yaml`
- `postgres/pg-pv-claim.yaml`
- `postgres/pg-deployment.yaml`

If you want to use local disk on one node instead, `postgres/pg-pv-local.yaml` uses the same claim name as the deployment. Apply either the NFS-backed PVC or the local PVC, not both.

### Recommended apply order

```bash
kubectl apply -f postgres/namespace.yaml
kubectl apply -f postgres/pg-config.yaml
kubectl apply -f postgres/pg-secret.yaml
kubectl apply -f postgres/pg-pv-nfs.yaml
kubectl apply -f postgres/pg-pv-claim.yaml
kubectl apply -f postgres/pg-deployment.yaml
kubectl apply -f postgres/pg-service.yaml
```

### Important notes

- The deployment pins `postgres:16.13`.
- The deployment includes `startupProbe`, `readinessProbe`, and `livenessProbe`.
- NFS-backed PostgreSQL is fine for learning, but local SSD-backed storage is the better default for anything performance-sensitive.
- Write down your backup and restore path instead of leaving recovery implicit.

## Optional infrastructure

These tools extend the cluster with logging, search, and build capabilities. They are not required for basic cluster operation.

### OpenSearch

This repo includes an OpenSearch setup in `opensearch/` with Dashboards already enabled.

Pinned versions as of March 14, 2026:

| Chart | Version |
|---|---|
| `opensearch-operator` | `2.8.2` |
| `opensearch-cluster` | `3.2.2` |

Install flow:

```bash
helm repo add opensearch https://opensearch-project.github.io/opensearch-k8s-operator/
helm repo update
kubectl create namespace opensearch
helm upgrade --install opensearch-operator opensearch/opensearch-operator \
  --namespace opensearch \
  --version 2.8.2
helm upgrade --install opensearch-cluster opensearch/opensearch-cluster \
  --namespace opensearch \
  --version 3.2.2 \
  -f opensearch/values.yaml
```

For more detail, see `opensearch/README.md`.

### Kaniko

Kaniko builds container images inside a Kubernetes pod instead of requiring Docker to be installed on the node.

> [!WARNING]
> The original `GoogleContainerTools/kaniko` repository was archived in June 2025. Use the maintained fork at [`osscontainertools/kaniko`](https://github.com/osscontainertools/kaniko) instead.

Quick start:

```bash
kubectl apply -f kaniko/namespace.yaml
kubectl apply -f kaniko/kaniko.yaml
kubectl logs -n kaniko -f job/kaniko-build
```

For full details, see `kaniko/README.md`.

### GitOps with Flux or ArgoCD

GitOps is a deployment approach where a controller running inside the cluster continuously reconciles what is deployed against a Git repository.

Quick Flux bootstrap example:

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
flux bootstrap github \
  --owner=<your-github-username> \
  --repository=<your-repo-name> \
  --branch=main \
  --path=clusters/my-cluster \
  --personal
```

## Cluster operations

See [../OPERATIONS.md](../OPERATIONS.md) for full runbooks on K3s upgrades, backups, restores, and Velero-based application backup.

## Next step

Once the shared platform services are in place, continue with [Example applications](example-applications.md).
