# Kubernetes Troubleshooting Guide

This guide is a practical triage playbook for figuring out why something in the cluster is not working.

The guidance below was checked against the official Kubernetes docs on **March 14, 2026**. K3s-specific notes are called out separately.

## The first five minutes

Start broad before you go deep:

```bash
kubectl get pods -A
kubectl get events -A --sort-by=.metadata.creationTimestamp
kubectl get nodes -o wide
kubectl get svc,ingress -A
kubectl get pvc -A
```

Then narrow to the namespace and object that is failing:

```bash
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --all-containers
kubectl logs <pod> -n <namespace> --previous
```

Most Kubernetes problems fall into one of these buckets:

- scheduling
- image pull or startup
- probes or runtime crashes
- service discovery and networking
- storage binding or mount problems
- node health

## Common symptoms

### Pod is stuck in `Pending`

Check:

```bash
kubectl describe pod <pod> -n <namespace>
kubectl get pvc -n <namespace>
kubectl describe pvc <claim> -n <namespace>
kubectl describe node <node>
```

Common causes:

- resource requests are larger than what any node can provide
- required PVC is still `Pending`
- `nodeSelector`, affinity, taints, or tolerations do not match available nodes
- image pull secrets or volumes reference missing objects

### Pod is in `CrashLoopBackOff`

Check:

```bash
kubectl logs <pod> -n <namespace> --previous
kubectl describe pod <pod> -n <namespace>
kubectl get pod <pod> -n <namespace> -o yaml
```

Common causes:

- the container process exits immediately
- probe configuration is too aggressive
- config or secret values are missing or malformed
- the container cannot write to the filesystem it expects

If the crash is too fast to inspect with `exec`, `kubectl debug` is often easier.

### Pod is in `ImagePullBackOff` or `ErrImagePull`

Check:

```bash
kubectl describe pod <pod> -n <namespace>
kubectl get secret -n <namespace>
```

Common causes:

- image name or tag is wrong
- private registry credentials are missing or invalid
- nodes cannot reach the registry
- the image exists, but on a different registry hostname than you configured

For registry-specific patterns in this repo, see [SECRETS.md](SECRETS.md) and the registry sections of [README.md](README.md).

### Service is not reachable

Check the chain in order:

```bash
kubectl get svc <service> -n <namespace>
kubectl get endpointslice -n <namespace> -l kubernetes.io/service-name=<service>
kubectl get pods -n <namespace> --show-labels
kubectl describe svc <service> -n <namespace>
```

What usually goes wrong:

- the `Service` selector does not match the Pod labels
- the Pods are not Ready, so they do not appear in EndpointSlices
- `targetPort` does not match the container's actual listening port
- a `NetworkPolicy` blocks ingress

If you only need to prove the app works before debugging ingress, `kubectl port-forward` is the fastest sanity check.

### DNS is not resolving inside the cluster

Check:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl exec -n <namespace> <pod> -- cat /etc/resolv.conf
kubectl exec -n <namespace> <pod> -- nslookup kubernetes.default
```

If you do not have a suitable Pod to test from:

```bash
kubectl run -it --rm dns-test \
  --restart=Never \
  --image=busybox:1.28 \
  -- nslookup kubernetes.default
```

Common causes:

- CoreDNS is not healthy
- a restrictive egress `NetworkPolicy` forgot to allow DNS
- node resolver settings are broken
- the application image lacks basic DNS tools, making the failure harder to inspect

### Ingress works partially or returns the wrong backend

Check:

```bash
kubectl get ingress -A
kubectl describe ingress <name> -n <namespace>
kubectl get ingressclass
kubectl get svc -n kube-system traefik
```

If TLS is involved, also check:

```bash
kubectl describe certificate <name> -n <namespace>
kubectl get challenge,order -A
```

Common causes:

- wrong `ingressClassName`
- hostname does not resolve to the cluster ingress
- backend `Service` or port is wrong
- TLS secret does not exist yet

### PVC is not binding or the volume will not mount

Check:

```bash
kubectl get pv,pvc,storageclass
kubectl describe pvc <claim> -n <namespace>
kubectl describe pv <pv>
```

Common causes:

- storage class name does not exist
- access mode does not match what the backend supports
- static PV capacity or selectors do not match the claim
- NFS path or server is wrong
- the CSI driver is missing or unhealthy

### Node is `NotReady` or workloads are evicting

Check:

```bash
kubectl get nodes
kubectl describe node <node>
kubectl top node
kubectl top pod -A
```

`kubectl top` requires Metrics Server or another compatible metrics pipeline.

Common causes:

- node pressure: disk, memory, or PID
- kubelet or container runtime problems
- CNI or DNS failures on that node
- the node lost reachability to the API server

## Safe debug tools

These are usually the least destructive options:

```bash
kubectl exec -it <pod> -n <namespace> -- sh
kubectl debug pod/<pod> -n <namespace> -it --image=busybox:1.28 --target=<container>
kubectl port-forward -n <namespace> svc/<service> 8080:80
kubectl get pods -n <namespace> -w
```

Use `kubectl debug` when:

- the container image does not include a shell or useful tools
- the main container crashes too quickly for `exec`
- you need a temporary debug container without rebuilding the image

## K3s-specific checks

If the problem looks cluster-wide, these are high value:

```bash
sudo systemctl status k3s
sudo systemctl status k3s-agent
sudo journalctl -u k3s -f
sudo journalctl -u k3s-agent -f
sudo k3s kubectl get nodes
kubectl get pods -n kube-system
```

Things worth checking in this repo's setup:

- Traefik and ServiceLB resources in `kube-system`
- CoreDNS health in `kube-system`
- NFS CSI driver health if PVCs are involved
- cert-manager resources if TLS issuance is failing

## A useful troubleshooting order

When something is broken, follow this order:

1. Confirm the desired object exists and is in the namespace you think it is.
2. Read `kubectl describe` output before changing anything.
3. Check events and logs.
4. Decide whether the failure is scheduler, image, runtime, network, or storage related.
5. Use the smallest possible debug tool: `logs`, then `describe`, then `exec`, then `debug`.
6. Only edit manifests after you know what failed.

## Related guides

- [WORKLOADS.md](WORKLOADS.md) for controller-specific behavior
- [SECURITY.md](SECURITY.md) for PSA, RBAC, and network policies
- [SECRETS.md](SECRETS.md) for secret handling patterns

## Upstream references

- [Troubleshooting applications](https://kubernetes.io/docs/tasks/debug/debug-application/)
- [Debug Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- [Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
- [Debugging DNS resolution](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)
- [Troubleshooting clusters](https://kubernetes.io/docs/tasks/debug/debug-cluster/)
- [kubectl debug reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_debug/)
