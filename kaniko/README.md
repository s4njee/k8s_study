# Kaniko

This folder contains a current Kubernetes example for Kaniko.

## 2026 note

The original `GoogleContainerTools/kaniko` repository was archived in June 2025.
For 2026 installs, use the maintained fork from `osscontainertools/kaniko`.

As of March 14, 2026, the latest release is `v1.27.0`:

- Release: <https://github.com/osscontainertools/kaniko/releases/tag/v1.27.0>
- Image: `ghcr.io/osscontainertools/kaniko:v1.27.0`

## 1. Create a registry auth secret

Kaniko expects a Docker config at `/kaniko/.docker/config.json`.
Create a standard Kubernetes registry secret in the namespace where you will run the build:

```bash
kubectl apply -f kaniko/namespace.yaml
kubectl create secret docker-registry regcred \
  --namespace kaniko \
  --docker-server=registry.example.com \
  --docker-username=<registry_user> \
  --docker-password=<registry_password>
```

## 2. Update the build manifest

Edit `kaniko.yaml` and change:

- `--context` to your build context
- `--dockerfile` if your Dockerfile is not at the repo root
- `--destination` to your target image
- `--cache-repo` to a writable cache repository in the same registry
- `secretName` if your registry secret uses a different name

The included example uses a public Git context:

```text
--context=git://github.com/your-org/your-repo.git#refs/heads/main
```

If you want to build from a mounted workspace instead, switch to:

```text
--context=dir:///workspace
```

and mount a PVC or other volume at `/workspace`.

## 3. Run the build

```bash
kubectl apply -f kaniko/namespace.yaml
kubectl apply -f kaniko/kaniko.yaml
kubectl logs -n kaniko -f job/kaniko-build
```

## 4. Clean up the finished job

```bash
kubectl delete job -n kaniko kaniko-build
```

## Notes

- This example uses `--cache=true` and `--cache-repo=...` because remote layer caching is one of the easiest speed wins.
- `--snapshot-mode=redo` and `--use-new-run` are commonly used modern defaults for better performance.
- Avoid `:latest` for the executor image. Pin the Kaniko image version explicitly.

Upstream docs:

- <https://github.com/osscontainertools/kaniko>
