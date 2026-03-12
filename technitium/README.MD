# technitium

Technitium DNS Server — a self-hosted DNS server with blocking, caching, and a web UI.

## Files

| File | Description |
|------|-------------|
| `namespace.yaml` | Dedicated namespace |
| `serviceaccount.yaml` | Dedicated service account |
| `pvc.yaml` | 20Gi Longhorn persistent volume for DNS data, config, and block lists |
| `servicedns.yaml` | LoadBalancer service for DNS traffic (TCP/UDP port 53) |
| `serviceweb.yaml` | ClusterIP service for web UI (port 80 → container 5380) |
| `deployment.yaml` | Technitium DNS server deployment |
| `httproute.yaml` | Traefik Gateway API HTTPRoute for web UI |

## Accessing

https://technitium.petkus.id.lv

## Dependencies

- Traefik Gateway API controller in `traefik` namespace
- Longhorn storage
- MetalLB for LoadBalancer IP assignment

## Deployment

```bash
kubectl apply -f .
```

Verify rollout:

```bash
kubectl rollout status deployment/technitium-dnsserver -n technitium
kubectl get pods -n technitium
```

## Network

| Service | Type | IP | Port | Protocol |
|---------|------|----|------|----------|
| DNS | LoadBalancer | 192.168.20.199 | 53 | TCP + UDP |
| Web UI | ClusterIP | — | 80 (→ 5380) | TCP |
| HTTPRoute | — | — | 443 | HTTPS via Traefik |

`externalTrafficPolicy: Local` is set on the DNS service to preserve the original client IP in DNS query logs.

## Upgrading

Update the image version in `deployment.yaml` and the version label in all files:

```yaml
image: technitium/dns-server:NEW_VERSION
app.kubernetes.io/version: "NEW_VERSION"
```

Then apply:

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/technitium-dnsserver -n technitium
```

Check releases at https://github.com/TechnitiumSoftware/DnsServer/releases.

## Troubleshooting

### DNS not resolving

Check the pod is running and the LoadBalancer IP has been assigned:
```bash
kubectl get pods -n technitium
kubectl get svc technitium-dns-service -n technitium
```

Test DNS resolution directly against the service IP:
```bash
dig @192.168.20.199 google.com
```

### Web UI not reachable

Check the HTTPRoute has been accepted by the gateway:
```bash
kubectl get httproute technitium -n technitium
kubectl describe httproute technitium -n technitium
```

### Check logs
```bash
kubectl logs -n technitium deployment/technitium-dnsserver --tail=50
```

## Notes

- `imagePullPolicy: IfNotPresent` — image version is pinned so `Always` is unnecessary
- `automountServiceAccountToken: false` — Technitium does not need to call the Kubernetes API
- `enableServiceLinks: false` — prevents Kubernetes injecting service env vars into the pod
- `NET_BIND_SERVICE` capability is required to bind to privileged port 53
- TLS is handled by the wildcard cert (`*.petkus.id.lv`) in the `traefik` namespace — no per-app certificate needed
- `externalTrafficPolicy: Local` on the DNS service preserves real client IPs in query logs but means DNS only works through the node where the pod is scheduled — acceptable for a single-replica deployment