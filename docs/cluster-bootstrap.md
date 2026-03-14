# Cluster Bootstrap

This guide covers the base cluster setup: K3s, local kubeconfig access, Headlamp, the NFS CSI driver, and Traefik ingress.

If you are new to Kubernetes, read [../CONCEPTS.md](../CONCEPTS.md) first so the objects in this guide are easier to reason about.

## Table of contents

- [Install k3s](#install-k3s)
- [Install Headlamp](#install-headlamp)
- [Install the NFS CSI driver](#install-the-nfs-csi-driver)
- [Set up Traefik ingress](#set-up-traefik-ingress)

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

On the server node, K3s writes a kubeconfig file to `/etc/rancher/k3s/k3s.yaml`. A kubeconfig is a credentials file that tells `kubectl` how to connect to your cluster.

Copy that file to your local machine and save it as `~/.kube/config` so you can manage the cluster from your own computer. The file is usually only readable by `root`, so use one of these approaches to copy it.

If your SSH user has passwordless `sudo`, stream the file over SSH:

```bash
mkdir -p ~/.kube
ssh <user>@<server_ip_or_dns> "sudo cat /etc/rancher/k3s/k3s.yaml" > ~/.kube/config
chmod 600 ~/.kube/config
```

If you prefer `scp`, first copy it to your user's home directory on the server with `sudo`, then download it:

```bash
ssh <user>@<server_ip_or_dns> "sudo cp /etc/rancher/k3s/k3s.yaml /home/<user>/k3s.yaml && sudo chown <user>:<user> /home/<user>/k3s.yaml"
mkdir -p ~/.kube
scp <user>@<server_ip_or_dns>:/home/<user>/k3s.yaml ~/.kube/config
chmod 600 ~/.kube/config
```

> [!IMPORTANT]
> After copying it, update the `server:` value inside `~/.kube/config` if it still points to `https://127.0.0.1:6443`. Replace it with the same DNS name or IP address you used for `--tls-san`.

## Install Headlamp

Headlamp is a modern web-based UI for Kubernetes. It lets you browse your cluster's resources, view logs, and manage workloads through a graphical interface instead of the command line. It is a better default choice than the older Kubernetes Dashboard for most homelab use.

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

You can access it locally with port-forwarding, which creates a temporary tunnel from your computer to a service inside the cluster:

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

## Install the NFS CSI driver

If you have an NFS file server on your network, you can use it to store data for your Kubernetes apps. The NFS CSI driver is a plugin that teaches Kubernetes how to mount NFS shares as persistent volumes.

Add the Helm repo and install the driver in `kube-system`:

```bash
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm repo update
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs --namespace kube-system
```

If you want to pin a tested chart version, the current upstream release as of March 14, 2026 is `4.13.1`:

```bash
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
  --namespace kube-system \
  --version 4.13.1
```

> [!NOTE]
> Make sure each Kubernetes node has the NFS client packages installed so it can mount NFS shares.

After the driver is installed, you can create a PV and PVC that reference it. In your manifest files, set:

```yaml
storageClassName: nfs-csi
csi:
  driver: nfs.csi.k8s.io
```

An example static NFS CSI-backed Postgres volume is in:

- `postgres/pg-pv-nfs.yaml`
- `postgres/pg-pv-claim.yaml`
- `postgres/pg-deployment.yaml`

## Set up Traefik ingress

Traefik is a reverse proxy that acts as the front door to your cluster. When someone visits `https://myapp.example.com`, Traefik receives the request and routes it to the correct service inside Kubernetes based on Ingress resources.

K3s includes Traefik by default unless you installed K3s with `--disable=traefik`. On K3s, Traefik is typically exposed on ports `80` and `443` by the built-in ServiceLB.

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

Let's Encrypt needs to reach your cluster on ports `80` and `443` to verify that you own the domain name.

- If this is a public setup, point your router or firewall at the K3s node that is serving Traefik.
- If this is a LAN-only homelab, make sure the hostname resolves to a reachable node IP.
- If multiple nodes exist, remember that K3s ServiceLB may advertise more than one node IP unless you restrict the Traefik service to specific nodes.

### 4. Optional: customize the packaged Traefik install

> [!CAUTION]
> Do not edit `/var/lib/rancher/k3s/server/manifests/traefik.yaml` directly. K3s will replace it.

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

## Next step

Once the base cluster is healthy, continue with [Platform services](platform-services.md).
