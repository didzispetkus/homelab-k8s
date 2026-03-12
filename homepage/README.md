# homepage

[gethomepage/homepage](https://github.com/gethomepage/homepage) — a highly customisable application dashboard with Kubernetes service discovery.

## Files

| File | Description |
|------|-------------|
| `namespace.yaml` | Dedicated namespace |
| `serviceaccount.yaml` | Service account for Kubernetes discovery |
| `secret.yaml` | Long-lived service account token (required for cluster discovery) |
| `clusterrole.yaml` | ClusterRole and ClusterRoleBinding for k8s API access |
| `sealedsecret.yaml` | Sealed Secret for all widget API keys and passwords |
| `configmap.yaml` | Homepage configuration with `{{HOMEPAGE_VAR_*}}` placeholders |
| `deployment.yaml` | Homepage deployment |
| `service.yaml` | ClusterIP service on port 3000 |
| `httproute.yaml` | Traefik Gateway API HTTPRoute |

## Accessing

https://home.petkus.id.lv

## Dependencies

- Traefik Gateway API controller in `traefik` namespace
- Sealed Secrets controller in `kube-system`

## Deployment

### Step 1: Generate the SealedSecret

```bash
kubectl create secret generic homepage-secrets \
  --namespace homepage \
  --from-literal=HOMEPAGE_VAR_JELLYFIN_KEY=YOUR_VALUE \
  --from-literal=HOMEPAGE_VAR_SONARR_KEY=YOUR_VALUE \
  --from-literal=HOMEPAGE_VAR_RADARR_KEY=YOUR_VALUE \
  --from-literal=HOMEPAGE_VAR_BAZARR_KEY=YOUR_VALUE \
  --from-literal=HOMEPAGE_VAR_PROWLARR_KEY=YOUR_VALUE \
  --from-literal=HOMEPAGE_VAR_LIDARR_KEY=YOUR_VALUE \
  --from-literal=HOMEPAGE_VAR_QBITTORRENT_PASSWORD=YOUR_VALUE \
  --from-literal=HOMEPAGE_VAR_GLUETUN_KEY=YOUR_VALUE \
  --from-literal=HOMEPAGE_VAR_JELLYSEERR_KEY=YOUR_VALUE \
  --from-literal=HOMEPAGE_VAR_IMMICH_KEY=YOUR_VALUE \
  --from-literal=HOMEPAGE_VAR_MIKROTIK_RB5009_PASSWORD=YOUR_VALUE \
  --from-literal=HOMEPAGE_VAR_MIKROTIK_HAPAC2_PASSWORD=YOUR_VALUE \
  --from-literal=HOMEPAGE_VAR_PIHOLE_KEY=YOUR_VALUE \
  --from-literal=HOMEPAGE_VAR_NGINX_PASSWORD=YOUR_VALUE \
  --dry-run=client -o yaml | \
  kubeseal \
    --controller-name=sealed-secrets-controller \
    --controller-namespace=kube-system \
    --format yaml > sealedsecret.yaml
```

### Step 2: Apply all manifests

```bash
kubectl apply -f .
```

### Step 3: Verify

```bash
kubectl rollout status deployment/homepage -n homepage
kubectl logs -n homepage deployment/homepage -f
```

## Secrets management

All widget API keys and passwords are stored in a `SealedSecret` as `HOMEPAGE_VAR_*` environment variables. The `configmap.yaml` references them using homepage's variable substitution syntax:

```yaml
key: "{{HOMEPAGE_VAR_JELLYFIN_KEY}}"
```

### Adding a new secret

Add a new `--from-literal` to the kubeseal command and re-seal. Then reference it in `configmap.yaml` as `"{{HOMEPAGE_VAR_YOUR_KEY}}"`.

### Rotating a secret

```bash
kubectl delete sealedsecret homepage-secrets -n homepage

kubectl create secret generic homepage-secrets \
  --namespace homepage \
  ... \
  --dry-run=client -o yaml | \
  kubeseal \
    --controller-name=sealed-secrets-controller \
    --controller-namespace=kube-system \
    --format yaml > sealedsecret.yaml

kubectl apply -f sealedsecret.yaml
kubectl rollout restart deployment/homepage -n homepage
```

## Upgrading

Update the image version in `deployment.yaml`:

```yaml
image: ghcr.io/gethomepage/homepage:vNEW_VERSION
app.kubernetes.io/version: "vNEW_VERSION"
```

Check releases at https://github.com/gethomepage/homepage/releases.

## Notes

- `automountServiceAccountToken: true` is intentionally set on the deployment — homepage requires it for Kubernetes pod/node/HTTPRoute discovery
- `enableServiceLinks: false` — prevents Kubernetes injecting service env vars into the pod, which could collide with `HOMEPAGE_VAR_*` vars
- TLS is handled by the wildcard cert (`*.petkus.id.lv`) in the `traefik` namespace — no per-app certificate needed
- Widget secrets in `configmap.yaml` use `{{HOMEPAGE_VAR_*}}` placeholders — homepage substitutes these at runtime from environment variables
- `secret.yaml` is a `kubernetes.io/service-account-token` type — it contains no sensitive data at commit time; Kubernetes populates the actual token after apply