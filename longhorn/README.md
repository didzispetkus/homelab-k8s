# longhorn

Longhorn distributed block storage for k3s, installed via bare metal manifest.

## Files

| File | Description |
|------|-------------|
| `httproute.yaml` | Traefik Gateway API HTTPRoute for the Longhorn UI |

## Accessing

https://longhorn.petkus.id.lv

## Dependencies

- Traefik Gateway API controller in `traefik` namespace

## Installation

Longhorn is installed directly from the official manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.11.0/deploy/longhorn.yaml
```

After installation, apply the HTTPRoute:

```bash
kubectl apply -f httproute.yaml
```

Verify all Longhorn pods are running:

```bash
kubectl get pods -n longhorn-system
```

## Upgrading

Check the current version:

```bash
kubectl get deploy longhorn-manager -n longhorn-system \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Find the latest version at https://github.com/longhorn/longhorn/releases, then apply the new manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/vX.Y.Z/deploy/longhorn.yaml
```

Verify the rollout:

```bash
kubectl rollout status deployment/longhorn-manager -n longhorn-system
kubectl rollout status deployment/longhorn-driver-deployer -n longhorn-system
```

> ⚠️ Always check the [Longhorn upgrade notes](https://longhorn.io/docs/latest/deploy/upgrade/) before upgrading — some versions require specific upgrade paths and do not support skipping minor versions.

## Post-installation configuration

### Scale frontend to single replica

After installation, scale the UI deployment to one replica to reduce resource usage:

```bash
kubectl scale deployment longhorn-ui -n longhorn-system --replicas=1
```

To make this persistent across upgrades, patch the deployment:

```bash
kubectl patch deployment longhorn-ui -n longhorn-system \
  --type=json \
  -p='[{"op": "replace", "path": "/spec/replicas", "value": 1}]'
```

### Default storage class

Longhorn is set as the default storage class after installation. Verify:

```bash
kubectl get storageclass
```

## Backup configuration

### Backup target

Backups are stored on Unraid via NFS. The share `/mnt/user/backups/longhorn` is exported from the Unraid server at `192.168.20.3`.

Configure in **Longhorn UI → Settings → General**:

| Setting | Value |
|---------|-------|
| Backup Target | `nfs://192.168.20.3:/mnt/user/backups/longhorn` |
| Backup Target Credential Secret | (leave empty — NFS requires no credentials) |

> Make sure the NFS share is exported on Unraid under **Settings → NFS** and that `/mnt/user/backups/longhorn` is accessible. Verify from a cluster node:
> ```bash
> showmount -e 192.168.20.3
> ```

### Recurring jobs

Four recurring jobs are configured in **Longhorn UI → Recurring Jobs**:

| Name | Task | Cron | Retain | Description |
|------|------|------|--------|-------------|
| `daily-backup` | Backup | `0 2 * * *` | `7` | Daily at 2am, keep 7 days |
| `weekly-backup` | Backup | `0 3 * * 0` | `4` | Sunday at 3am, keep 4 weeks |
| `daily-snapshot-cleanup` | Snapshot Cleanup | `30 3 * * *` | — | Daily at 3:30am, removes orphaned snapshots |
| `weekly-trim` | Filesystem Trim | `0 4 * * 0` | — | Sunday at 4am, reclaims unused blocks |

### Jobs assigned per volume

| PVC | Namespace | Backup | Snapshot Cleanup | Trim |
|-----|-----------|--------|-----------------|------|
| `home-assistant-config` | `home-assistant` | ✅ | ✅ | ✅ |
| `lubelog-data` | `lubelog` | ✅ | ✅ | ✅ |
| `lubelog-postgres` | `lubelog` | ✅ | ✅ | ✅ |
| `technitium-dnsserver-pvc` | `technitium` | ✅ | ✅ | ✅ |
| `diun-data` | `diun` | ✅ | ✅ | ✅ |
| `zigbee2mqtt-data` | `zigbee2mqtt` | ❌ | ✅ | ✅ |
| `mosquitto-data` | `mosquitto` | ❌ | ✅ | ✅ |

To assign jobs to a volume: **Longhorn UI → Volumes** → click volume → **Edit Recurring Jobs**.

### Offsite backups (Duplicati)

Duplicati on Unraid uploads the NFS share to OneDrive daily at 4am — after Longhorn's 2am backup window. The full backup chain is:

```
Longhorn → NFS on Unraid (onsite) → Duplicati → OneDrive (offsite)
```

### Identifying volumes in the UI

Longhorn uses PVC UIDs as volume names by default. To identify volumes easily, enable the **PV/PVC** and **Namespace** columns in the Longhorn UI via the columns selector in the top right of the Volumes table.

### Snapshot maintenance

Over time Longhorn accumulates snapshot delta blocks which inflate the **Actual Size** of volumes. To reclaim space:

1. **Longhorn UI → Volumes** → click the volume
2. **Snapshots** → select old snapshots → **Delete**
3. Click **Trim** to reclaim space from deleted snapshots

Recurring backup jobs manage snapshot retention automatically going forward.

### Restoring from backup

1. **Longhorn UI → Backup** → find the backup → **Restore**
2. Give it a new volume name
3. Scale down the affected deployment:
   ```bash
   kubectl scale deployment <name> -n <namespace> --replicas=0
   ```
4. Swap the PVC to the restored volume, then scale back up:
   ```bash
   kubectl scale deployment <name> -n <namespace> --replicas=1
   ```

## Notes

- Longhorn manifests are not stored in this repo — the official URL is the source of truth
- TLS is handled by the wildcard cert (`*.petkus.id.lv`) in the `traefik` namespace — no per-app certificate needed
- No authentication is configured on the HTTPRoute — Longhorn UI is protected only by TLS. Consider adding basic auth if the cluster is exposed publicly