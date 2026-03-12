# lubelog

LubeLogger — vehicle maintenance and fuel economy tracker, backed by PostgreSQL.

## Files

| File | Description |
|------|-------------|
| `namespace.yaml` | Dedicated namespace |
| `serviceaccount.yaml` | Dedicated service accounts for lubelog and postgres |
| `db-sealedsecret.yaml` | Sealed Secret for database credentials |
| `pvc.yaml` | 2Gi Longhorn PVC for app data and DataProtection keys |
| `servicepostgres.yaml` | Headless service for PostgreSQL StatefulSet |
| `statefulset.yaml` | PostgreSQL StatefulSet with Longhorn volume |
| `deployment.yaml` | LubeLogger application deployment |
| `service.yaml` | ClusterIP service for LubeLogger |
| `httproute.yaml` | Traefik Gateway API HTTPRoute |

## Accessing

https://lubelog.petkus.id.lv

## Dependencies

- Traefik Gateway API controller in `traefik` namespace
- Longhorn storage
- Sealed Secrets controller in `kube-system`

## Deployment

### Step 1: Generate the database SealedSecret

```bash
kubectl create secret generic lubelog-db-secret \
  --namespace lubelog \
  --from-literal=POSTGRES_CONNECTION="Host=lubelog-postgres;Port=5432;Username=lubelogger;Password=YOUR_PASSWORD;Database=lubelogger" \
  --from-literal=POSTGRES_USER="lubelogger" \
  --from-literal=POSTGRES_PASSWORD="YOUR_PASSWORD" \
  --from-literal=POSTGRES_DB="lubelogger" \
  --dry-run=client -o yaml | \
  kubeseal \
    --controller-name=sealed-secrets-controller \
    --controller-namespace=kube-system \
    --format yaml > db-sealedsecret.yaml
```

### Step 2: Apply all manifests

```bash
kubectl apply -f .
```

### Step 3: Verify

```bash
# Wait for postgres to be ready first
kubectl rollout status statefulset/lubelog-postgres -n lubelog

# Then check lubelog
kubectl rollout status deployment/lubelog -n lubelog
kubectl get pods -n lubelog
```

## Upgrading

### LubeLogger

Update the image version in `deployment.yaml` and the version label in all files:

```yaml
image: ghcr.io/hargata/lubelogger:vNEW_VERSION
app.kubernetes.io/version: "NEW_VERSION"
```

Check releases at https://github.com/hargata/lubelog/releases.

### PostgreSQL

> ⚠️ Always back up the database before upgrading PostgreSQL. Major version upgrades (e.g. 18 → 19) require a dump/restore — they cannot be done by changing the image tag.

Backup before upgrading:
```bash
kubectl exec -n lubelog lubelog-postgres-0 -- \
  pg_dump -U lubelogger lubelogger > lubelog-backup.sql
```

Then update the image in `statefulset.yaml`:
```yaml
image: postgres:NEW_VERSION
```

## Database backup

```bash
# Manual backup
kubectl exec -n lubelog lubelog-postgres-0 -- \
  pg_dump -U lubelogger lubelogger > lubelog-backup-$(date +%Y%m%d).sql

# Restore from backup
kubectl exec -i -n lubelog lubelog-postgres-0 -- \
  psql -U lubelogger lubelogger < lubelog-backup.sql
```

## PVC volume layout

The `lubelog-pvc` is shared between two mount points using `subPath`:

| Mount path | subPath | Contents |
|------------|---------|----------|
| `/App/data` | `data/` | images, documents, config, translations |
| `/root/.aspnet/DataProtection-Keys` | `keys/` | ASP.NET DataProtection key files |

## Notes

- `strategy: Recreate` is set on the lubelog deployment since the PVC is ReadWriteOnce
- Both `/App/data` and `/root/.aspnet/DataProtection-Keys` share `lubelog-pvc` using `subPath` — `data/` and `keys/` subdirectories respectively
- TLS is handled by the wildcard cert (`*.petkus.id.lv`) in the `traefik` namespace — no per-app certificate needed
- PostgreSQL image is pinned to `18.1` — do not downgrade
- `fsGroup: 999` and `runAsUser: 999` match the postgres user inside the official postgres image
- `automountServiceAccountToken: false` is set on both service accounts — neither workload needs to call the Kubernetes API
- `initialDelaySeconds` is not used on readiness/liveness probes — `startupProbe` already gates them until the container is ready, making the delay redundant
- `Cookie is invalid or does not exist` warnings in logs after a restart are normal — they clear once you log in again