# Kubernetes Study Guide

A hands-on K3s and Kubernetes lab repo focused on repeatable installs, teachable manifests, and practical homelab operations.

## Project goals

This repo is meant to be:

- a guided learning path for core Kubernetes concepts and day-2 operations
- a repeatable lab reference with pinned versions and dated verification notes
- a collection of small, readable examples for workloads, security, secrets, storage, ingress, monitoring, and backups

It isn't trying to be a polished product distro for every app in here. Some folders are first-party study material. Others are checked-in upstream references that support the guide.

## Quick start

If you're new here, don't try to absorb this whole repo at once.

Start with the path below and keep day 1 small:

1. Read [CONCEPTS.md](CONCEPTS.md) to build the Kubernetes mental model first.
2. Follow [docs/cluster-bootstrap.md](docs/cluster-bootstrap.md) to build the base cluster.
3. Add shared services from [docs/platform-services.md](docs/platform-services.md).
4. Deploy optional apps from [docs/example-applications.md](docs/example-applications.md).
5. Use the companion guides below as references while you work.

That's the intended learning path. Get the basics working first. Then layer in the rest.

## Documentation map

Step-by-step guides:

- [docs/README.md](docs/README.md)
- [docs/cluster-bootstrap.md](docs/cluster-bootstrap.md)
- [docs/platform-services.md](docs/platform-services.md)
- [docs/example-applications.md](docs/example-applications.md)

Companion guides:

- [CONCEPTS.md](CONCEPTS.md)
- [SECRETS.md](SECRETS.md)
- [WORKLOADS.md](WORKLOADS.md)
- [SECURITY.md](SECURITY.md)
- [OPERATIONS.md](OPERATIONS.md)
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md)

## Repo layout

The top level mixes repo-owned study material with embedded upstream references. Treat them a little differently:

| Area | Status | Purpose |
|---|---|---|
| `README.md`, `docs/`, `CONCEPTS.md`, `SECRETS.md`, `WORKLOADS.md`, `SECURITY.md`, `OPERATIONS.md`, `TROUBLESHOOTING.md` | first-party | Main learning path and repo-owned guidance |
| `secrets/`, `security/`, `workloads/`, `postgres/`, `kaniko/`, `opensearch/`, `kube-prometheus-stack/`, `certbox/`, `PV/`, `rutorrent/` | first-party examples | Local manifests, values files, and app notes |
| `actions-runner-controller/`, `grafana-dashboards-kubernetes/`, `hoarder/`, `immich-charts/`, `docker-registry.helm/` | upstream or mirrored content | Checked-in upstream references or charts; usually better to change the repo-owned docs around them unless you intentionally want to update the mirror |

## Best practices

A few good defaults go a long way.

### Day 1 defaults

- pin versions for anything you want to be repeatable: K3s, Helm charts, container images, and manifest URLs
- use override files instead of editing base manifests whenever Helm or Kustomize already gives you an extension point
- give each application its own namespace, and reserve `kube-system` for cluster-level services
- keep secrets out of git by using generated Kubernetes Secrets, file-based inputs excluded from version control, or dedicated secret tooling
- preview changes with `kubectl diff`, `kubectl apply --dry-run=server`, and `helm upgrade --install --wait --atomic`

### Once things feel comfortable

- expose HTTP apps through ingress instead of `LoadBalancer` unless you need direct non-HTTP network exposure
- set image pins, resource requests, health probes, and baseline security context before treating a workload as reusable
- write down backup and restore paths for stateful services instead of leaving recovery implicit

## Project standards

These are the standards this repo is aiming for:

- pinned versions are intentional and updated with exact verification dates
- docs explain what is teaching material, what is optional, and what is not fully maintained here
- examples use safe defaults and avoid live secrets
- repo hygiene files and CI make it easier to collaborate confidently

See [CONTRIBUTING.md](CONTRIBUTING.md) for contribution expectations and [LICENSE](LICENSE) for the repo license.
