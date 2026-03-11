# metallb

MetalLB load balancer for k3s, providing LoadBalancer IP assignment for bare metal clusters.

## Files

| File | Description |
|------|-------------|
| `ipaddresspool.yaml` | IP address pool for LoadBalancer services |
| `l2advertisement.yaml` | L2 advertisement configuration |

## Installation

MetalLB is installed directly from the official manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml
```

Wait for all pods to be ready:

```bash
kubectl get pods -n metallb-system -w
```

Then apply the configuration:

```bash
kubectl apply -f ipaddresspool.yaml
kubectl apply -f l2advertisement.yaml
```

## Configuration

### IP Address Pool

MetalLB is configured to assign IPs from the following range:

| Setting | Value |
|---------|-------|
| Pool name | `first-pool` |
| IP range | `192.168.20.101` – `192.168.20.200` |
| Mode | L2 (ARP) |

### Currently assigned IPs

| Service | Namespace | IP | Description |
|---------|-----------|-----|-------------|
| ingress-nginx-controller-loadbalancer | ingress-nginx | dynamic | HTTP/HTTPS ingress |
| mosquitto-service | mosquitto | 192.168.20.190 | MQTT broker |
| technitium-dns-service | technitium | 192.168.20.199 | DNS server |

> Update this table as new LoadBalancer services are added.

## Upgrading

Check the current version:

```bash
kubectl get deploy controller -n metallb-system \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Find the latest version at https://github.com/metallb/metallb/releases, then apply the new manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/vX.Y.Z/config/manifests/metallb-native.yaml
```

Verify the rollout:

```bash
kubectl rollout status deployment/controller -n metallb-system
kubectl get pods -n metallb-system
```

> ⚠️ Always check the [MetalLB upgrade notes](https://metallb.universe.tf/release-notes/) before upgrading.

## Requesting a specific IP

To assign a specific IP to a LoadBalancer service, add the annotation to the service manifest:

```yaml
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.20.190
```

Or use the MetalLB annotation for newer versions:

```yaml
metadata:
  annotations:
    metallb.universe.tf/loadBalancerIPs: 192.168.20.190
```

## Notes

- MetalLB manifests are not stored in this repo — the official URL is the source of truth
- L2 mode uses ARP — only one node handles traffic for each IP at a time (no true HA)
- Ensure assigned IPs are outside your router's DHCP range to avoid conflicts