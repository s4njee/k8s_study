# Kubernetes Workloads Guide

This guide focuses on the core built-in workload controllers you will use most often in Kubernetes.

The guidance below was checked against the official Kubernetes docs on **March 14, 2026**.

## What this folder is for

The `workloads/` folder contains teaching examples for the main workload patterns:

- `Deployment` for stateless applications
- `StatefulSet` for stable identity and per-replica storage
- `DaemonSet` for one Pod per node
- `Job` for one-time tasks
- `CronJob` for scheduled tasks
- `PodDisruptionBudget` for voluntary disruption protection

These examples are intentionally small and are meant to show controller shape and defaults, not to be production-ready applications.

## Which controller to choose

| Workload | Use it when | Typical examples |
|---|---|---|
| `Deployment` | Pods are interchangeable and rolling updates are normal | web apps, APIs, workers |
| `StatefulSet` | Each replica needs a stable name, identity, or its own volume | databases, queues, clustered systems |
| `DaemonSet` | You need one Pod on each matching node | log agents, node exporters, CNI helpers |
| `Job` | A task should run to completion once | migrations, one-off maintenance, backfills |
| `CronJob` | The same task should run on a schedule | backups, reports, cleanup tasks |

If you are unsure, start with a `Deployment`. Move to `StatefulSet` only when stable identity or per-Pod storage actually matters.

## Workload best practices

Use these defaults unless you have a strong reason not to:

- Pin the image tag or digest on every workload.
- Set CPU and memory requests before you tune limits.
- Add readiness, liveness, and startup probes where the application supports them.
- Disable service-account token mounting for workloads that do not call the Kubernetes API.
- Prefer `Deployment` plus `Service` for stateless HTTP apps.
- Pair `StatefulSet` with a headless `Service`, and use `volumeClaimTemplates` when each replica needs its own PVC.
- Be intentional with `DaemonSet` tolerations so you know whether it should run on control-plane nodes.
- Set `backoffLimit`, cleanup behavior, and idempotent commands for `Job`.
- Set `concurrencyPolicy`, history limits, and an explicit `.spec.timeZone` for `CronJob`.
- Add a `PodDisruptionBudget` for multi-replica applications that must stay available during drains or upgrades.

## Files in `workloads/`

| File | Purpose |
|---|---|
| `workloads/namespace.yaml` | Creates a dedicated `workloads-demo` namespace |
| `workloads/01-deployment.yaml` | Example stateless web app managed by a `Deployment` and exposed by a `Service` |
| `workloads/02-statefulset.yaml` | Example headless `Service` plus `StatefulSet` with per-replica PVCs |
| `workloads/03-daemonset.yaml` | Example node-local agent pattern using a `DaemonSet` |
| `workloads/04-job.yaml` | Example one-time batch task |
| `workloads/05-cronjob.yaml` | Example scheduled batch task with explicit time zone and concurrency policy |
| `workloads/06-pdb.yaml` | Example `PodDisruptionBudget` protecting the web deployment |

## Recommended apply flow

Create the example namespace first:

```bash
kubectl apply -f workloads/namespace.yaml
```

Then apply the specific examples you want to study:

```bash
kubectl apply -f workloads/01-deployment.yaml
kubectl apply -f workloads/02-statefulset.yaml
kubectl apply -f workloads/03-daemonset.yaml
kubectl apply -f workloads/04-job.yaml
kubectl apply -f workloads/05-cronjob.yaml
kubectl apply -f workloads/06-pdb.yaml
```

If you want a preview first:

```bash
kubectl diff -f workloads/01-deployment.yaml
kubectl diff -f workloads/02-statefulset.yaml
```

## Example notes

### 1. Deployment

Manifest: `workloads/01-deployment.yaml`

Use this for stateless apps where any replica can replace any other replica.

What this example demonstrates:

- rolling updates with `maxSurge` and `maxUnavailable`
- a `Service` selecting Pods by label
- readiness and liveness probes
- baseline resource requests and limits

This is the normal default for web applications and APIs.

### 2. StatefulSet

Manifest: `workloads/02-statefulset.yaml`

Use this when each replica needs a stable Pod name, stable network identity, or its own volume.

What this example demonstrates:

- a headless `Service` with `clusterIP: None`
- a `StatefulSet` using `serviceName`
- `volumeClaimTemplates` that create one PVC per replica

The example uses `storageClassName: local-path`, which is a good fit for default K3s labs. Replace it if your cluster uses a different storage class.

### 3. DaemonSet

Manifest: `workloads/03-daemonset.yaml`

Use this for software that should run once on every matching node.

What this example demonstrates:

- one Pod per node
- control-plane tolerations so the Pod can also run on K3s server nodes
- `fieldRef` to expose the node name inside the container

Real-world examples include log shippers, node monitoring agents, and storage or networking helpers.

### 4. Job

Manifest: `workloads/04-job.yaml`

Use this for one-time work that should run until it completes successfully.

What this example demonstrates:

- `restartPolicy: Never`
- `backoffLimit` to avoid infinite retries
- `ttlSecondsAfterFinished` for automatic cleanup after completion

Jobs are a better fit than long-running Deployments for migrations and maintenance tasks.

### 5. CronJob

Manifest: `workloads/05-cronjob.yaml`

Use this when a Job should run on a schedule.

What this example demonstrates:

- an explicit schedule
- `.spec.timeZone` instead of trying to embed `TZ` or `CRON_TZ` in the schedule string
- `concurrencyPolicy: Forbid` so a delayed run does not overlap with the next one
- job history limits to keep the namespace tidy

CronJobs are commonly used for backups, report generation, and cleanup tasks.

### 6. PodDisruptionBudget

Manifest: `workloads/06-pdb.yaml`

Use this for replicated applications that should keep serving during voluntary disruptions such as `kubectl drain`.

What this example demonstrates:

- `minAvailable` protecting a multi-replica `Deployment`
- selector-based matching against the workload Pods

This does not protect against involuntary disruptions such as node crashes. It only influences voluntary evictions that use the Eviction API.

## Rollout commands to know

These are worth memorizing:

```bash
kubectl rollout status deployment/web-demo -n workloads-demo
kubectl rollout history deployment/web-demo -n workloads-demo
kubectl rollout undo deployment/web-demo -n workloads-demo
kubectl get pods -n workloads-demo -w
kubectl describe job one-off-maintenance -n workloads-demo
kubectl get cronjob,job -n workloads-demo
```

If a workload is not behaving, use [TROUBLESHOOTING.md](TROUBLESHOOTING.md) alongside these manifests.

## Upstream references

- [Workloads overview](https://kubernetes.io/docs/concepts/workloads/)
- [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
- [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [Pod disruptions and PodDisruptionBudget](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
