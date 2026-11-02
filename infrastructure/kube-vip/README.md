# kube-vip

Managed by Flux (HelmRelease). Static manifests maintained manually on control plane nodes for the control plane VIP.

## Control plane VIP (`192.168.2.30`)

Static pod manifests on each control plane node are used to manage the control plane VIP — this cannot be done via a DaemonSet because kubelet starts before the API server is reachable, and a DaemonSet requires the scheduler/API to bootstrap itself.

On each control plane node (`192.168.2.{31,32}`):

```bash
export VIP=192.168.2.30
KVVERSION=v1.3.1
export INTERFACE=lo

alias kube-vip="docker run --network host --rm ghcr.io/kube-vip/kube-vip:$KVVERSION"

# control plane 1 (node IP 192.168.2.31)
kube-vip manifest pod \
    --interface $INTERFACE \
    --address $VIP \
    --controlplane \
    --log 5 \
    --bgp \
    --sourceIP 192.168.2.31 \
    --bgpRouterID 192.168.2.31 \
    --localAS 64500 \
    --peerAddress 192.168.2.1 \
    --peerAS 64501 | tee /etc/kubernetes/manifests/kube-vip.yaml

# control plane 2 (node IP 192.168.2.32)
kube-vip manifest pod \
    --interface $INTERFACE \
    --address $VIP \
    --controlplane \
    --log 5 \
    --bgp \
    --sourceIP 192.168.2.32 \
    --bgpRouterID 192.168.2.32 \
    --localAS 64500 \
    --peerAddress 192.168.2.1 \
    --peerAS 64501 | tee /etc/kubernetes/manifests/kube-vip.yaml
```

## Service load balancer (Flux)

A [HelmRelease](./helm-release.yaml) managed by Flux handles service LB on worker nodes only (no control plane nodes). This is a separate kube-vip instance from the static pods — it only manages `LoadBalancer`-type services with BGP routing.

```bash
# View current service VIPs
kubectl get svc -A | grep LoadBalancer

# Check kube-vip logs for service LB activity
kubectl logs -n kube-system -l app.kubernetes.io/name=kube-vip -f
```

## Architecture

| Component | Mode | Purpose |
|-----------|------|---------|
| Control plane static manifests | Static pod (`/etc/kubernetes/manifests/`) | API server VIP + BGP leader election across control planes |
| HelmRelease DaemonSet | Flux (Helm) | Service load balancing for `LoadBalancer`-type services on worker nodes only |

- **BGP peer:** 192.168.2.1 (UniFi router), AS 64501
- **Local AS:** 64500
- **VIP:** 192.168.2.30/32
- **Interface:** lo (loopback) / ens18 (upstream)
