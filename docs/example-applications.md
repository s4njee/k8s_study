# Example Applications

These are end-user applications that run on top of the cluster. None of them are required for the cluster to function, but they are useful examples once storage, ingress, and supporting services are already working.

Finish [Platform services](platform-services.md) first so the cluster has the foundation these apps expect.

## Table of contents

- [Immich](#immich)
- [Hoarder](#hoarder)
- [ruTorrent](#rutorrent)

## Immich

[Immich](https://immich.app) is a self-hosted photo and video management app. The project provides an official Helm chart for Kubernetes deployments.

If you want to pin a tested chart version, the latest chart release as of March 14, 2026 is `0.10.3`, published on November 14, 2025. The latest Immich app release as of March 14, 2026 is `v2.5.6`, published on February 10, 2026.

> [!IMPORTANT]
> The Helm chart version and the Immich app version are released independently. The chart does not automatically set the app version for you. Specify `image.tag` in your values file to control which version of Immich actually runs.

### 1. Add the Immich Helm repo

```bash
helm repo add immich https://immich-app.github.io/immich-charts
helm repo update
```

### 2. Create the Immich namespace and storage

This repo includes a sample NFS-backed PV and PVC in `immich-charts/rt-pvs.yaml`.

```bash
kubectl create namespace immich
kubectl apply -f immich-charts/rt-pvs.yaml
```

### 3. Create a local values override

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

### 4. Install Immich

```bash
helm upgrade --install immich immich/immich \
  --namespace immich \
  --create-namespace \
  --version 0.10.3 \
  -f immich-charts/values-local.yaml
```

### 5. Verify the install

```bash
helm list -n immich
kubectl get pods -n immich
kubectl get svc -n immich
```

### 6. Optional ingress note for this repo

This repo already contains a sample ingress manifest at `immich-charts/gingress.yaml`, but the chart can also manage ingress directly through `values-local.yaml`. Prefer one approach or the other so you do not create overlapping ingress definitions.

Further reading:

- [Kubernetes install docs](https://docs.immich.app/install/kubernetes)
- [Chart repo](https://github.com/immich-app/immich-charts)
- [Immich releases](https://github.com/immich-app/immich/releases)

## Hoarder

[Hoarder](https://docs.karakeep.app) is now officially renamed to Karakeep. The project moved from `hoarder-app/hoarder` to `karakeep-app/karakeep`, but this repo still uses the `hoarder/` folder name.

Unlike most other apps in this guide, Karakeep uses Kustomize manifests rather than a Helm chart. As of March 14, 2026, the latest release is `v0.31.0` (published February 22, 2026).

### 1. Use the Kubernetes manifests in the repo

The local Kubernetes setup lives in `hoarder/kubernetes/`.

### 2. Create local env and secrets files

Use the sample files:

```bash
cp hoarder/kubernetes/.env.sample hoarder/kubernetes/.env
cp hoarder/kubernetes/.secrets.sample hoarder/kubernetes/.secrets
```

Then edit them for your environment. At minimum, set:

| Variable | Value |
|---|---|
| `NEXTAUTH_URL` | Your actual URL |
| `KARAKEEP_VERSION` | `v0.31.0` |
| `NEXTAUTH_SECRET` | Generate with `openssl rand -base64 36` |
| `MEILI_MASTER_KEY` | Generate with `openssl rand -base64 36` |
| `NEXT_PUBLIC_SECRET` | Generate with `openssl rand -base64 36` |

If you want AI tagging, also set:

```bash
OPENAI_API_KEY=<your_key>
```

### 3. Check storage and service exposure

The manifests in `hoarder/kubernetes/` include PVCs and expose the web service by default. Review these before deploying:

- `hoarder/kubernetes/data-pvc.yaml`
- `hoarder/kubernetes/meilisearch-pvc.yaml`
- `hoarder/kubernetes/web-service.yaml`

If you plan to expose Hoarder through ingress instead of a `LoadBalancer`, change the web service to `ClusterIP` and add your ingress manifest.

### 4. Deploy Hoarder

```bash
cd hoarder/kubernetes
make deploy
```

### 5. Verify the install

```bash
kubectl get all -n hoarder
```

### 6. Important note about the local manifests

> [!WARNING]
> The local `hoarder/kubernetes/` directory is older than the current upstream Karakeep manifests.

Current upstream uses:

| What changed | Old value | New value |
|---|---|---|
| Namespace | `hoarder` | `karakeep` |
| Version variable | `HOARDER_VERSION` | `KARAKEEP_VERSION` |
| Secret management | inline | separate `.secrets` file + `secretGenerator` |
| Web image | — | `ghcr.io/karakeep-app/karakeep` |

For a new install, use the guidance here and the current Karakeep docs as the source of truth even though the local folder is still named `hoarder/`.

Further reading:

- [Kubernetes install docs](https://docs.karakeep.app/installation/kubernetes)
- [Karakeep releases](https://github.com/karakeep-app/karakeep/releases)

## ruTorrent

[ruTorrent](https://github.com/Novik/ruTorrent) is a web-based front end for the rTorrent BitTorrent client. The `rutorrent/` folder contains Kubernetes manifests for running it.

| Variant | Path | Description |
|---|---|---|
| All-in-one | `rutorrent/rt-deployment.yaml` | Deployment, Services, ConfigMap, and NFS-backed PVs/PVCs in a single file |
| Split-file | `rutorrent/main/` | Separate manifests: `rt-main-deploy.yaml`, `rt-service.yaml`, `rt-lb.yaml`, `rt-config.yaml`, `rt-pvs.yaml` |

### Before deploying

Both variations contain values that are specific to the original author's environment. Update them before deploying:

- the container `image` to match your registry
- the NFS `server` and `path` values in the PV definitions
- the `LoadBalancer` IP address, if you use one
- resource requests and limits as needed

### Apply the all-in-one variant

```bash
kubectl apply -f rutorrent/namespace.yaml
kubectl apply -f rutorrent/rt-deployment.yaml
```

### Apply the split-file variant

```bash
kubectl apply -f rutorrent/main/namespace.yaml
kubectl apply -f rutorrent/main/
```

### Additional manifests

- `rutorrent/rt-restart.yaml` creates a `ServiceAccount`, `Role`, and `RoleBinding` in `rtorrent-x` that grant pod-delete permissions, useful for automated restart CronJobs.
