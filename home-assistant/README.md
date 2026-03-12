# home-assistant

Home Assistant deployment on k3s.

## Files

| File | Description |
|------|-------------|
| `namespace.yaml` | Dedicated namespace |
| `serviceaccount.yaml` | Dedicated service account |
| `pvc.yaml` | 5Gi Longhorn persistent volume for HA config |
| `service.yaml` | ClusterIP service on port 8123 |
| `deployment.yaml` | Home Assistant deployment |
| `middleware.yaml` | Traefik buffering middleware (50Mi request body for backup uploads) |
| `httproute.yaml` | Traefik Gateway API HTTPRoute |

## Accessing

https://haas.petkus.id.lv

## Dependencies

- Traefik Gateway API controller in `traefik` namespace
- Longhorn storage

## Deployment

```bash
kubectl apply -f .
```

Verify rollout:
```bash
kubectl rollout status deployment/home-assistant -n home-assistant
kubectl get pods -n home-assistant
```

## Upgrading

Update the image version in `deployment.yaml` and the version label in all files:

```yaml
image: "ghcr.io/home-assistant/home-assistant:NEW_VERSION"
app.kubernetes.io/version: "NEW_VERSION"
```

Then apply:
```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/home-assistant -n home-assistant
```

Check releases at https://github.com/home-assistant/core/releases.

## Reverse proxy configuration

HA must be configured to trust the Traefik ingress. Add the following to `/config/configuration.yaml` inside the pod:

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

> The Traefik buffering middleware (`middleware.yaml`) is configured with a 50Mi request body limit to support large backup file uploads.

## Notes

- `hostNetwork: true` is required for device discovery (mDNS, UPnP, Zigbee)
- `dnsPolicy: ClusterFirstWithHostNet` is required when using `hostNetwork: true` to retain cluster DNS resolution
- `strategy: Recreate` is required since the PVC is ReadWriteOnce
- `replicas: 1` — HA does not support multi-replica deployments
- `NET_ADMIN` and `NET_RAW` capabilities are required for network device discovery
- TLS is handled by the wildcard cert (`*.petkus.id.lv`) in the `traefik` namespace — no per-app certificate needed
- Probes use `tcpSocket` instead of `httpGet` because the `/api/` endpoint requires authentication — HTTP probes would trigger ban warnings in HA logs
- `securityContext` with `allowPrivilegeEscalation` and `capabilities` must be set at the **container level**, not the pod level