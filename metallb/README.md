# metallb

MetalLB load balancer for k3s, providing LoadBalancer service support on bare metal.

## Files

| File | Description |
|------|-------------|
| `ipaddresspool.yaml` | IP address pool assigned to LoadBalancer services |
| `l2advertisement.yaml` | L2 advertisement config linking the pool to the network |

## IP Address Pool

| Range | Usage |
|-------|-------|
| `192.168.20.101` – `192.168.20.200` | Available for LoadBalancer services |

Currently assigned static IPs:

| IP | Service |
|----|---------|
| `192.168.20.190` | Mosquitto MQTT (`mosquitto` namespace) |

> Update this table whenever a new static LoadBalancer IP is assigned.

## Dependencies

- A network that supports L2/ARP (standard home/homelab network)

## Installation

MetalLB is installed directly from the official manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml
```

Wait for all pods to be ready:

```bash
kubectl get pods -n metallb-system
```

Then apply the pool and advertisement config:

```bash
kubectl apply -f ipaddresspool.yaml
kubectl apply -f l2advertisement.yaml
```

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

> ⚠️ Always check the [MetalLB upgrade notes](https://metallb.io/release-notes/) before upgrading.

## Assigning a static IP to a service

To request a specific IP for a LoadBalancer service, add `loadBalancerIP` to the service spec:

```yaml
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.20.xxx
```

The IP must be within the `192.168.20.101–192.168.20.200` range defined in `ipaddresspool.yaml`.

## Notes

- MetalLB manifests are not stored in this repo — the official URL is the source of truth
- L2 mode works by responding to ARP requests on the local network — no BGP or special router config needed
- Only one node announces each IP at a time (leader election) — failover happens automatically if that node goes down
- The IP pool must not overlap with your DHCP server's range to avoid conflicts