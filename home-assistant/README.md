# home-assistant

Home Assistant deployment on k3s with TLS via cert-manager and Cloudflare DNS-01.

## Files

| File | Description |
|------|-------------|
| `namespace.yaml` | Dedicated namespace |
| `certificate.yaml` | TLS certificate via cert-manager |
| `serviceaccount.yaml` | Dedicated service account |
| `pvc.yaml` | 5Gi Longhorn persistent volume for HA config |
| `service.yaml` | ClusterIP service on port 8123 |
| `deployment.yaml` | Home Assistant deployment |
| `ingress.yaml` | nginx ingress with TLS |

## Accessing

https://haas.petkus.id.lv

## Dependencies

- cert-manager with `cloudflare-clusterissuer` ClusterIssuer
- nginx ingress controller
- Longhorn storage

## Deployment

```bash
kubectl apply -f namespace.yaml
kubectl apply -f certificate.yaml
kubectl apply -f serviceaccount.yaml
kubectl apply -f pvc.yaml
kubectl apply -f service.yaml
kubectl apply -f deployment.yaml
kubectl apply -f ingress.yaml
```

Or apply the entire directory at once:
```bash
kubectl apply -f .
```

Verify rollout:
```bash
kubectl rollout status deployment/home-assistant -n home-assistant
kubectl get pods -n home-assistant
```

## Upgrading

Update the image version in `deployment.yaml`:
```yaml
image: "ghcr.io/home-assistant/home-assistant:NEW_VERSION"
```

Also update the version label in all files:
```yaml
app.kubernetes.io/version: "NEW_VERSION"
```

Then apply:
```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/home-assistant -n home-assistant
```

## Reverse proxy configuration

HA must be configured to trust the nginx ingress controller. Add the following to `/config/configuration.yaml` inside the pod:

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 192.168.20.0/24   # cluster network range
    - 10.42.0.0/16      # k3s pod CIDR
    - 10.43.0.0/16      # k3s service CIDR
```

Edit via exec:
```bash
kubectl exec -it -n home-assistant deployment/home-assistant -- vi /config/configuration.yaml
```

Then restart:
```bash
kubectl rollout restart deployment/home-assistant -n home-assistant
```

## Backup and restore

Backups are managed via **Settings → System → Backups** in the HA UI.

To restore a backup, go to **Settings → System → Backups**, upload the `.tar` file and select **Full restore**.

> The nginx ingress is configured with a 50m upload limit to support large backup files.

## Notes

- `hostNetwork: true` is required for device discovery (mDNS, UPnP, Zigbee)
- `strategy: Recreate` is required since the PVC is ReadWriteOnce
- `replicas: 1` — HA does not support multi-replica deployments
- Probes use `tcpSocket` instead of `httpGet` because the `/api/` endpoint requires authentication — HTTP probes would trigger ban warnings in HA logs
- `securityContext` with `allowPrivilegeEscalation` and `capabilities` must be set at the **container level**, not the pod level