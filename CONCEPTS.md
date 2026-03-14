# Kubernetes Core Concepts

This guide explains the foundational Kubernetes ideas that come up again and again across this repo. If you are new to Kubernetes, read this before diving into the installation steps — understanding these concepts will make everything else click faster.

The guidance below was checked against the official Kubernetes docs on **March 14, 2026**.

---

## Table of contents

- [What is a Pod?](#what-is-a-pod)
- [Services — how apps talk to each other](#services--how-apps-talk-to-each-other)
- [ConfigMaps — app settings without rebuilding](#configmaps--app-settings-without-rebuilding)
- [Node scheduling — controlling where Pods run](#node-scheduling--controlling-where-pods-run)
- [CRDs and the Operator pattern](#crds-and-the-operator-pattern)

---

## What is a Pod?

Before understanding the concepts below, it helps to know what a **Pod** is, because everything in Kubernetes revolves around them.

A Pod is the smallest unit Kubernetes manages. Think of it as a wrapper around one or more containers that should always run together on the same machine. In practice, most Pods contain a single container (your app), though some add a small helper container alongside it.

The key thing to know about Pods: **they are temporary**. If a Pod crashes, Kubernetes replaces it with a new one — but the new Pod gets a different IP address and a different name. This is intentional. Kubernetes is designed around the idea that individual containers come and go, and the system keeps things running regardless.

This "temporary" nature is what motivates everything else on this page.

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
