# longhorn

Longhorn distributed block storage for k3s, installed via bare metal manifest.

## Files

| File | Description |
|------|-------------|
| `certificate.yaml` | TLS certificate via cert-manager |
| `ingress.yaml` | nginx ingress with TLS |

## Accessing

https://longhorn.petkus.id.lv

## Dependencies

- cert-manager with `cloudflare-clusterissuer` ClusterIssuer
- nginx ingress controller

## Installation

Longhorn is installed directly from the official manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.11.0/deploy/longhorn.yaml
```

After installation, apply the ingress manifests:

```bash
kubectl apply -f certificate.yaml
kubectl apply -f ingress.yaml
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

### Set frontend to single replica

After installation, scale the frontend deployment to a single replica to reduce resource usage on a homelab cluster:

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

It is strongly recommended to configure an off-cluster backup target (S3, NFS) in the Longhorn UI under **Settings → Backup Target** to protect against node failure.

## Notes

- Longhorn manifests are not stored in this repo — the official URL is the source of truth
- `proxy-body-size: 10000m` is set on the ingress to support backing image uploads
- No authentication is configured on the ingress — Longhorn UI is protected only by TLS. Consider adding basic auth if the cluster is exposed publicly