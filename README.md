# Kubernetes Study Guide

## Install k8s distribution (k3s or k3d)

### Set up k3d with internal registry

    k3d cluster create <clustername> --kubeconfig-update-default --kubeconfig-switch-context --registry-create <registry_name> -v <host_shared_folder>:/k8s

### Or set up k3s

#### Server Installation
    curl -sfL https://get.k3s.io | sh -
#### Node installation

On the server node, get the node-token

    sudo cat /var/lib/rancher/k3s/server/node-token

On the worker nodes, install k3s with

    curl -sfL https://get.k3s.io | K3S_URL=https://<kmaster_IP_from_above>:6443 K3S_TOKEN=<token_from_above> sh -
---
### Install k8s dashboard

    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.6.1/aio/deploy/recommended.yaml

---

### Set up k8s sample user

In the `dashboard_user` directory, `user.yaml` creates a sample user named `admin-user`.

The `cluster_role_binding.yaml` is used to bind `admin_user` to the role `cluster_admin` which allows the user to access
the cluster.

#### Get dashboard token
The token to login to the dashboard is generated with the `dashboard-token.yaml` and can be retreived with

    kubectl describe secret dashboard-token

#### Get dashboard access

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

An example of a nfs PersistentVolume and PersistentVolumeClaim can be found in the PV folder


---
### Set up MetalLB

MetalLB allows the creation of `LoadBalancer` services without provisioning one from a cloud provider

    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml

The address pool that the `LoadBalancer` services use can be modified and applied with `metallb/addresspool.yaml`


---

### Set up Docker Registry

If the k8s distribution was created with k3d with a registry specified on creation, this section is not needed.

The `registry/registry.yaml` file contains the deployment for a docker registry on the control plane node. An example of setting resource limits is also seen here.

#### Allow access to insecure registry with k3s
If k3s is being used, an entry needs to be added to the `/etc/rancher/k3s/registries.yaml` file on each node running k3s

    mirrors:
      "<hostname>:5000":
        endpoint:
          - "http://<ip-address>:5000"

---

### Set up PostgreSQL

A single instance postgresql database can be set up with the files inside the postgres folder.

#### ConfigMap
The `pg-config.yaml` is a `ConfigMap` that sets the db name, username, and password for the postgres database.

#### Persistent Volume and Claim
The `pg-pv-local.yaml` file creates a persistent volume and claim that can be accessed by a single node.

#### LoadBalancer Service
The `pg-loadbalancer.yaml` file creates a service with metallb to allow access to the postgres database from outside the cluster.

#### Deployment
The `pg-deployment.yaml` file creates the actual postgres deployment as a single container deployment.