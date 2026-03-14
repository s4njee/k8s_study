# Kubernetes Cluster Operations

This guide covers day-two operations for a K3s cluster: upgrading nodes and backing up and restoring cluster state.

The guidance below was checked against the official K3s docs on **March 14, 2026**.

---

## Table of contents

- [Upgrade K3s](#upgrade-k3s)
- [Backup and restore](#backup-and-restore)

---

## Upgrade K3s

K3s upgrades are safe when done in the right order: **upgrade the server node first, then each agent node**. Never upgrade an agent before the server it connects to — a newer agent talking to an older server is unsupported and can cause issues.

### Before you upgrade

- Check the [K3s release notes](https://github.com/k3s-io/k3s/releases) for breaking changes between your current version and the target.
- Back up the cluster state first (see [Backup and restore](#backup-and-restore) below).
- Check current versions across all nodes:

```bash
kubectl get nodes -o wide
```

### Manual upgrade

Re-run the installer with a pinned version. K3s detects that it is already installed and upgrades in place.

**Server node:**

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=<target_version> sh -
```

**Each agent node:**

```bash
curl -sfL https://get.k3s.io | \
  K3S_URL=https://<server_ip>:6443 \
  K3S_TOKEN=<node_token> \
  INSTALL_K3S_VERSION=<target_version> sh -
```

Verify the upgrade completed on each node:

```bash
kubectl get nodes -o wide
```

### Draining a node before upgrading

If you want workloads to shift off a node before you upgrade it (to avoid brief disruption), drain it first:

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

This evicts all Pods from the node and marks it unschedulable. After the upgrade, mark it schedulable again:

```bash
kubectl uncordon <node-name>
```

### Automated upgrade with the System Upgrade Controller

For clusters where you want Kubernetes-native rolling upgrades, K3s has an official [System Upgrade Controller](https://github.com/rancher/system-upgrade-controller). You create a `Plan` custom resource that specifies the target channel or version, and the controller coordinates the rolling upgrade across nodes automatically.

```bash
# Install the controller
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/latest/download/system-upgrade-controller.yaml
```

After installing the controller, create a `Plan` manifest that targets your desired K3s channel or version. The controller will handle draining, upgrading, and uncordoning nodes in sequence.

See the [System Upgrade Controller README](https://github.com/rancher/system-upgrade-controller) for Plan examples.

### Upstream references

- [K3s upgrades](https://docs.k3s.io/upgrades)
- [System Upgrade Controller](https://github.com/rancher/system-upgrade-controller)
- [K3s release notes](https://github.com/k3s-io/k3s/releases)

---

## Backup and restore

K3s stores all cluster state in a datastore. The datastore type depends on how you installed K3s:

| Setup | Datastore | Default location |
|---|---|---|
| Single server (default) | SQLite | `/var/lib/rancher/k3s/server/db/` |
| Multi-server with `--cluster-init` | Embedded etcd | `/var/lib/rancher/k3s/server/db/etcd/` |

### SQLite backup (default single-server setup)

K3s does not have a built-in automated snapshot mechanism for SQLite. The simplest reliable approach is to stop K3s, copy the entire server state directory, then restart.

```bash
# 1. Stop K3s
sudo systemctl stop k3s

# 2. Copy the server state directory to a safe location
sudo cp -r /var/lib/rancher/k3s/server /your/backup/location/k3s-server-$(date +%Y%m%d)

# 3. Restart K3s
sudo systemctl start k3s
```

To automate this, put it in a cron job and copy the backup directory to off-node storage (NFS, S3, etc.).

**Restoring from a SQLite backup:**

```bash
# 1. Stop K3s
sudo systemctl stop k3s

# 2. Replace the current server directory with the backup
sudo rm -rf /var/lib/rancher/k3s/server
sudo cp -r /your/backup/location/k3s-server-<date> /var/lib/rancher/k3s/server

# 3. Restart K3s
sudo systemctl start k3s
```

### Embedded etcd snapshots (multi-server setup)

If you installed K3s with `--cluster-init`, K3s uses embedded etcd and takes automated snapshots automatically:

- **Frequency:** every 12 hours by default
- **Retention:** 5 snapshots by default
- **Location:** `/var/lib/rancher/k3s/server/db/snapshots/`

**Take a manual snapshot:**

```bash
k3s etcd-snapshot save --name pre-upgrade-$(date +%Y%m%d)
```

**List available snapshots:**

```bash
k3s etcd-snapshot list
```

**Restore from a snapshot:**

> [!WARNING]
> Restoring a snapshot replaces the current cluster state. This cannot be undone. All changes made after the snapshot was taken will be lost.

```bash
# 1. Stop K3s on all nodes first

# 2. On the server node, restore from the snapshot
k3s server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/k3s/server/db/snapshots/<snapshot-name>

# 3. Restart K3s
sudo systemctl start k3s
```

For multi-server clusters, after the restore completes on the first server, rejoin the remaining server nodes.

### Application-level backup with Velero

K3s datastore backups capture the cluster's raw state. A complementary approach is to back up at the application level — backing up the Kubernetes objects (Deployments, Services, ConfigMaps, Secrets, PVCs) and optionally snapshotting persistent volume contents.

[Velero](https://velero.io) is the standard tool for this.

**Install Velero** (example using AWS S3 as the backup destination):

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.x.x \
  --bucket <your-backup-bucket> \
  --backup-location-config region=<region>
```

Velero also supports MinIO (self-hosted S3-compatible storage), Google Cloud Storage, and Azure Blob Storage.

**Take an on-demand backup:**

```bash
# Back up specific namespaces
velero backup create my-backup --include-namespaces app,postgres

# Back up the entire cluster
velero backup create full-cluster-backup
```

**List and inspect backups:**

```bash
velero backup get
velero backup describe my-backup
velero backup logs my-backup
```

**Restore from a backup:**

```bash
velero restore create --from-backup my-backup

# Check restore status
velero restore get
velero restore describe <restore-name>
```

### Minimum viable backup strategy for a homelab

If you want a simple starting point before setting up full Velero:

1. **Cluster state:** A weekly cron job that stops K3s, copies `/var/lib/rancher/k3s/server` to your NFS share, and restarts K3s.
2. **Database data:** A daily `pg_dump` for PostgreSQL, or equivalent for other databases.
3. **Store backups off the cluster node.** A backup on the same machine that hosts your cluster does not protect against disk failure.

### Upstream references

- [K3s backup and restore](https://docs.k3s.io/datastore/backup-restore)
- [K3s etcd snapshots](https://docs.k3s.io/datastore/ha-embedded)
- [Velero documentation](https://velero.io/docs/)
- [Velero GitHub](https://github.com/vmware-tanzu/velero)
