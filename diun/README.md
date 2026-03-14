# diun

Docker Image Update Notifier — watches all running container images cluster-wide and sends Telegram notifications when new versions are available.

## Files

| File | Description |
|------|-------------|
| `namespace.yaml` | Dedicated namespace |
| `serviceaccount.yaml` | Dedicated service account |
| `clusterrole.yaml` | ClusterRole and ClusterRoleBinding for pod/node read access |
| `pvc.yaml` | 1Gi Longhorn persistent volume for DIUN database |
| `sealedsecret.yaml` | Sealed Telegram bot token and chat ID |
| `deployment.yaml` | DIUN deployment |

## Accessing

DIUN has no web UI — notifications are delivered via Telegram. Interact via `kubectl exec` if needed.

## Dependencies

- Longhorn storage
- Sealed Secrets controller

## Creating the SealedSecret

```bash
kubectl create secret generic diun-secret \
  --namespace diun \
  --from-literal=DIUN_NOTIF_TELEGRAM_TOKEN=your-bot-token \
  --from-literal=DIUN_NOTIF_TELEGRAM_CHATIDS=your-chat-id \
  --dry-run=client -o yaml | \
  kubeseal \
    --controller-name=sealed-secrets-controller \
    --controller-namespace=kube-system \
    --format yaml > sealedsecret.yaml
```

## Deployment

```bash
kubectl apply -f namespace.yaml
kubectl apply -f serviceaccount.yaml
kubectl apply -f clusterrole.yaml
kubectl apply -f sealedsecret.yaml
kubectl apply -f pvc.yaml
kubectl apply -f deployment.yaml
```

Verify rollout:
```bash
kubectl rollout status deployment/diun -n diun
kubectl logs -n diun deployment/diun --tail=30
```

## Testing

Send a test Telegram notification:
```bash
kubectl exec -n diun deployment/diun -- diun notif test
```

List all images DIUN is currently tracking:
```bash
kubectl exec -n diun deployment/diun -- diun image list
```

## Upgrading

Update the image version in `deployment.yaml` and the version label in all files:

```yaml
image: crazymax/diun:NEW_VERSION
app.kubernetes.io/version: "NEW_VERSION"
```

Then apply:
```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/diun -n diun
```

Check releases at https://github.com/crazy-max/diun/releases.

## Configuration

All configuration is via environment variables in `deployment.yaml`. Key settings:

| Variable | Value | Description |
|----------|-------|-------------|
| `DIUN_WATCH_SCHEDULE` | `0 19 * * *` | Daily scan at 7pm |
| `DIUN_WATCH_WORKERS` | `20` | Parallel image check workers |
| `DIUN_WATCH_JITTER` | `30s` | Random delay between checks to avoid rate limiting |
| `DIUN_PROVIDERS_KUBERNETES_WATCHBYDEFAULT` | `true` | Watch all pods cluster-wide without needing per-pod annotations |

## Notes

- `automountServiceAccountToken: true` is required — DIUN needs it to query the Kubernetes API and discover running pods
- `readOnlyRootFilesystem: true` is safe since the database is stored on the mounted PVC at `/data`
- DIUN watches all namespaces by default due to `watchByDefault: true` — no per-pod annotations needed