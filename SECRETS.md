# Kubernetes Secrets Guide

This repo already uses Secrets in a few places, but the examples were scattered. This guide collects the main patterns in one place and documents the sample manifests in `secrets/`.

The guidance below was checked against the official Kubernetes and K3s docs on **March 14, 2026**.

## What this folder is for

The `secrets/` folder contains **teaching examples only**:

- all secret values are dummy placeholders
- some workloads use placeholder images such as `registry.example.com/demo-app:1.0.0`
- the TLS example uses placeholder PEM blocks that must be replaced before real use

Do **not** commit real passwords, tokens, private keys, or registry credentials to git.

## Best practices

Use these defaults unless you have a specific reason not to:

- Treat Kubernetes Secrets as sensitive API objects, not as encrypted vault entries. `data` is base64-encoded, not safely hidden.
- Enable encryption at rest for Secrets in the cluster datastore. For K3s, that means starting servers with `--secrets-encryption` and managing rotation intentionally.
- Keep secrets out of git. For real environments, prefer generated Secrets, `.env` or file-based inputs excluded from git, or tooling such as SOPS, Sealed Secrets, or External Secrets.
- Use dedicated namespaces and tight RBAC. In practice, someone who can create Pods or Deployments in a namespace can usually expose Secrets from that namespace.
- Mount only the specific secret keys a workload needs. Avoid giving every container broad access to every secret in the Pod.
- Prefer secret volumes when the application can read files. Use environment variables only when the application requires them, because env vars are easier to leak through logs, crash dumps, or debug output.
- Disable automatic service-account token mounting when a workload does not need to call the Kubernetes API by setting `automountServiceAccountToken: false`.
- Avoid long-lived `kubernetes.io/service-account-token` Secrets unless you truly have no better option. Short-lived projected tokens or `kubectl create token` are safer defaults.
- Rotate credentials and certificates on purpose. If a secret rarely changes, consider `immutable: true` and replace the whole object during rotation instead of editing in place.
- Protect backups and snapshots the same way you protect the live cluster. Encrypting the API datastore does not help if your backups are left exposed.

## Files in `secrets/`

| File | Purpose |
|---|---|
| `secrets/namespace.yaml` | Creates a dedicated `secrets-demo` namespace for the examples |
| `secrets/01-env-secret.yaml` | Stores app credentials in an `Opaque` Secret and injects selected keys as environment variables |
| `secrets/02-volume-secret.yaml` | Mounts selected secret keys as read-only files inside a Pod |
| `secrets/03-image-pull-secret.yaml` | Shows a `kubernetes.io/dockerconfigjson` Secret attached to a `ServiceAccount` for private image pulls |
| `secrets/04-tls-secret.yaml` | Shows a `kubernetes.io/tls` Secret used by an `Ingress` for HTTPS termination |

## Recommended apply flow

Create the example namespace first:

```bash
kubectl apply -f secrets/namespace.yaml
```

Then apply whichever example you want to study:

```bash
kubectl apply -f secrets/01-env-secret.yaml
kubectl apply -f secrets/02-volume-secret.yaml
kubectl apply -f secrets/03-image-pull-secret.yaml
kubectl apply -f secrets/04-tls-secret.yaml
```

If you want a safer preview first:

```bash
kubectl apply --dry-run=server -f secrets/namespace.yaml
kubectl diff -f secrets/01-env-secret.yaml
```

## Example notes

### 1. Secret as environment variables

Manifest: `secrets/01-env-secret.yaml`

Use this pattern when the application only knows how to read credentials from environment variables.

What this example demonstrates:

- `stringData` keeps the source manifest readable while letting Kubernetes write the encoded `data` field server-side
- `secretKeyRef` exposes only the needed keys instead of importing the whole Secret
- `automountServiceAccountToken: false` avoids giving the Pod an API token it does not need

Tradeoffs:

- environment variables are convenient, but they are also easy to leak through logs, debug endpoints, support bundles, or process inspection
- changing the Secret usually means restarting the workload so the process picks up the new values

### 2. Secret as mounted files

Manifest: `secrets/02-volume-secret.yaml`

Use this when the application can read credentials, client certificates, or keys from files.

What this example demonstrates:

- `items` mounts only the keys the container actually needs
- `defaultMode: 0400` narrows file permissions
- the secret volume is mounted read-only at `/etc/app-secrets`

This is often the better default for certificates, CA bundles, SSH material, or applications that can reload files without relying on environment variables.

### 3. Secret for private image pulls

Manifest: `secrets/03-image-pull-secret.yaml`

Use this when a namespace needs to pull images from a private registry.

What this example demonstrates:

- the Secret type is `kubernetes.io/dockerconfigjson`
- the registry credentials are attached to a `ServiceAccount`
- Pods can reuse that `ServiceAccount` instead of repeating `imagePullSecrets` on every workload

For day-to-day use, `kubectl create secret docker-registry ...` is usually easier than hand-authoring the JSON, but the manifest is useful to understand the object shape.

### 4. TLS Secret for ingress

Manifest: `secrets/04-tls-secret.yaml`

Use this when an ingress controller needs a certificate and private key for HTTPS termination.

What this example demonstrates:

- the Secret type is `kubernetes.io/tls`
- the `Ingress` references the Secret by name under `spec.tls`
- the example includes a simple Deployment and Service so the routing path is visible in one file

For real clusters, prefer letting `cert-manager` create and rotate TLS Secrets instead of checking certificates into git.

## What to replace before real use

Before applying these examples outside a lab, replace:

- all `replace-me` placeholder values
- all placeholder images under `registry.example.com/...`
- `demo.example.com` in the ingress example
- the fake certificate and private key in `secrets/04-tls-secret.yaml`

## Upstream references

- [Kubernetes Secret concepts](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Good practices for Kubernetes Secrets](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)
- [Kubernetes Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
- [Kubernetes RBAC good practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
- [K3s secrets encryption](https://docs.k3s.io/security/secrets-encryption)
