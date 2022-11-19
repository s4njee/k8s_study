# Kubernetes Study Guide

### Install k8s dashboard

    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.6.1/aio/deploy/recommended.yaml

---

### Set up k8s sample user

In the `dashboard_user` directory, `user.yaml` creates a sample user named `admin-user`.

The `cluster_role_binding.yaml` is used to bind `admin_user` to the role `cluster_admin` which allows the user to access
the cluster.

The token to login to the dashboard is generated with

    kubectl -n kubernetes-dashboard create token admin-user

The dashboard can be accessed using the command `kubectl proxy` and the url

> [http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/)

---

### Set up the NFS Provisioner

To use persistent volumes later with postgresql, we can set up the nfs_provisioner using helm.

    helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
    helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
        --set nfs.server=IP_ADDRESS \
        --set nfs.path=NFS_PATH

This creates a `StorageClass` named `nfs-client` which can be used in a `PersistentVolume`.

---

### Set up PostgreSQL

A single instance postgresql database can be set up with the files inside the postgres folder.

The `postgres-config.yaml` is a `ConfigMap` that sets the db name, username, and password for the postgres database.

The `postgres_pv.yaml` file creates a persistent volume that can be accessed by a single node.