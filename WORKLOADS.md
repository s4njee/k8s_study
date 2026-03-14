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
- `InitContainer` for ordered setup tasks that run before the main containers start
- `HorizontalPodAutoscaler` for automatic scaling based on CPU, memory, or custom metrics

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
- Use init containers for ordered startup dependencies (waiting for a database, seeding config files) rather than embedding retry loops in the main container.
- Add an `HPA` when a Deployment should scale under load. Remove it before manually setting replicas to avoid a fight between you and the autoscaler.

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
| *(inline example)* | `InitContainer` — ordered setup tasks that run before the main containers start |
| *(inline example)* | `HorizontalPodAutoscaler` — automatic scaling based on CPU or memory |

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

### 7. Init containers

Init containers run to completion before any of the main containers start. The Pod stays in `Init:0/N` status until each init container exits successfully. If an init container fails, Kubernetes restarts the Pod according to `restartPolicy`.

Use init containers for:

- waiting for a dependency to be ready (a database, another service) before starting the app
- seeding a shared volume with config files, certificates, or data the main container needs
- running schema migrations before the application pod starts accepting traffic

Init containers share the same Pod network and volumes as the main containers. An `emptyDir` volume is the standard way to pass files between an init container and the main container.

Example shape:

```yaml
spec:
  volumes:
    - name: shared-data
      emptyDir: {}
  initContainers:
    - name: seed-config
      image: busybox:1.36
      command: ['sh', '-c', 'echo "ready" > /shared/status']
      volumeMounts:
        - name: shared-data
          mountPath: /shared
  containers:
    - name: app
      image: my-app:1.0.0
      volumeMounts:
        - name: shared-data
          mountPath: /config
```

Multiple init containers run sequentially in the order they are listed, not in parallel.

### 8. HorizontalPodAutoscaler (HPA)

An `HPA` watches a Deployment (or StatefulSet, or any scale-able workload) and adjusts the replica count automatically based on resource utilization or custom metrics.

Use HPA when:

- traffic to a Deployment varies significantly over time
- you want replicas to scale up under load and back down when idle to reclaim cluster resources

The default HPA algorithm compares current utilization against a target and adjusts replicas up or down to close the gap. The controller checks every 15 seconds by default.

Example that targets 60% CPU utilization with between 2 and 10 replicas:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-demo-hpa
  namespace: workloads-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-demo
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

Requirements and notes:

- The target Deployment must have CPU requests set, or the HPA cannot compute utilization.
- The Metrics Server must be installed in the cluster. K3s ships with it enabled by default.
- Do not manually set `replicas` on a Deployment that has an active HPA — the two will fight. Either remove the `replicas` field from the Deployment spec or delete the HPA before scaling manually.
- For memory-based or custom metric scaling, use `autoscaling/v2` and add additional entries under `.spec.metrics`.

Useful commands:

```bash
kubectl get hpa -n workloads-demo
kubectl describe hpa web-demo-hpa -n workloads-demo
```

The `TARGETS` column in `kubectl get hpa` shows `<current>/<target>`. If it shows `<unknown>/60%`, the Metrics Server is not reachable or the Deployment is missing CPU requests.

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
- [Init containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
- [HorizontalPodAutoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Metrics Server](https://github.com/kubernetes-sigs/metrics-server)
