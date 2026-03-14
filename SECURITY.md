# Kubernetes Security Guide

This guide covers practical security defaults for a small Kubernetes or K3s cluster.

The guidance below was checked against the official Kubernetes and K3s docs on **March 14, 2026**.

## What this folder is for

The `security/` folder contains small examples for common security building blocks:

- namespace-level Pod Security Admission labels
- least-privilege `ServiceAccount` and RBAC
- a restricted-style Deployment
- basic `NetworkPolicy` patterns

For secret-specific guidance, see [SECRETS.md](SECRETS.md).

## Security layers that matter most

Think in layers instead of one silver bullet:

- namespace isolation and policy
- service accounts and RBAC
- pod and container hardening
- network policies
- secrets handling and encryption at rest
- node and control-plane hardening
- backup and audit-log protection

## Security best practices

Use these defaults unless a workload explicitly needs something broader:

- Put applications in dedicated namespaces.
- Enforce Pod Security Standards with namespace labels, starting from `restricted` where possible.
- Do not hand out `cluster-admin` to normal workloads or day-to-day users.
- Create dedicated service accounts only when a workload needs Kubernetes API access.
- Set `automountServiceAccountToken: false` unless the Pod actually needs that token.
- Run containers as non-root where the image supports it.
- Set `seccompProfile.type: RuntimeDefault`.
- Drop Linux capabilities and keep `allowPrivilegeEscalation: false`.
- Use a read-only root filesystem when the image supports it, and add writable volumes only where needed.
- Avoid `privileged`, `hostNetwork`, `hostPID`, `hostIPC`, and broad `hostPath` mounts unless the workload truly requires them.
- Use `NetworkPolicy` to make namespace traffic rules explicit instead of relying on default openness.
- Protect the cluster datastore and backups. For K3s, enable secrets encryption intentionally and treat snapshots as sensitive data.
- Enable audit logging if you need traceability. K3s does not enable audit logging by default.

## K3s-specific notes

These are especially relevant for this repo:

- K3s ships with a network-policy controller that can enforce policies if you create them.
- K3s does not create Pod Security or NetworkPolicy objects for you by default, so you still need to apply them.
- K3s runs the `PodSecurity` admission controller by default, but cluster-wide defaults and exemptions require explicit API server configuration if you want more than namespace labels.
- The K3s hardening guide is worth treating as a checklist for anything beyond a throwaway lab.

## Files in `security/`

| File | Purpose |
|---|---|
| `security/namespace.yaml` | Creates a namespace with `restricted` Pod Security Admission labels |
| `security/01-serviceaccount-rbac.yaml` | Demonstrates a least-privilege `ServiceAccount`, `Role`, and `RoleBinding` |
| `security/02-secure-deployment.yaml` | Demonstrates a hardened Deployment and `Service` with non-root execution and restricted container settings |
| `security/03-networkpolicy.yaml` | Demonstrates default-deny plus explicit DNS egress and same-namespace ingress rules |

## Recommended apply flow

Create the namespace first:

```bash
kubectl apply -f security/namespace.yaml
```

Then apply the supporting examples:

```bash
kubectl apply -f security/01-serviceaccount-rbac.yaml
kubectl apply -f security/02-secure-deployment.yaml
kubectl apply -f security/03-networkpolicy.yaml
```

> [!IMPORTANT]
> The default-deny policy in `security/03-networkpolicy.yaml` is intentionally restrictive. Adapt the allowed ingress and egress rules before using that pattern with real workloads.

## Example notes

### 1. Restricted namespace

Manifest: `security/namespace.yaml`

Use namespace labels to make Pod Security Admission expectations explicit.

What this example demonstrates:

- `enforce`, `audit`, and `warn` labels
- the `restricted` profile as a strong default for application namespaces
- `latest` policy tracking instead of pinning a namespace to an old Kubernetes minor version

System namespaces and some infrastructure components may need broader rules. Application namespaces usually should not.

### 2. ServiceAccount and RBAC

Manifest: `security/01-serviceaccount-rbac.yaml`

Use a dedicated service account only when a workload truly needs Kubernetes API access.

What this example demonstrates:

- a dedicated `ServiceAccount`
- a namespace-scoped `Role`
- a `RoleBinding` instead of a cluster-wide permission grant

Keep these roles as narrow as possible. If the workload only needs `get` on one resource type, do not also grant `list`, `watch`, or cross-namespace access.

### 3. Secure deployment

Manifest: `security/02-secure-deployment.yaml`

Use this as a baseline example for a restricted application Pod.

What this example demonstrates:

- `automountServiceAccountToken: false`
- pod-level `seccompProfile: RuntimeDefault`
- `runAsNonRoot`, explicit UID and GID, and dropped capabilities
- `allowPrivilegeEscalation: false`
- `readOnlyRootFilesystem: true`
- writable `emptyDir` mounts only where the process actually needs them

This is not universal. Some vendor images still assume root or a writable root filesystem. The point is to force those exceptions to be deliberate.

### 4. Network policy

Manifest: `security/03-networkpolicy.yaml`

Use network policies to make east-west traffic rules intentional.

What this example demonstrates:

- a namespace-wide default deny
- explicit DNS egress on TCP and UDP port `53`
- explicit same-namespace ingress to the `secure-app` Pods on port `8080`

Without an allow rule, default-deny policies will block more than people expect. DNS is the classic first thing to break.

## Practical checklist

If you are tightening a real namespace, this is a good first pass:

- patch the default service account to disable automount if nothing in the namespace needs it
- label the namespace for Pod Security Admission
- review every workload for root, privilege escalation, and writable filesystem assumptions
- add resource requests and limits
- add default-deny `NetworkPolicy` plus explicit allow rules
- review RBAC bindings for wildcard verbs, wildcard resources, and cluster-wide grants
- move secret values out of git and enable encryption at rest
- decide whether audit logging should be enabled on the cluster

## Upstream references

- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [RBAC good practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
- [Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
- [Linux kernel security constraints and features](https://kubernetes.io/docs/concepts/security/linux-kernel-security-constraints/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [K3s hardening guide](https://docs.k3s.io/security/hardening-guide)
