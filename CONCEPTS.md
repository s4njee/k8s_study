# Kubernetes Core Concepts

This guide explains the foundational Kubernetes ideas that come up again and again across this repo. If you are new to Kubernetes, read this before diving into the installation steps — understanding these concepts will make everything else click faster.

The guidance below was checked against the official Kubernetes docs on **March 14, 2026**.

---

## Table of contents

- [What is a Pod?](#what-is-a-pod)
- [The declarative model — how Kubernetes thinks](#the-declarative-model--how-kubernetes-thinks)
- [The control plane — what's running behind the scenes](#the-control-plane--whats-running-behind-the-scenes)
- [Namespaces — organizing your cluster](#namespaces--organizing-your-cluster)
- [Labels and selectors — finding things](#labels-and-selectors--finding-things)
- [Workload controllers — managing Pods at scale](#workload-controllers--managing-pods-at-scale)
- [Services — how apps talk to each other](#services--how-apps-talk-to-each-other)
- [Ingress — routing external traffic](#ingress--routing-external-traffic)
- [ConfigMaps — app settings without rebuilding](#configmaps--app-settings-without-rebuilding)
- [Secrets — sensitive configuration](#secrets--sensitive-configuration)
- [Persistent Volumes — giving Pods lasting storage](#persistent-volumes--giving-pods-lasting-storage)
- [Health probes — knowing when a Pod is ready](#health-probes--knowing-when-a-pod-is-ready)
- [Resource requests and limits](#resource-requests-and-limits)
- [Node scheduling — controlling where Pods run](#node-scheduling--controlling-where-pods-run)
- [CRDs and the Operator pattern](#crds-and-the-operator-pattern)
- [RBAC — controlling who can do what](#rbac--controlling-who-can-do-what)

---

## What is a Pod?

Before understanding the concepts below, it helps to know what a **Pod** is, because everything in Kubernetes revolves around them.

A Pod is the smallest unit Kubernetes manages. Think of it as a wrapper around one or more containers that should always run together on the same machine. In practice, most Pods contain a single container (your app), though some add a small helper container alongside it.

The key thing to know about Pods: **they are temporary**. If a Pod crashes, Kubernetes replaces it with a new one — but the new Pod gets a different IP address and a different name. This is intentional. Kubernetes is designed around the idea that individual containers come and go, and the system keeps things running regardless.

This "temporary" nature is what motivates everything else on this page.

---

## The declarative model — how Kubernetes thinks

The single biggest mental shift when learning Kubernetes is understanding that you never tell it *how* to do something — you tell it *what you want*, and it figures out the steps.

This is called the **declarative model**:

1. You write a YAML file describing the desired state ("I want 3 replicas of this web app running")
2. You apply it with `kubectl apply`
3. Kubernetes reads it, compares it against what is currently running, and takes action to close the gap

This happens continuously. A loop called the **reconciliation loop** runs constantly inside Kubernetes: observe current state → compare to desired state → take action if they differ.

**This is why things happen automatically.** It is not magic — it is just a loop running in the background.

- A crashed Pod gets replaced — Kubernetes noticed current state (2 replicas) did not match desired state (3 replicas) and created a new one
- You can re-apply the same manifest safely — if nothing changed, nothing happens
- GitOps works — you push desired state to Git, and a controller applies it to the cluster

### Imperative vs declarative

| Approach | How you think | Example |
|---|---|---|
| Imperative | "Start this container on node 3" | `docker run ...` |
| Declarative | "I want 3 copies of this app running" | `kubectl apply -f deployment.yaml` |

Kubernetes supports both styles, but the declarative approach — YAML files committed to Git — is how real clusters are managed. It is self-documenting, auditable, and repeatable.

### Upstream references

- [Declarative management](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/)
- [Controllers](https://kubernetes.io/docs/concepts/architecture/controller/)

---

## The control plane — what's running behind the scenes

When you run `kubectl apply`, something has to receive that request, decide what to do, and make it happen. That "something" is the **control plane** — the set of components that manage the cluster.

In a K3s setup, all control plane components run on the **server node**. Worker nodes (agents) just run your workload Pods.

### The four key components

**API server** (`kube-apiserver`)

Every interaction with Kubernetes goes through the API server — your `kubectl` commands, Headlamp, Helm, and controllers all talk to it. Think of it as the front desk: nothing happens without checking in here first.

**etcd**

The database where Kubernetes stores all cluster state: what Pods exist, what their desired state is, what Secrets contain, what nodes are registered. This is what you are backing up when you back up the cluster. If etcd is lost, the cluster's memory is gone.

**Scheduler** (`kube-scheduler`)

When a new Pod needs to run, the scheduler decides which node it should land on — based on available resources, taints, affinity rules, and other constraints. It picks the destination but does not start the Pod.

**Controller manager** (`kube-controller-manager`)

Runs many control loops in a single process. The Deployment controller makes sure the right number of Pods are running. The Node controller watches for nodes going offline. Each controller handles one concern and follows the same "observe → compare → act" pattern.

### What happens when you run `kubectl apply`

```
kubectl apply → API server → etcd (stores desired state)
                           ↓
               Controller manager (notices change, decides to create Pods)
                           ↓
               Scheduler (picks a node for each Pod)
                           ↓
               kubelet on node (pulls the image, starts the container)
```

> [!NOTE]
> K3s combines all of these into a single binary on the server node. This is what makes it lightweight compared to full Kubernetes distributions where each component runs separately.

### Upstream references

- [Kubernetes components](https://kubernetes.io/docs/concepts/overview/components/)
- [K3s architecture](https://docs.k3s.io/architecture)

---

## Namespaces — organizing your cluster

### The problem

As you add more applications to a cluster, everything starts to pile up in one place. How do you keep a team's test workloads from accidentally affecting production? How do you make sure one app cannot consume all of the cluster's memory and starve out everything else?

### The solution: Namespaces

A **Namespace** is a virtual partition inside a Kubernetes cluster. Think of it like a folder on your computer — it groups related objects together and keeps them separate from objects in other folders.

By default, everything lands in the `default` namespace. In practice, it is much cleaner to give each application or team its own namespace:

```
cluster
├── namespace: production
│   ├── Deployment: web-app
│   └── Service: web-app
├── namespace: staging
│   ├── Deployment: web-app   ← same names, totally separate
│   └── Service: web-app
└── namespace: monitoring
    └── Deployment: prometheus
```

Most Kubernetes objects are namespace-scoped: Pods, Deployments, Services, ConfigMaps, Secrets, and so on. A few are cluster-scoped and exist outside any namespace — Nodes and PersistentVolumes are examples.

### What namespaces give you

- **Isolation:** Objects in different namespaces do not interfere with each other by default, even if they share the same name.
- **Access control:** You can grant a team permission to manage their own namespace without giving them cluster-wide access.
- **Resource limits:** A `ResourceQuota` applied to a namespace caps how much CPU, memory, and how many objects that namespace can use.
- **Network policies:** You can write rules that allow or deny traffic between namespaces.

### Useful commands

```bash
# List all namespaces
kubectl get namespaces

# Work in a specific namespace
kubectl get pods -n my-namespace

# Set a default namespace for your session so you don't need -n every time
kubectl config set-context --current --namespace=my-namespace
```

> [!TIP]
> Any time you create a new application or set up a new team's environment, create a dedicated namespace for it first. It is the single easiest thing you can do to keep a cluster organized.

### Upstream references

- [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)

---

## Labels and selectors — finding things

### The problem

Kubernetes manages many objects at once. How does a Service know which Pods it should route traffic to? How does a Deployment know which Pods it owns? How do you quickly find all objects related to a specific application?

### The solution: Labels and selectors

A **label** is a simple key-value pair you attach to any Kubernetes object. Labels are just metadata — Kubernetes does not interpret them. You decide what they mean.

```yaml
metadata:
  labels:
    app: web-frontend
    environment: production
    version: "2.1.0"
```

A **selector** is how one object says "find me everything with these labels." Services use selectors to decide which Pods to route traffic to:

```yaml
# In a Service
selector:
  app: web-frontend     # ← route traffic to Pods with this label
```

```yaml
# In a Deployment
selector:
  matchLabels:
    app: web-frontend   # ← this Deployment manages Pods with this label
```

This is the glue that holds Kubernetes together. When you deploy a new version and old Pods are being replaced, the Service keeps routing to whichever Pods are currently running and healthy — because it selects by label, not by Pod name.

### A common labeling convention

There are no required labels, but this pattern is widely used:

| Label | Example value | What it represents |
|---|---|---|
| `app` | `web-frontend` | The application name |
| `component` | `database` | The role within the app |
| `environment` | `production` | Which environment this runs in |
| `version` | `2.1.0` | The version of the app |

You do not need all of these. Even just `app` is enough to make selectors work.

### Annotations — labels for humans and tools

**Annotations** are similar to labels but are for information that is not used for selecting objects. Tools (like cert-manager or monitoring agents) use annotations to attach metadata they need:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
```

The distinction: labels are for selectors (used by Kubernetes to match objects), annotations are for attaching arbitrary information (used by tools and humans).

### Useful commands

```bash
# Add a label to a running Pod
kubectl label pod my-pod environment=production

# Filter output by label
kubectl get pods -l app=web-frontend
kubectl get pods -l environment=production,app=web-frontend

# See labels on all objects in a namespace
kubectl get pods --show-labels
```

> [!NOTE]
> If a Service is not routing traffic to your Pods, check that the labels on your Pods exactly match the selector in your Service. Even a small typo — or a missing label — will result in an empty Endpoints list and no traffic.

### Upstream references

- [Labels and selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
- [Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)

---

## Workload controllers — managing Pods at scale

In Kubernetes you rarely create Pods directly. Instead, you create a **workload controller** — an object that creates and manages Pods for you, replacing them if they crash and maintaining the desired count.

### The main controllers

| Controller | Use it when | Key characteristic |
|---|---|---|
| `Deployment` | Your app is stateless and any replica can replace any other | Rolling updates, easy scaling |
| `StatefulSet` | Each replica needs a stable name, ordered startup, or its own volume | Pods named `db-0`, `db-1`, `db-2` |
| `DaemonSet` | You need exactly one Pod running on every matching node | Log agents, monitoring exporters |
| `Job` | A task should run once to completion | Database migrations, backfill scripts |
| `CronJob` | The same task should run on a schedule | Backups, cleanup tasks |

### Deployment — the right default for most apps

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3            # Keep 3 Pods running at all times
  selector:
    matchLabels:
      app: web-app
  template:              # Pod template — what each Pod looks like
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: web
          image: my-app:1.0.0
```

When you apply this, the Deployment controller creates the 3 Pods. If one crashes, the controller notices and creates a replacement. When you update the image tag, it performs a rolling update — replacing Pods one at a time so the app stays available.

### StatefulSet — when identity matters

```
web-app-7d4f9b-xkp2m  ← Deployment Pod name (random, changes on restart)
postgres-0             ← StatefulSet Pod name (stable, always postgres-0)
```

Because StatefulSet Pod names are stable, you can address `postgres-0.postgres-svc` directly from other Pods. This is how database replicas know which one is the primary, and why `StatefulSet` is always paired with a headless `Service`.

> [!TIP]
> If you are unsure which controller to use, start with a `Deployment`. Only switch to `StatefulSet` if you actually need stable identities or per-Pod volumes.

See [`WORKLOADS.md`](WORKLOADS.md) for detailed examples of all five controller types, including PodDisruptionBudget, init containers, and HorizontalPodAutoscaler.

### Upstream references

- [Workloads](https://kubernetes.io/docs/concepts/workloads/)
- [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

---

## Services — how apps talk to each other

### The problem

Imagine your web app needs to talk to a database. Normally you would give it the database's IP address. But in Kubernetes, every time the database Pod restarts, it gets a **new IP address**. Your web app would immediately lose the connection.

This would be a nightmare to manage manually.

### The solution: Services

A **Service** is a stable address that sits in front of a group of Pods. It never changes, even when the Pods behind it are replaced. Kubernetes keeps the Service address constant and handles routing traffic to whichever Pods are currently running.

Think of it like a restaurant phone number. You always call the same number, and whoever picks up the phone handles your order. You do not need to know which specific employee answers — you just need the number to stay the same.

```
Your app → Service (stable address) → Pod A
                                     → Pod B
                                     → Pod C
```

If Pod B crashes and is replaced by Pod D, the Service automatically starts routing to Pod D instead. Your app never needs to know.

### Service types

There are four types of Services, and choosing the right one is important:

| Type | Who can reach it | Analogy | Use it when |
|---|---|---|---|
| `ClusterIP` | Only apps inside the cluster | An internal extension number — only employees can dial it | Default. Most services should use this. |
| `NodePort` | Anyone who can reach a cluster node, on a specific port | A side door to the building | Quick testing in a lab. Avoid in production. |
| `LoadBalancer` | The public internet, via an external IP | The main entrance with a doorman who checks IDs | Exposing non-HTTP services directly (game servers, databases, etc.) |
| `ExternalName` | Apps inside the cluster | A phone book entry that just redirects you somewhere else | Aliasing an outside service as if it lived inside the cluster |

> [!TIP]
> If you are exposing a web app, use `ClusterIP` + an **Ingress** rule (see the Traefik section in the main README) instead of `LoadBalancer`. Ingress handles HTTPS, routing, and certificates in one place.

### Headless Services

A special variant worth knowing: if you set `clusterIP: None`, you get a **headless Service**. Instead of a single stable address, Kubernetes returns the individual IP addresses of each Pod when you look up the Service name in DNS.

This is how `StatefulSet` databases give each replica its own addressable name, like `postgres-0.postgres-svc.namespace.svc.cluster.local`. You would not use this for a normal web app, but you will see it in database and queue configurations.

### Useful commands

```bash
# List all Services in a namespace
kubectl get svc -n <namespace>

# See which Pods a Service is routing to
kubectl describe svc <service-name> -n <namespace>

# Check the Endpoints (actual Pod IPs the Service is using)
kubectl get endpoints <service-name> -n <namespace>
```

If a Service exists but traffic is not reaching your app, check the `Endpoints` — if that list is empty, the Service's label selector does not match any running Pods.

### Upstream references

- [Services overview](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Connecting applications with Services](https://kubernetes.io/docs/tutorials/services/connect-applications-service/)

---

## Ingress — routing external traffic

### The problem

Services give Pods stable internal addresses, but by default a `ClusterIP` Service is only reachable inside the cluster. How does traffic from a browser visiting `https://myapp.example.com` get in?

You could use `LoadBalancer` Services, but each one gets its own external IP. Ten apps would mean ten external IPs and ten separate HTTPS certificates — expensive and complicated.

### The solution: Ingress + Ingress Controller

An **Ingress** is a rule that says: "HTTP traffic arriving for `myapp.example.com` at path `/` should go to Service `my-app-svc` on port 8080."

An **Ingress Controller** is the program that reads those rules and actually does the routing. It runs as a Pod inside the cluster and handles all incoming HTTP/HTTPS traffic through a single external IP. K3s ships with **Traefik** as the default Ingress Controller.

```
Internet
    │
    ▼
Traefik (one external IP, ports 80 and 443)
    │
    ├── host: grafana.example.com  →  Service: monitoring-stack-grafana
    ├── host: registry.example.com →  Service: private-registry
    └── host: immich.example.com   →  Service: immich-server
```

A basic Ingress manifest:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: my-app
spec:
  ingressClassName: traefik        # Which Ingress Controller handles this rule
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-svc  # The ClusterIP Service to route to
                port:
                  number: 8080
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls        # cert-manager writes the certificate here
```

### How HTTPS fits in

cert-manager watches for Ingress objects with TLS configuration, requests a certificate from Let's Encrypt, and stores it in the Secret named under `spec.tls[].secretName`. Traefik reads that Secret and terminates TLS — your app sees plain HTTP inside the cluster.

> [!TIP]
> The recommended pattern for HTTP apps in this guide: `ClusterIP` Service + Ingress rule + cert-manager certificate. Only use `LoadBalancer` for non-HTTP protocols like database connections or game servers.

### Useful commands

```bash
# List all Ingress rules in a namespace
kubectl get ingress -n <namespace>

# See which Service an Ingress routes to
kubectl describe ingress <name> -n <namespace>

# Check available IngressClasses (which controllers are installed)
kubectl get ingressclass
```

### Upstream references

- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Ingress controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)

---

## ConfigMaps — app settings without rebuilding

### The problem

Imagine you have a container image for your web app. You want to run it in three different environments: development, staging, and production. Each environment needs different settings — a different database hostname, a different log level, a different feature flag.

You could build three separate container images, one for each environment. But that is wasteful and error-prone. A better approach is to build **one image** and inject the configuration separately at runtime.

### The solution: ConfigMaps

A **ConfigMap** is a Kubernetes object that stores plain text configuration data. You create a ConfigMap with your settings, then tell your Pod to read from it. The Pod gets the config injected either as environment variables or as files — without the config ever being baked into the image.

Think of it like a separate settings file that lives next to your app, rather than being compiled into it.

```
Container image: my-app:1.0.0  (same image in all environments)
ConfigMap: dev-config           (different settings per environment)
ConfigMap: prod-config
```

### Two ways to use a ConfigMap

**Option 1: As environment variables**

The ConfigMap's key-value pairs become environment variables inside the container.

```yaml
# The ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "debug"
  DATABASE_HOST: "postgres.myapp.svc.cluster.local"
  MAX_CONNECTIONS: "10"
```

```yaml
# In your Deployment, reference it like this
spec:
  containers:
    - name: app
      envFrom:
        - configMapRef:
            name: app-config
```

Your app now sees `LOG_LEVEL`, `DATABASE_HOST`, and `MAX_CONNECTIONS` as normal environment variables.

**Option 2: As a mounted file**

Kubernetes writes the ConfigMap's contents as files into a directory inside the container.

```yaml
spec:
  volumes:
    - name: config-volume
      configMap:
        name: app-config
  containers:
    - name: app
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
```

Your app can then read `/etc/config/LOG_LEVEL`, `/etc/config/DATABASE_HOST`, etc. as regular files. This is useful for apps that expect a config file rather than environment variables (Nginx, for example).

### What ConfigMaps are NOT for

> [!WARNING]
> ConfigMaps are **not encrypted**. Anyone with access to the namespace can read them.

Do not put passwords, API keys, tokens, or any sensitive value in a ConfigMap. Use a Kubernetes `Secret` instead. See [`SECRETS.md`](SECRETS.md) for how Secrets work and how to manage them safely.

A good rule of thumb:
- Log levels, hostnames, feature flags, timeouts → ConfigMap
- Passwords, tokens, private keys, certificates → Secret

### Useful commands

```bash
# Create a ConfigMap from a file
kubectl create configmap my-config --from-file=app.conf

# Create a ConfigMap from individual key-value pairs
kubectl create configmap my-config --from-literal=LOG_LEVEL=debug --from-literal=PORT=8080

# Inspect what is in a ConfigMap
kubectl get configmap my-config -o yaml

# Edit a ConfigMap in place
kubectl edit configmap my-config
```

> [!NOTE]
> When you update a ConfigMap that is mounted as a **file**, the running container will eventually see the new file contents — Kubernetes syncs it automatically (usually within a minute). However, environment variables are set once at Pod startup. If you update a ConfigMap used as env vars, you need to restart the Pod for the new values to take effect.

### Upstream references

- [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Configure a Pod to use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)

---

## Secrets — sensitive configuration

A **Secret** is like a ConfigMap, but for sensitive data: passwords, API keys, tokens, TLS certificates. The API and access controls treat them differently — Secrets can be encrypted at rest, and RBAC rules commonly restrict who can read them even in namespaces where the reader can see everything else.

> [!WARNING]
> By default, Kubernetes stores Secrets as base64-encoded strings, not encrypted. Base64 is encoding, not encryption — anyone who can read the etcd database can decode them. To actually encrypt Secrets at rest, you need to enable encryption in the API server configuration, or use a managed cloud cluster that handles this automatically.

### What a Secret looks like

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: my-app
type: Opaque
data:
  username: bXl1c2Vy          # echo -n "myuser" | base64
  password: c3VwZXJzZWNyZXQ=  # echo -n "supersecret" | base64
```

You use Secrets the same way you use ConfigMaps — as environment variables or mounted files:

```yaml
# As environment variables
envFrom:
  - secretRef:
      name: db-credentials

# As a mounted file
volumes:
  - name: db-creds
    secret:
      secretName: db-credentials
```

### Secret types

| Type | Use |
|---|---|
| `Opaque` | Generic key-value pairs — the default |
| `kubernetes.io/tls` | TLS certificates (cert-manager creates these automatically) |
| `kubernetes.io/dockerconfigjson` | Registry credentials for image pulls |

### Keeping Secrets out of git

The biggest practical challenge is that you need the values somewhere, but you cannot commit plaintext credentials to a git repo. Common approaches:

- Create them manually with `kubectl create secret generic ...` and document what fields to create (not the values)
- Use `--set-file` with Helm to pass secret files at install time without embedding them in YAML
- Use SOPS or Sealed Secrets to commit encrypted secrets to git
- Use External Secrets Operator to pull values from a vault at runtime

See [`SECRETS.md`](SECRETS.md) for a detailed walkthrough of each approach.

### Upstream references

- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Secret types](https://kubernetes.io/docs/concepts/configuration/secret/#secret-types)

---

## Persistent Volumes — giving Pods lasting storage

### The problem

Pods are temporary. When a Pod restarts or gets replaced, everything stored inside the container disappears. This is fine for stateless apps, but it is a problem for anything that needs to keep data — a database, an upload directory, a cache.

You need somewhere to store data that survives Pod restarts.

### The solution: PersistentVolumes and PersistentVolumeClaims

Kubernetes splits storage into two separate concepts:

- A **PersistentVolume (PV)** is an actual piece of storage that exists in your infrastructure — a directory on a node, an NFS share, a cloud disk. It describes how much space is available and how it can be used.
- A **PersistentVolumeClaim (PVC)** is a request from a Pod saying "give me some storage with these requirements." It does not know or care which physical storage it gets — it just asks for what it needs.

Think of it like a hotel room analogy. A PV is an actual room the hotel has available. A PVC is a guest's reservation saying "I need a room with a king bed and Wi-Fi." The hotel matches the request to an available room without the guest needing to know the room number in advance.

```
Pod → PersistentVolumeClaim → PersistentVolume → actual storage
         "I need 10 GB"          "I have 20 GB"    (NFS, local disk, cloud)
```

### What a PVC looks like

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-storage
  namespace: my-app
spec:
  accessModes:
    - ReadWriteOnce    # Only one node can write at a time
  resources:
    requests:
      storage: 10Gi
  storageClassName: local-path   # Which type of storage to use
```

Then in a Deployment or StatefulSet, you mount the PVC as a volume:

```yaml
spec:
  volumes:
    - name: db-data
      persistentVolumeClaim:
        claimName: database-storage   # ← reference the PVC by name
  containers:
    - name: database
      volumeMounts:
        - name: db-data
          mountPath: /var/lib/data    # ← where it appears inside the container
```

The data written to `/var/lib/data` inside the container now persists across Pod restarts.

### Access modes

| Mode | Meaning | Typical use |
|---|---|---|
| `ReadWriteOnce` | One node can read and write | Databases, single-instance apps |
| `ReadOnlyMany` | Many nodes can read, no writing | Shared config files, static assets |
| `ReadWriteMany` | Many nodes can read and write | Shared uploads, NFS-backed storage |

> [!NOTE]
> Not all storage types support all access modes. Local disk storage only supports `ReadWriteOnce`. NFS and some cloud storage support `ReadWriteMany`. Check your storage provider's docs.

### Storage classes

A **StorageClass** defines how storage is provisioned. When you create a PVC and name a StorageClass, Kubernetes automatically creates the PV for you. K3s ships with `local-path` as the default StorageClass, which stores data in a directory on the node where the Pod runs.

```bash
# See what StorageClasses are available
kubectl get storageclass

# See all PVs in the cluster
kubectl get pv

# See all PVCs in a namespace
kubectl get pvc -n my-namespace
```

> [!WARNING]
> If you use `local-path` storage and the Pod moves to a different node (due to a node failure or reschedule), it cannot access the data from the original node. For anything important, use networked storage like NFS, or a cloud provider's storage class.

### Upstream references

- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [StorageClasses](https://kubernetes.io/docs/concepts/storage/storage-classes/)

---

## Health probes — knowing when a Pod is ready

### The problem

A container can be running without actually being ready to serve traffic. Maybe the app is still loading configuration, waiting for a database connection, or warming up a cache. If Kubernetes sends requests to a Pod the moment it starts, users may see errors.

Conversely, a container can get stuck in a broken state — a deadlock, a full queue, a corrupted internal state — without actually crashing. If Kubernetes does not know about this, it keeps sending traffic to a broken Pod.

### The solution: Three types of probes

Kubernetes supports three probes you can add to any container:

| Probe | Question it asks | What Kubernetes does if it fails |
|---|---|---|
| **Readiness** | "Is this container ready to accept traffic?" | Removes the Pod from the Service's endpoint list — stops sending it traffic |
| **Liveness** | "Is this container still working?" | Restarts the container |
| **Startup** | "Has this container finished starting up?" | Delays readiness and liveness checks until startup succeeds |

### Readiness probe — traffic control

Use a readiness probe to tell Kubernetes when your app is actually ready to serve requests. Until the probe passes, the Pod will not receive any traffic from a Service, even if the container is running.

```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5    # Wait 5s before first check
  periodSeconds: 10         # Check every 10s
```

This is the most important probe to add. Without it, rolling updates can briefly send traffic to a new Pod before it is ready.

### Liveness probe — crash detection

Use a liveness probe to detect when an app has gotten stuck in a broken state that it cannot recover from on its own. Kubernetes will restart the container.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30   # Give the app time to start before checking
  periodSeconds: 30
  failureThreshold: 3       # Restart after 3 consecutive failures
```

> [!WARNING]
> Be careful with liveness probes. If you set one up incorrectly — for example, making it check something that legitimately takes time under load — Kubernetes will restart healthy Pods and make your outage worse. Only use a liveness probe when the container can genuinely get stuck and needs a restart to recover.

### Startup probe — slow-starting apps

Some apps (Java applications, large models, apps doing database migrations) take a long time to start. If you set `initialDelaySeconds` too short on a liveness probe, Kubernetes will restart the app before it finishes starting.

A startup probe solves this cleanly: liveness and readiness probes are paused until the startup probe first succeeds.

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30      # Allow up to 30 failures (5 minutes total at 10s intervals)
  periodSeconds: 10
```

### Probe types

Probes do not have to be HTTP calls. There are three ways to check:

```yaml
# HTTP GET — success if status code is 200–399
httpGet:
  path: /healthz
  port: 8080

# TCP socket — success if the port accepts a connection
tcpSocket:
  port: 5432

# Exec — success if the command exits with code 0
exec:
  command: ["pg_isready", "-U", "postgres"]
```

### Useful commands

```bash
# See the probe status for a Pod
kubectl describe pod <pod-name> -n <namespace>

# Check why a Pod is not becoming ready
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

### Upstream references

- [Configure liveness, readiness, and startup probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)

---

## Resource requests and limits

### The problem

If you run many containers on the same machine without any constraints, one misbehaving container could consume all available CPU and memory, starving every other container on that node.

You also need a way to tell Kubernetes how much resource a container needs, so it can decide which node to schedule it on.

### The solution: Requests and limits

Every container can (and should) declare two things for CPU and memory:

- A **request** is the minimum the container needs. Kubernetes uses this to decide where to schedule the Pod — it will only place the Pod on a node that has enough free resources to meet the request.
- A **limit** is the maximum the container is allowed to use. If the container tries to exceed this, Kubernetes throttles it (CPU) or kills it (memory).

```yaml
resources:
  requests:
    memory: "128Mi"   # Guarantee at least 128 MiB of memory
    cpu: "100m"       # Guarantee at least 100 millicores (0.1 CPU)
  limits:
    memory: "256Mi"   # Never allow more than 256 MiB of memory
    cpu: "500m"       # Never allow more than 500 millicores (0.5 CPU)
```

### CPU units

CPU is measured in **millicores**. 1000m = 1 CPU core.

- `100m` = 10% of one CPU core
- `500m` = half a CPU core
- `2000m` (or just `"2"`) = 2 CPU cores

If a container hits its CPU limit, Kubernetes throttles it — it slows down but keeps running.

### Memory units

Memory uses standard binary units:
- `Mi` = mebibytes (1 Mi = 1,048,576 bytes, roughly 1 MB)
- `Gi` = gibibytes (roughly 1 GB)

If a container hits its memory limit, Kubernetes kills it with an OOMKilled (Out of Memory Killed) error and restarts it. This is why requests and limits matter: an under-resourced container will keep restarting.

### Requests vs limits: what happens at each

| Situation | What Kubernetes does |
|---|---|
| Container uses less than its request | Normal — request is just a scheduling guarantee |
| Container uses between request and limit | Allowed — extra resources are used when available |
| Container hits CPU limit | Throttled — slows down, does not crash |
| Container hits memory limit | Killed and restarted (OOMKilled) |
| Node runs out of memory | Pods whose memory usage exceeds their request are evicted first |

### Why requests matter for HPA

The HorizontalPodAutoscaler (HPA) calculates CPU utilization as a percentage of the container's CPU **request**. If you do not set a request, the HPA cannot compute utilization and will show `<unknown>` instead of a percentage.

> [!TIP]
> As a starting point: set your request to the average resource usage you observe, and set your limit to 2–4x the request. Tune from there using real metrics. Setting limits too low causes unnecessary OOMKills; setting them too high wastes cluster capacity.

### Useful commands

```bash
# See current resource usage per Pod
kubectl top pods -n <namespace>

# See current resource usage per node
kubectl top nodes

# Check resource requests and limits on a Deployment
kubectl describe deployment <name> -n <namespace>
```

> [!NOTE]
> `kubectl top` requires the Metrics Server to be running. K3s ships with it enabled by default.

### Upstream references

- [Resource management for Pods and containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Metrics Server](https://github.com/kubernetes-sigs/metrics-server)

---

## Node scheduling — controlling where Pods run

### The problem

By default, Kubernetes places Pods on whichever node has enough available CPU and memory. Most of the time this is fine. But sometimes you need more control:

- You have a node with an expensive GPU and want only GPU workloads there
- You want database Pods to land only on nodes with fast SSDs
- You want to spread your app's replicas across different physical machines so a single failure does not take everything down
- You want a monitoring agent to run on every single node

Kubernetes provides three tools for this: **taints**, **tolerations**, and **affinity rules**.

### Taints and tolerations

Think of a **taint** as a "do not park here" sign on a node. Once a node has a taint, Pods will not be scheduled there unless they explicitly say they are allowed to be there — that explicit permission is a **toleration**.

**Adding a taint to a node:**

```bash
# Mark a node as dedicated for GPU workloads
kubectl taint nodes my-gpu-node dedicated=gpu:NoSchedule
```

This says: "Do not schedule anything on `my-gpu-node` unless it has a toleration for `dedicated=gpu`."

**Adding a toleration to a Pod:**

```yaml
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
```

A Pod with this toleration is allowed onto the tainted node. A Pod without it will be scheduled elsewhere.

**Removing a taint:**

```bash
kubectl taint nodes my-gpu-node dedicated=gpu:NoSchedule-
```

(The `-` at the end removes the taint.)

**The three taint effects:**

| Effect | What it does |
|---|---|
| `NoSchedule` | New Pods without a matching toleration will not be scheduled on this node |
| `PreferNoSchedule` | Kubernetes will try to avoid placing Pods here, but will if nothing else is available |
| `NoExecute` | New Pods are blocked AND any existing Pods without a matching toleration are evicted |

> [!NOTE]
> K3s automatically adds a `node-role.kubernetes.io/control-plane:NoSchedule` taint to server nodes. This is why you see toleration entries in DaemonSet manifests — without them, the DaemonSet would skip the server node.

### Node affinity — scheduling based on node labels

Taints and tolerations are about permission (can this Pod run here?). **Node affinity** is about preference — you are telling the scheduler "I want this Pod on a node with certain characteristics."

Nodes have labels. You can add your own:

```bash
# Label a node to indicate it has fast SSD storage
kubectl label nodes my-fast-node storage-type=ssd
```

Then in a Deployment, express a preference or requirement:

```yaml
affinity:
  nodeAffinity:
    # Hard requirement — Pod WILL NOT schedule if no matching node exists
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: storage-type
              operator: In
              values: ["ssd"]
    # Soft preference — scheduler will try to honor this but won't block
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
            - key: kubernetes.io/arch
              operator: In
              values: ["amd64"]
```

The naming is awkward but the distinction is important:
- `required...` = hard rule, Pod stays `Pending` if no node matches
- `preferred...` = soft preference, Pod still schedules even if no node matches

### Pod anti-affinity — spreading replicas apart

If you run three replicas of a web app and they all land on the same physical node, a single node failure takes all three down at once. **Pod anti-affinity** tells the scheduler to spread replicas across different nodes.

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          topologyKey: kubernetes.io/hostname   # "spread across different hostnames"
          labelSelector:
            matchLabels:
              app: web-demo                     # "relative to other Pods with this label"
```

This says: "When placing a `web-demo` Pod, prefer a node that does not already have a `web-demo` Pod on it." The `preferredDuring...` form means the scheduler will try but will not block the Pod from running if spreading is impossible (e.g. you have more replicas than nodes).

### Useful commands

```bash
# See the labels on your nodes
kubectl get nodes --show-labels

# See what taints a node has
kubectl describe node <node-name> | grep -A5 Taints

# Add a label to a node
kubectl label nodes <node-name> <key>=<value>
```

### Upstream references

- [Taints and tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Node affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity)
- [Pod affinity and anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity)

---

## CRDs and the Operator pattern

### The problem

Kubernetes ships with built-in object types: Pods, Deployments, Services, ConfigMaps, and so on. These cover most use cases, but sometimes an application needs more. A database cluster, for example, is much more complex than a plain Deployment — it needs to manage leader election, replication, rolling upgrades that do not lose data, and automatic failover.

Kubernetes cannot know in advance how to manage every possible application. So it provides a way to extend itself.

### Custom Resource Definitions (CRDs)

A **Custom Resource Definition (CRD)** is how you add new object types to Kubernetes. Once a CRD is installed, you can create, list, update, and delete your custom objects the same way you would any built-in type.

Think of it like a plugin system. Kubernetes is the base platform, and CRDs let tools extend it with their own vocabulary.

For example, cert-manager adds these new types to your cluster:

```bash
Certificate       # "please get me a TLS certificate for this domain"
ClusterIssuer     # "here is how to talk to Let's Encrypt"
CertificateRequest
```

Once cert-manager is installed, you can write a manifest like this, and Kubernetes knows what to do with it:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate        # <-- this is a CRD, not built-in
metadata:
  name: my-tls-cert
spec:
  dnsNames:
    - myapp.example.com
  issuerRef:
    name: letsencrypt-prod
```

Without cert-manager's CRD installed, Kubernetes would reject that manifest with an "unknown resource type" error.

Several components in this repo install CRDs:

- **Actions Runner Controller** registers `AutoscalingRunnerSet` and `EphemeralRunnerSet`
- **OpenSearch Operator** registers `OpenSearchCluster`
- **cert-manager** registers `Certificate`, `ClusterIssuer`, and others

```bash
# See every CRD installed in your cluster
kubectl get crd

# Inspect what fields a particular CRD accepts
kubectl describe crd certificates.cert-manager.io
```

### The Operator pattern

A CRD only defines the shape of a new object type. On its own, creating a `Certificate` resource does nothing — Kubernetes does not know what "get me a TLS certificate" actually means.

That is where an **Operator** comes in. An Operator is a controller (a program running inside the cluster) that:

1. Watches for changes to specific custom resources
2. Figures out what the real world should look like based on what you declared
3. Takes action to make the real world match

This "watch, compare, act" loop is called **reconciliation**, and it runs continuously.

Here is a concrete example with cert-manager:

1. You apply a `Certificate` resource asking for a TLS cert for `myapp.example.com`
2. The cert-manager Operator notices the new `Certificate` object
3. It contacts Let's Encrypt, completes the verification challenge, and downloads the certificate
4. It stores the certificate in a Kubernetes `Secret` in your namespace
5. If the certificate is about to expire, the Operator automatically renews it

You never ran any cert renewal commands. The Operator handled it.

The Operator pattern works especially well for stateful applications with complex operational requirements — database clusters, message queues, search engines. OpenSearch and cert-manager in this repo are both managed by Operators.

> [!NOTE]
> You do not need to write an Operator to use Kubernetes. But recognizing the pattern explains why installing an application often involves two Helm charts: one for the CRD + Operator (the "brains"), and one for the actual application resource (the "instructions").

### Upstream references

- [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [cert-manager concepts](https://cert-manager.io/docs/concepts/)

---

## RBAC — controlling who can do what

By default, the service account a Pod runs as has very limited permissions. If a workload needs to call the Kubernetes API — to list Pods, read Secrets, or update a ConfigMap — you have to explicitly grant it permission. This is done with **Role-Based Access Control (RBAC)**.

The core idea is a three-part sentence: **"Subject can perform Verb on Resource in Namespace."**

- **Subject**: who is asking (a `ServiceAccount`, a user, or a group)
- **Verb**: what action (`get`, `list`, `watch`, `create`, `update`, `patch`, `delete`)
- **Resource**: what object type (`pods`, `secrets`, `configmaps`, `deployments`, etc.)

### The four RBAC objects

| Object | Scope | What it does |
|---|---|---|
| `Role` | Namespace | Defines a set of permissions within one namespace |
| `ClusterRole` | Cluster-wide | Defines permissions across all namespaces, or for cluster-scoped objects like Nodes |
| `RoleBinding` | Namespace | Grants a Role to a subject in a specific namespace |
| `ClusterRoleBinding` | Cluster-wide | Grants a ClusterRole to a subject across the entire cluster |

### A minimal example

A background job that only needs to read ConfigMaps in its own namespace:

```yaml
# 1. A dedicated identity for the workload
apiVersion: v1
kind: ServiceAccount
metadata:
  name: config-reader
  namespace: my-app
---
# 2. The permissions (what verbs are allowed on which resources)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
  namespace: my-app
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
---
# 3. Link the identity to the permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-configmaps
  namespace: my-app
subjects:
  - kind: ServiceAccount
    name: config-reader
    namespace: my-app
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
```

Then in the Deployment, reference the ServiceAccount:

```yaml
spec:
  serviceAccountName: config-reader
```

### Important defaults to know

- Every Pod uses the `default` ServiceAccount in its namespace unless you specify one. The `default` ServiceAccount has no extra permissions.
- Set `automountServiceAccountToken: false` on Pods that do not call the Kubernetes API — this removes the credential from the Pod entirely.
- Avoid `cluster-admin` for regular workloads. It grants unrestricted access to the entire cluster.

> [!NOTE]
> Most apps you deploy do not need any Kubernetes API access at all. Only components like Operators, GitOps controllers, monitoring agents, and ARC runners need RBAC grants — because they deliberately manage or read cluster state.

### Useful commands

```bash
# Check what permissions a service account has
kubectl auth can-i --list --as=system:serviceaccount:my-app:config-reader -n my-app

# See all RoleBindings in a namespace
kubectl get rolebindings -n my-app

# See all ClusterRoleBindings
kubectl get clusterrolebindings
```

### Upstream references

- [RBAC authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [RBAC good practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
- [Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
