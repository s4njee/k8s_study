# Kubernetes Study Guide

## Introduction

Kubernetes, often shortened to `k8s`, is an open source platform for running containerized applications across one or more machines. Instead of manually logging into servers to start, stop, or update your apps, you describe what you want in configuration files and Kubernetes handles the rest — scheduling work, restarting crashed containers, routing traffic, and managing storage.

Teams use Kubernetes to run web apps, APIs, background workers, CI/CD pipelines, and databases in a repeatable way. It is especially useful when you want the same setup to work on a laptop, a homelab, on-prem servers, and public cloud environments.

Kubernetes is widely used in production by large organizations, including [Spotify](https://kubernetes.io/case-studies/spotify/), [The New York Times](https://kubernetes.io/case-studies/newyorktimes/index.html), [Box](https://kubernetes.io/case-studies/box/), and [Pinterest](https://kubernetes.io/case-studies/pinterest/).

This repo is a hands-on study guide. You will learn Kubernetes by building a small but useful cluster with [K3s](https://k3s.io) (a lightweight Kubernetes distribution), network storage, HTTPS ingress (a way to expose apps to the internet), monitoring dashboards, a private container image registry, and a few example workloads.

## Table of contents

- [Introduction](#introduction)
- [Quickstart](#quickstart)
- [First-cluster defaults](#first-cluster-defaults)
- [Install k3s](#install-k3s)
- [Install Headlamp](#install-headlamp)
- [Next steps after the basics work](#next-steps-after-the-basics-work)
- [Secrets guide](SECRETS.md)
- [Workloads guide](WORKLOADS.md)
- [Security guide](SECURITY.md)
- [Troubleshooting guide](TROUBLESHOOTING.md)
- [Install the NFS CSI driver](#install-the-nfs-csi-driver)
- [Install Actions Runner Controller](#install-actions-runner-controller)
- [Install kube-prometheus-stack](#install-kube-prometheus-stack)
- [Install Grafana Dashboards for Kubernetes](#install-grafana-dashboards-for-kubernetes)
- [Set up Traefik ingress](#set-up-traefik-ingress)
- [Set up cert-manager for the Docker Registry certificate](#set-up-cert-manager-for-the-docker-registry-certificate)
- [Set up Docker Registry](#set-up-docker-registry)
- [Set up PostgreSQL](#set-up-postgresql)
- [Optional components](#optional-components)
  - [OpenSearch](#opensearch)
  - [Kaniko](#kaniko)
- [After install](#after-install)
  - [Immich](#immich)
  - [Hoarder](#hoarder)
  - [ruTorrent](#rutorrent)
- [Legacy PV examples](#legacy-pv-examples)

---

## Quickstart

If you're new to Kubernetes, don't try to absorb this whole repo in one sitting.

The shortest path to something useful:

1. install K3s
2. copy your kubeconfig locally
3. run `kubectl get nodes` and confirm it works
4. deploy a workload from [`workloads/`](workloads)
5. come back for storage, ingress, security, and add-ons once the basics click

The goal is **a working cluster you understand**, not a perfectly hardened platform.

## First-cluster defaults

Keep the day-one rules small:

- **Pin versions** when you want a repeatable setup — K3s, Helm charts, image tags, downloaded manifests.
- **Give each app its own namespace** so resources stay organized.
- **Keep secrets out of git**, even in a lab repo.
- **Preview before you apply** — `kubectl diff` or `kubectl apply --dry-run=server`.

That's enough to get moving.

Once things feel comfortable, layer in the stronger defaults:

- resource requests and limits
- health probes (readiness, liveness, startup)
- standard `app.kubernetes.io/*` labels
- graceful shutdown via `terminationGracePeriodSeconds`
- replica spreading with `topologySpreadConstraints` or anti-affinity
- namespace guardrails like `LimitRange` and `ResourceQuota`
- backup and recovery planning for stateful apps

For deeper dives once the basics work, see [`SECRETS.md`](SECRETS.md), [`WORKLOADS.md`](WORKLOADS.md), [`SECURITY.md`](SECURITY.md), and [`TROUBLESHOOTING.md`](TROUBLESHOOTING.md).

---

## Install k3s

### Server installation

Install K3s on the server node:

```bash
curl -sfL https://get.k3s.io | sh -
```

For a repeatable lab build, pin the K3s version instead of taking whatever the installer serves that day:

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=<tested_k3s_version> sh -
```

If you want the API server certificate to include a public DNS name or IP address, install K3s with `--tls-san`:

```bash
curl -sfL https://get.k3s.io | sh -s - --tls-san <server_dns_or_ip>
```

You can repeat `--tls-san` if you want to allow multiple hostnames or IP addresses:

```bash
curl -sfL https://get.k3s.io | sh -s - \
  --tls-san <server_dns_name> \
  --tls-san <server_ip_address>
```

### Node installation

On the server node, get the node token:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

On each worker node, install K3s and join it to the server:

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<server_ip_or_dns>:6443 K3S_TOKEN=<token_from_above> sh -
```

If you pinned the server version, pin the workers to the same K3s version:

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=<tested_k3s_version> \
  K3S_URL=https://<server_ip_or_dns>:6443 \
  K3S_TOKEN=<token_from_above> sh -
```

### Save the kubeconfig locally

On the server node, K3s writes a **kubeconfig** file to `/etc/rancher/k3s/k3s.yaml`. A kubeconfig is a credentials file that tells `kubectl` (the Kubernetes command-line tool) how to connect to your cluster.

Copy that file to your local machine and save it as `~/.kube/config` so you can manage the cluster from your own computer. The file is usually only readable by `root`, so use one of these approaches to copy it.

**If your SSH user has passwordless `sudo`**, stream the file over SSH:

```bash
mkdir -p ~/.kube
ssh <user>@<server_ip_or_dns> "sudo cat /etc/rancher/k3s/k3s.yaml" > ~/.kube/config
chmod 600 ~/.kube/config
```

**If you prefer `scp`**, first copy it to your user's home directory on the server with `sudo`, then download it:

```bash
ssh <user>@<server_ip_or_dns> "sudo cp /etc/rancher/k3s/k3s.yaml /home/<user>/k3s.yaml && sudo chown <user>:<user> /home/<user>/k3s.yaml"
mkdir -p ~/.kube
scp <user>@<server_ip_or_dns>:/home/<user>/k3s.yaml ~/.kube/config
chmod 600 ~/.kube/config
```

> [!IMPORTANT]
> After copying it, update the `server:` value inside `~/.kube/config` if it still points to `https://127.0.0.1:6443`. Replace it with the same DNS name or IP address you used for `--tls-san`.

---

## Install Headlamp

Optional — if you just want a working cluster, skip this for now and come back later.

Headlamp is a web UI for Kubernetes. It lets you browse resources, view logs, and manage workloads through a graphical interface instead of the command line. It's a better default than the older Kubernetes Dashboard for most homelab use.

### Option 1: Use the desktop app

If you already copied your kubeconfig to `~/.kube/config`, you can use Headlamp without installing anything in the cluster.

On macOS:

```bash
brew install --cask --no-quarantine headlamp
```

Then open Headlamp and point it at your default kubeconfig, or launch it with a specific config file:

```bash
KUBECONFIG=~/.kube/config /Applications/Headlamp.app/Contents/MacOS/Headlamp
```

### Option 2: Install Headlamp in the cluster

Add the Helm repo and install Headlamp in its own namespace:

```bash
helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/
helm repo update
helm upgrade --install my-headlamp headlamp/headlamp \
  --namespace headlamp \
  --create-namespace \
  --wait
```

You can access it locally with **port-forwarding**, which creates a temporary tunnel from your computer to a service inside the cluster:

```bash
kubectl -n headlamp port-forward svc/my-headlamp 4466:80
```

While that command is running, open [http://localhost:4466](http://localhost:4466) in your browser.

### Optional: Create an admin token for Headlamp

Prefer the desktop app with your kubeconfig when possible. If you want a simple admin login for a homelab cluster, create a service account and token, but treat it as a short-lived bootstrap credential because `cluster-admin` is intentionally very broad:

```bash
kubectl -n headlamp create serviceaccount headlamp-admin
kubectl create clusterrolebinding headlamp-admin \
  --serviceaccount=headlamp:headlamp-admin \
  --clusterrole=cluster-admin
kubectl create token headlamp-admin -n headlamp
```

Paste that token into Headlamp when prompted.

---

## Next steps after the basics work

Everything below this point is useful, but none of it is required to have a working cluster.

Think of these as follow-on building blocks — pick what you need:

- storage (NFS CSI driver)
- ingress and TLS (Traefik + cert-manager)
- monitoring (Prometheus + Grafana)
- private registries (Docker Registry)
- CI/CD runners (Actions Runner Controller)
- stateful apps and homelab add-ons

You don't need all of them before your cluster is "real."

## Install the NFS CSI driver

If you have an NFS file server on your network (a common setup in homelabs), you can use it to store data for your Kubernetes apps. The **NFS CSI driver** is a plugin that teaches Kubernetes how to mount NFS shares as persistent volumes (storage that survives pod restarts).

Add the Helm repo and install the driver in `kube-system`:

```bash
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm repo update
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs --namespace kube-system
```

If you want to pin a tested chart version, the current upstream release as of March 14, 2026 is **`4.13.1`**:

```bash
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
  --namespace kube-system \
  --version 4.13.1
```

> [!NOTE]
> Make sure each Kubernetes node has the NFS client packages installed so it can mount NFS shares.

After the driver is installed, you can create a **PersistentVolume** (PV) and **PersistentVolumeClaim** (PVC) that reference it. A PV describes a piece of storage (like an NFS share), and a PVC is a request from an application to use that storage. In your manifest files, set:

```yaml
storageClassName: nfs-csi
csi:
  driver: nfs.csi.k8s.io
```

An example static NFS CSI-backed Postgres volume is in:

- `postgres/pg-pv-nfs.yaml`
- `postgres/pg-pv-claim.yaml`
- `postgres/pg-deployment.yaml`

---

## Install Actions Runner Controller

Actions Runner Controller (ARC) lets you run GitHub Actions CI/CD jobs on your own Kubernetes cluster instead of using GitHub's hosted runners. This is useful when your builds need access to private network resources or when you want more control over the build environment.

GitHub's current recommendation is to use the **runner scale set** approach, which installs two Helm charts: one for the controller and one for each set of runners.

If you want to pin a tested chart version, the latest upstream release as of March 14, 2026 is **`0.13.1`** (release tag `gha-runner-scale-set-0.13.1` published on December 23, 2025).

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

The runners need permission to talk to the GitHub API. GitHub recommends storing these credentials in a Kubernetes Secret (a resource designed to hold sensitive data) rather than passing them on the command line.

For organization or repository runners, a **GitHub App** is the recommended authentication method. Create the secret in the runner namespace:

```bash
export ARC_RUNNERS_NS=arc-runners
kubectl create namespace "${ARC_RUNNERS_NS}"
kubectl create secret generic arc-github-app \
  --namespace "${ARC_RUNNERS_NS}" \
  --from-literal=github_app_id='<app_id>' \
  --from-literal=github_app_installation_id='<installation_id>' \
  --from-file=github_app_private_key=/path/to/private-key.pem
```

If you prefer a **token-based** setup instead, create:

```bash
kubectl create secret generic arc-github-token \
  --namespace "${ARC_RUNNERS_NS}" \
  --from-literal=github_token='<pat_or_fine_grained_pat>'
```

If you use a classic PAT (Personal Access Token), give it `repo` scope for repository runners or `admin:org` scope for organization runners. Fine-grained PATs (GitHub's newer, more restrictive token type) are also supported.

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

The controller namespace (`arc-systems`) should show a running controller manager pod. The runner namespace (`arc-runners`) should show a listener pod that watches for new workflow jobs.

### 5. Use the runner scale set in a GitHub Actions workflow

In your workflow YAML file, set `runs-on` to the Helm release name you chose for the runner scale set. This tells GitHub to send the job to your self-hosted runners instead of GitHub's cloud runners:

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

**Further reading:**

- [Quickstart](https://docs.github.com/en/actions/tutorials/use-actions-runner-controller/quickstart)
- [Deploying runner scale sets](https://docs.github.com/en/actions/tutorials/use-actions-runner-controller/deploy-runner-scale-sets)
- [Authentication](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/authenticating-to-the-github-api)

---

## Install kube-prometheus-stack

This installs a complete monitoring stack in one step: **Prometheus** (collects metrics from your cluster), **Grafana** (displays dashboards and graphs), **Alertmanager** (sends alerts when something goes wrong), and supporting components like kube-state-metrics and node-exporter.

The `kube-prometheus-stack` Helm chart bundles all of these together with Grafana already configured to read data from Prometheus.

If you want to pin a tested chart version, the latest release as of March 14, 2026 is **`82.10.3`** (app version `v0.89.0`, published March 10, 2026).

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

Update the hostnames and storage settings in that file before installing.

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

You should see pods, services, and other resources for Grafana, Prometheus, Alertmanager, and the supporting exporters in the `monitoring` namespace.

### 6. Get the Grafana admin password

The chart generates a random admin password and stores it in a Kubernetes Secret. This command reads that secret and decodes it:

```bash
kubectl get secret --namespace monitoring monitoring-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

### 7. Access Grafana

If you set up ingress in the values file, you can open the Grafana hostname you configured in `kube-prometheus-stack/values.yaml` directly in your browser.

If you just want to try it locally first, use port-forwarding to create a tunnel:

```bash
kubectl port-forward -n monitoring svc/monitoring-stack-grafana 3000:80
```

Then open [http://127.0.0.1:3000](http://127.0.0.1:3000). Log in with username `admin` and the decoded password from the previous step.

### 8. Access Prometheus

If ingress is configured, open the Prometheus hostname from `kube-prometheus-stack/values.yaml`.

If you want local access instead:

```bash
kubectl port-forward -n monitoring svc/monitoring-stack-kube-prom-prometheus 9090:9090
```

Then open [http://127.0.0.1:9090](http://127.0.0.1:9090).

**Further reading:**

- [Chart repo](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Values reference](https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/values.yaml)

---

## Install Grafana Dashboards for Kubernetes

The `grafana-dashboards-kubernetes/` folder contains a set of pre-made Grafana dashboards that show CPU, memory, network, and pod-level metrics for your cluster. They are managed with **Kustomize** (a tool that generates Kubernetes resources from templates).

The dashboards are stored as `ConfigMap` resources (key-value data objects in Kubernetes). Grafana's **sidecar** (a helper container that runs alongside Grafana) automatically detects ConfigMaps with the `grafana_dashboard: "1"` label and loads them as dashboards.

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

These dashboards are designed to work with the Grafana instance from the [kube-prometheus-stack](#install-kube-prometheus-stack) section above. If you followed that section, the sidecar is already enabled and watching for dashboard ConfigMaps — no extra configuration needed.

### Deploy the dashboards

```bash
kubectl apply -k grafana-dashboards-kubernetes/
```

The `kustomization.yaml` sets the `grafana_dashboard: "1"` label and the `grafana_folder: "Kubernetes"` annotation on every generated ConfigMap, so Grafana will group them under a "Kubernetes" folder.

> [!TIP]
> If your Grafana is in a namespace other than `default`, uncomment and set the `namespace` field in `grafana-dashboards-kubernetes/kustomization.yaml`.

### Verify

After applying, check that the ConfigMaps exist:

```bash
kubectl get configmaps -l grafana_dashboard=1
```

Then open Grafana and look for the "Kubernetes" folder.

---

## Set up Traefik ingress

**Traefik** is a reverse proxy that acts as the front door to your cluster. When someone visits `https://myapp.example.com`, Traefik receives the request and routes it to the correct service inside Kubernetes based on rules you define (called **Ingress** resources).

K3s includes Traefik by default unless you installed K3s with `--disable=traefik`. On K3s, Traefik is typically exposed on ports `80` (HTTP) and `443` (HTTPS) by the built-in ServiceLB.

### 1. Check whether Traefik is already running

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=traefik
kubectl get svc -n kube-system traefik
kubectl get ingressclass
```

You should see a `traefik` service in `kube-system` and an ingress class named `traefik`.

### 2. If Traefik was disabled, install your ingress controller first

If you started K3s with `--disable=traefik`, install Traefik or another ingress controller before continuing with the certificate setup. The cert-manager manifests in this repo expect an ingress class named `traefik`.

### 3. Make sure ports 80 and 443 reach the cluster

Let's Encrypt (the free certificate authority used later for HTTPS) needs to reach your cluster on ports `80` and `443` to verify that you own the domain name. This is called **HTTP-01 validation**.

- If this is a **public** setup, point your router or firewall at the K3s node that is serving Traefik
- If this is a **LAN-only homelab**, make sure the hostname resolves to a reachable node IP
- If multiple nodes exist, remember that K3s ServiceLB may advertise more than one node IP unless you restrict the Traefik service to specific nodes

### 4. Optional: customize the packaged Traefik install

> [!CAUTION]
> Do not edit `/var/lib/rancher/k3s/server/manifests/traefik.yaml` directly — K3s will replace it.

If you need to customize Traefik, create a `HelmChartConfig` on the server node:

```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    service:
      spec:
        externalTrafficPolicy: Local
```

Save it to `/var/lib/rancher/k3s/server/manifests/traefik-config.yaml`, then wait for K3s to reconcile the Traefik deployment.

---

## Set up cert-manager for the Docker Registry certificate

**cert-manager** is a Kubernetes add-on that automatically obtains and renews TLS certificates (the files that enable HTTPS). This repo uses cert-manager with **Let's Encrypt**, a free certificate authority, to issue the HTTPS certificate for the Docker registry.

Even though the folder is named `certbox`, the manifests inside are for cert-manager, not the older Certbot tool.

### 1. Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

The `latest/download` URL is convenient for a one-off lab, but for a repeatable guide prefer a release-tagged URL after you verify the version you want to teach:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/<tested_cert_manager_version>/cert-manager.yaml
```

Wait for the pods to become ready:

```bash
kubectl get pods -n cert-manager
```

### 2. Create a Let's Encrypt issuer

An **issuer** tells cert-manager where to get certificates. Update the email address and ingress class in the issuer manifest before applying it. Let's Encrypt will send renewal reminders to that email.

For **testing**, start with staging:

```bash
kubectl apply -f certbox/letsencrypt-staging.yaml
```

For **production**, use:

```bash
kubectl apply -f certbox/letsencrypt-prod.yaml
```

The production issuer in `certbox/letsencrypt-prod.yaml` creates a `ClusterIssuer` (an issuer available to all namespaces) named `letsencrypt-prod`. The certificate request in `docker-registry.helm/certificate.yaml` references this issuer by name.

### 3. Make the registry hostname reachable

Before requesting the certificate, point your registry DNS name at your cluster ingress:

1. Create a DNS `A` record for your registry hostname (e.g. `registry.example.com`)
2. Point it at the public IP or reachable LAN IP for your K3s ingress
3. Make sure ports `80` and `443` can reach Traefik so the HTTP-01 challenge can succeed

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

When the certificate is ready, cert-manager will create the TLS secret referenced by the registry chart.

### 5. Install the registry

After the certificate secret exists, install the Docker registry chart.

> [!TIP]
> If you want to test issuance without hitting Let's Encrypt production limits, temporarily change the certificate issuer to `letsencrypt-staging`, verify that it works, then switch back to `letsencrypt-prod`.

---

## Set up Docker Registry

A **container image registry** is where you store the Docker images that Kubernetes pulls when it starts your apps. If you only need a registry for personal builds, a managed service like GitHub Container Registry (GHCR) is usually easier. If you want a private registry running inside your own cluster, this repo includes a Helm chart for one in `docker-registry.helm/`.

> [!NOTE]
> If you need a full registry platform with UI, RBAC, vulnerability scanning, replication, and robot accounts, **Harbor** is the more common 2026 choice. If you want a lighter modern OCI-native registry with more built-in features than plain Distribution, **zot** is also worth a look. This repo keeps CNCF Distribution because it is still the simplest option for a small homelab registry.

### Recommended setup

Use HTTPS (TLS) and password authentication. The registry uses an **htpasswd** file (a simple username/password file format from Apache) for authentication. Instead of putting the password directly in `values.yaml`, pass the file to Helm separately with `--set-file` so it stays out of version control.

For a fresh install, keep the Helm release, `Certificate`, and TLS secret together in a dedicated namespace such as `registry` instead of `kube-system`.

Create an `htpasswd` file locally:

```bash
mkdir -p auth
docker run --rm --entrypoint htpasswd httpd:2 -Bbn <registry_user> <registry_password> > auth/htpasswd
```

Review and update these files before installing:

- `docker-registry.helm/values.yaml`
- `docker-registry.helm/certificate.yaml`

Make sure the hostname, ingress class, TLS secret name, issuer, and persistence settings match your cluster. The chart's built-in ingress is configured through `values.yaml`.

Then install the chart from this repo:

```bash
helm upgrade --install private-registry ./docker-registry.helm \
  --namespace registry \
  --create-namespace \
  --set persistence.enabled=true \
  --set-file secrets.htpasswd=auth/htpasswd
```

This keeps the auth file out of the shell history and avoids embedding credentials directly in the README command.

### Log in and push images

After DNS and TLS are working, log in from your workstation:

```bash
docker login <registry_hostname>
```

Then tag and push an image:

```bash
docker tag my-image:latest <registry_hostname>/my-image:latest
docker push <registry_hostname>/my-image:latest
```

### Let Kubernetes pull from the private registry

When Kubernetes needs to pull an image from your private registry, it needs credentials. Create an **imagePullSecret** (a Secret that contains registry login details) in each namespace where your apps run:

```bash
kubectl create secret docker-registry regcred \
  --namespace <namespace> \
  --docker-server=<registry_hostname> \
  --docker-username=<registry_user> \
  --docker-password=<registry_password>
```

Then add `imagePullSecrets: [{name: regcred}]` to your Pod, Deployment, or ServiceAccount spec so Kubernetes knows to use those credentials.

### Configure K3s to trust your registry

If your registry uses a certificate from a well-known authority (like Let's Encrypt), K3s will trust it automatically.

If your registry uses a self-signed certificate or a private certificate authority (CA), you need to tell K3s to trust it. You can also configure node-wide registry credentials here so you do not need imagePullSecrets. Add an entry to `/etc/rancher/k3s/registries.yaml` on each node:

**Example with TLS and node-wide auth:**

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

**Example without TLS** (intentionally running without TLS, use `http://` explicitly):

```yaml
mirrors:
  "<registry_hostname>:5000":
    endpoint:
      - "http://<registry_hostname>:5000"
```

> [!IMPORTANT]
> Restart K3s after changing `registries.yaml`.

---

## Set up PostgreSQL

The `postgres/` folder contains a simple single-instance PostgreSQL deployment for homelab use.

### Files in this folder

| File | Purpose |
|---|---|
| `postgres/namespace.yaml` | Creates the dedicated `postgres` namespace |
| `postgres/pg-config.yaml` | Sets `POSTGRES_DB`, `POSTGRES_USER`, and `PGDATA` |
| `postgres/pg-secret.yaml` | Stores `POSTGRES_PASSWORD` |
| `postgres/pg-deployment.yaml` | Runs the PostgreSQL container and mounts the PVC at `/var/lib/postgresql/data` |
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

That setup assumes:

- the NFS CSI driver is already installed
- the NFS server and share in `postgres/pg-pv-nfs.yaml` are reachable from your cluster nodes
- you want the deployment to use the PVC named `postgres-pv-claim`

If you want to use local disk on one node instead, `postgres/pg-pv-local.yaml` now uses the same `postgres-pv-claim` claim name as the deployment. Apply either the NFS-backed PVC or the local PVC, not both.

For a fresh install, put the namespaced PostgreSQL resources in a dedicated namespace such as `postgres`. Keep the `PersistentVolume` cluster-scoped, but create the `ConfigMap`, `Secret`, `PersistentVolumeClaim`, `Deployment`, and `Service` in the same namespace as the workload.

### Recommended apply order

For the NFS CSI-backed setup:

```bash
kubectl apply -f postgres/namespace.yaml
kubectl apply -f postgres/pg-config.yaml
kubectl apply -f postgres/pg-secret.yaml
kubectl apply -f postgres/pg-pv-nfs.yaml
kubectl apply -f postgres/pg-pv-claim.yaml
kubectl apply -f postgres/pg-deployment.yaml
kubectl apply -f postgres/pg-service.yaml
```

If you want the database reachable from outside the cluster, also apply:

```bash
kubectl apply -f postgres/pg-loadbalancer.yaml
```

Keep `postgres/pg-service.yaml` for in-cluster access. Add `postgres/pg-loadbalancer.yaml` only if you also want outside-cluster access. On K3s, `type: LoadBalancer` is usually handled by the built-in ServiceLB.

### Important notes

> [!WARNING]
> `postgres/pg-secret.yaml` still stores the password in plaintext in git. That is better than a `ConfigMap`, but for anything more serious you should generate it outside the repo or use something like External Secrets, Sealed Secrets, or SOPS.

- The deployment now pins `postgres:16.13`. Keep that version intentional when you upgrade so you do not accidentally change database major versions.
- The deployment includes `startupProbe`, `readinessProbe`, and `livenessProbe`. Adjust the thresholds if your storage or initialization path is slower than the sample defaults.
- The deployment includes baseline CPU and memory requests/limits. Tune them for your workload instead of treating the sample values as universal.
- NFS-backed PostgreSQL is fine for learning, but local SSD-backed storage or a database-oriented storage class is the better default for anything performance-sensitive.
- Write down the backup and restore path you expect to use. For a study guide, even a simple `pg_dump` / `pg_restore` example is better than leaving recovery implicit.
- The checked-in manifests now assume a dedicated `postgres` namespace. If you choose a different namespace, keep the `ConfigMap`, `Secret`, `PersistentVolumeClaim`, `Deployment`, and `Service` together.

---

## Optional components

These tools extend the cluster with logging, search, and build capabilities. They are not required for basic cluster operation but are commonly used alongside it.

### OpenSearch

**OpenSearch** is an open source search and analytics engine, commonly used for log aggregation and full-text search. **OpenSearch Dashboards** (the open source equivalent of Kibana) provides a web UI for querying logs and building visualizations.

This repo includes an OpenSearch setup in `opensearch/` with Dashboards already enabled.

The setup uses the official Helm operator charts. An **operator** is a Kubernetes-native way to manage complex applications — it watches for custom resources you create and handles the details of deploying and configuring the application. The pinned versions as of March 14, 2026:

| Chart | Version |
|---|---|
| `opensearch-operator` | `2.8.2` |
| `opensearch-cluster` | `3.2.2` |

#### 1. Add the Helm repo

```bash
helm repo add opensearch https://opensearch-project.github.io/opensearch-k8s-operator/
helm repo update
```

#### 2. Install the operator

```bash
kubectl create namespace opensearch
helm upgrade --install opensearch-operator opensearch/opensearch-operator \
  --namespace opensearch \
  --version 2.8.2
```

#### 3. Review the local values file

The starter values file is `opensearch/values.yaml`. It already enables:

- one OpenSearch node pool
- one OpenSearch Dashboards replica
- Traefik ingress for Dashboards

Before installing, update at least:

- the Dashboards hostname
- storage size
- CPU and memory requests

#### 4. Install the cluster

```bash
helm upgrade --install opensearch-cluster opensearch/opensearch-cluster \
  --namespace opensearch \
  --version 3.2.2 \
  -f opensearch/values.yaml
```

#### 5. Access OpenSearch Dashboards

The local values file exposes Dashboards through ingress. Update the hostname in `opensearch/values.yaml`, then open that hostname in your browser after the ingress is ready.

If you want local access first, port-forward the Dashboards service:

```bash
kubectl port-forward -n opensearch svc/opensearch-dashboards 5601:5601
```

Then open [http://127.0.0.1:5601](http://127.0.0.1:5601).

For more detail, see `opensearch/README.md`.

---

### Kaniko

**Kaniko** builds container images (like `docker build`) but runs inside a Kubernetes pod instead of requiring Docker to be installed. This is useful for CI/CD pipelines where you want to build images on your cluster. The `kaniko/` folder contains a ready-to-use Kubernetes Job manifest and a dedicated README.

#### Important: upstream repo change

> [!WARNING]
> The original `GoogleContainerTools/kaniko` repository was archived in June 2025. Use the maintained fork at [`osscontainertools/kaniko`](https://github.com/osscontainertools/kaniko) instead.

The manifest in this repo already references the new image:

```text
ghcr.io/osscontainertools/kaniko:v1.27.0
```

#### Quick start

1. Create a registry auth secret (see `kaniko/README.md` for details)
2. Edit `kaniko/kaniko.yaml` to set your build context, Dockerfile path, and destination image
3. Run the build:

```bash
kubectl apply -f kaniko/namespace.yaml
kubectl apply -f kaniko/kaniko.yaml
kubectl logs -n kaniko -f job/kaniko-build
```

For full details, see `kaniko/README.md`.

---

## After install

These are end-user applications that run on top of the cluster. None of them are required for the cluster to function — they are example workloads you can deploy once the infrastructure above is in place.

### Immich

[Immich](https://immich.app) is a self-hosted photo and video management app (similar to Google Photos). The project provides an official Helm chart for Kubernetes deployments.

If you want to pin a tested chart version, the latest chart release as of March 14, 2026 is **`0.10.3`**, published on November 14, 2025. The latest Immich app release as of March 14, 2026 is **`v2.5.6`**, published on February 10, 2026.

> [!IMPORTANT]
> The Helm chart version and the Immich app version are released independently. The chart does not automatically set the app version for you — you need to specify `image.tag` in your values file to control which version of Immich actually runs.

#### 1. Add the Immich Helm repo

```bash
helm repo add immich https://immich-app.github.io/immich-charts
helm repo update
```

You can confirm the chart version with:

```bash
helm search repo immich/immich
```

#### 2. Create the Immich namespace and storage

Immich stores your photos on a persistent volume. The chart expects you to create the storage ahead of time. This repo includes a sample NFS-backed PV and PVC in `immich-charts/rt-pvs.yaml`.

Apply or adapt that file first:

```bash
kubectl create namespace immich
kubectl apply -f immich-charts/rt-pvs.yaml
```

Update the NFS server, paths, storage class, and capacity in that file before applying it.

#### 3. Create a local values override

Use a small override file such as `immich-charts/values-local.yaml`:

```yaml
image:
  tag: v2.5.6

immich:
  persistence:
    library:
      existingClaim: immich-pvc

postgresql:
  enabled: true

redis:
  enabled: true

server:
  ingress:
    main:
      enabled: true
      hosts:
        - host: photos.example.com
          paths:
            - path: /
```

This keeps the app pinned to the current Immich release while still using the Helm chart for the rest of the stack.

#### 4. Install Immich

```bash
helm upgrade --install immich immich/immich \
  --namespace immich \
  --create-namespace \
  --version 0.10.3 \
  -f immich-charts/values-local.yaml
```

#### 5. Verify the install

```bash
helm list -n immich
kubectl get pods -n immich
kubectl get svc -n immich
```

If everything comes up cleanly, the main app should be reachable on the `immich-server` service or through your configured ingress.

#### 6. Optional ingress note for this repo

This repo already contains a sample ingress manifest at `immich-charts/gingress.yaml`, but the chart can also manage ingress directly through `values-local.yaml`. Prefer one approach or the other so you do not create overlapping ingress definitions.

**Further reading:**

- [Kubernetes install docs](https://docs.immich.app/install/kubernetes)
- [Chart repo](https://github.com/immich-app/immich-charts)
- [Immich releases](https://github.com/immich-app/immich/releases)

---

### Hoarder

[Hoarder](https://docs.karakeep.app) (now officially renamed to **Karakeep**) is a self-hosted bookmarking and read-it-later app with optional AI-powered tagging. The project moved from `hoarder-app/hoarder` to `karakeep-app/karakeep`, but this repo still uses the `hoarder/` folder name.

Unlike most other apps in this guide, Karakeep uses **Kustomize manifests** (plain YAML files processed by `kustomize build`) rather than a Helm chart. As of March 14, 2026, the latest release is **`v0.31.0`** (published February 22, 2026).

#### 1. Use the Kubernetes manifests in the repo

The local Kubernetes setup lives in `hoarder/kubernetes/`.

Current upstream docs still follow the same general model:

1. Clone or copy the `kubernetes/` directory
2. Populate `.env` and `.secrets`
3. Run `make deploy`

#### 2. Create local env and secrets files

Use the sample files:

```bash
cp hoarder/kubernetes/.env.sample hoarder/kubernetes/.env
cp hoarder/kubernetes/.secrets.sample hoarder/kubernetes/.secrets
```

Then edit them for your environment. At minimum, set:

| Variable | Value |
|---|---|
| `NEXTAUTH_URL` | Your actual URL |
| `KARAKEEP_VERSION` | `v0.31.0` (to pin the release instead of following `release`) |
| `NEXTAUTH_SECRET` | Generate with `openssl rand -base64 36` |
| `MEILI_MASTER_KEY` | Generate with `openssl rand -base64 36` |
| `NEXT_PUBLIC_SECRET` | Generate with `openssl rand -base64 36` |

If you want AI tagging, also set:

```bash
OPENAI_API_KEY=<your_key>
```

#### 3. Check storage and service exposure

The manifests in `hoarder/kubernetes/` include PVCs and expose the web service by default. Review these before deploying:

- `hoarder/kubernetes/data-pvc.yaml`
- `hoarder/kubernetes/meilisearch-pvc.yaml`
- `hoarder/kubernetes/web-service.yaml`

If you plan to expose Hoarder through ingress instead of a `LoadBalancer`, change the web service to `ClusterIP` and add your ingress manifest.

#### 4. Deploy Hoarder

```bash
cd hoarder/kubernetes
make deploy
```

#### 5. Verify the install

```bash
kubectl get all -n hoarder
```

If you keep the default `LoadBalancer` service, get the external address with:

```bash
kubectl get svc -n hoarder
```

Then browse to `http://<loadbalancer-ip>:3000`.

#### 6. Important note about the local manifests

> [!WARNING]
> The local `hoarder/kubernetes/` directory is older than the current upstream Karakeep manifests.

Current upstream uses:

| What changed | Old value | New value |
|---|---|---|
| Namespace | `hoarder` | `karakeep` |
| Version variable | `HOARDER_VERSION` | `KARAKEEP_VERSION` |
| Secret management | inline | separate `.secrets` file + `secretGenerator` |
| Web image | — | `ghcr.io/karakeep-app/karakeep` |

For a new install, use the README guidance here and the current Karakeep docs as the source of truth even though the local folder is still named `hoarder/`.

**Further reading:**

- [Kubernetes install docs](https://docs.karakeep.app/installation/kubernetes)
- [Karakeep releases](https://github.com/karakeep-app/karakeep/releases)

---

### ruTorrent

[ruTorrent](https://github.com/Novik/ruTorrent) is a web-based front end for the rTorrent BitTorrent client. The `rutorrent/` folder contains Kubernetes manifests for running it. There are two layout variations:

| Variant | Path | Description |
|---|---|---|
| All-in-one | `rutorrent/rt-deployment.yaml` | Deployment, Services, ConfigMap, and NFS-backed PVs/PVCs in a single file |
| Split-file | `rutorrent/main/` | Separate manifests: `rt-main-deploy.yaml`, `rt-service.yaml`, `rt-lb.yaml`, `rt-config.yaml`, `rt-pvs.yaml` |

#### Before deploying

Both variations contain values that are specific to the original author's environment — you will need to update them before deploying:

- the container `image` to match your registry
- the NFS `server` and `path` values in the PV definitions
- the `LoadBalancer` IP address (if using `status.loadBalancer.ingress`)
- resource requests and limits as needed

#### Apply the all-in-one variant

```bash
kubectl apply -f rutorrent/namespace.yaml
kubectl apply -f rutorrent/rt-deployment.yaml
```

#### Apply the split-file variant

```bash
kubectl apply -f rutorrent/main/namespace.yaml
kubectl apply -f rutorrent/main/
```

#### Additional manifests

- `rutorrent/rt-restart.yaml` — creates a `ServiceAccount`, `Role`, and `RoleBinding` in `rtorrent-x` that grant pod-delete permissions, useful for automated restart CronJobs

---

## Legacy PV examples

The `PV/` folder contains older standalone PersistentVolume and PersistentVolumeClaim examples from earlier versions of this repo. They use the `nfs-client` storage class (an older NFS provisioner that has been superseded by the NFS CSI driver). The per-service manifest folders like `postgres/` now contain their own, more up-to-date storage definitions.

| File | Description |
|---|---|
| `PV/pv-nfs.yaml` | Generic NFS PV using `nfs-client` |
| `PV/pv-claim.yaml` | Matching PVC (creates a claim named `postgres-pv-claim`) |
| `PV/pv-local.yaml` | Local `hostPath` PV and PVC example |

> [!NOTE]
> These are kept for reference. For new deployments, prefer the NFS CSI-backed PVs in `postgres/pg-pv-nfs.yaml` or create similar manifests with `storageClassName: nfs-csi`.
