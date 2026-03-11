# technitium

Technitium DNS Server — a self-hosted DNS server with blocking, caching, and a web UI.

## Files

| File | Description |
|------|-------------|
| `namespace.yaml` | Dedicated namespace |
| `certificate.yaml` | TLS certificate via cert-manager |
| `serviceaccount.yaml` | Dedicated service account |
| `pvc.yaml` | 1Gi Longhorn persistent volume for DNS data |
| `service-dns.yaml` | LoadBalancer service for DNS traffic (TCP/UDP port 53) |
| `service-web.yaml` | ClusterIP service for web UI (port 5380) |
| `deployment.yaml` | Technitium DNS server deployment |
| `ingress.yaml` | nginx ingress with TLS for web UI |

## Accessing

https://technitium.petkus.id.lv

## Dependencies

- cert-manager with `cloudflare-clusterissuer` ClusterIssuer
- nginx ingress controller
- Longhorn storage
- MetalLB for LoadBalancer IP assignment

## Deployment

```bash
kubectl apply -f namespace.yaml
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
| Web UI | ClusterIP | — | 5380 | TCP |
| Ingress | — | — | 443 | HTTPS |

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

## Notes

- `imagePullPolicy: IfNotPresent` is used since the image version is pinned — `Always` is unnecessary and wastes bandwidth
- `automountServiceAccountToken: false` — Technitium does not need to call the Kubernetes API
- `enableServiceLinks: false` — prevents unnecessary service environment variables being injected into the pod
- `NET_BIND_SERVICE` capability is required to bind to port 53
- Homepage is configured directly in homepage rather than via auto-discovery annotations to avoid leaking API keys in the ingress manifest