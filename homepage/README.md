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
  --from-literal=HOMEPAGE_VAR_MIKROTIK_CAPAX_PASSWORD=YOUR_VALUE \
  --from-literal=HOMEPAGE_VAR_PIHOLE_KEY=YOUR_VALUE \
  --from-literal=HOMEPAGE_VAR_NGINX_PASSWORD=YOUR_VALUE \
  --from-literal=HOMEPAGE_VAR_TECHNITIUM_KEY=YOUR_VALUE \
  --dry-run=client -o yaml | \
  kubeseal \
    --controller-name=sealed-secrets-controller \
    --controller-namespace=kube-system \
    --format yaml > sealedsecret.yaml
```

Use this full-recreation flow only for the **initial** deploy, or when you genuinely want to regenerate every key at once (see "Rotating every secret at once" below). For day-to-day changes — adding one key, rotating one password — use the single-key `--raw` approach below instead; it avoids re-encrypting keys that haven't changed and avoids ever having every plaintext secret in one file/command on your screen at once.

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

Each entry in `sealedsecret.yaml`'s `encryptedData` is sealed **independently** — sealing one key doesn't require or affect the ciphertext of any other key. This means you should almost never regenerate the whole file; instead, seal and patch in one key at a time.

### Adding a new secret, or rotating an existing one (recommended, day-to-day method)

This adds/updates exactly one key in `homepage-secrets` without touching any other value.

**1. Seal the single value**, scoped to the existing secret's name and namespace so the controller will accept it as part of that SealedSecret:

```bash
echo -n 'YOUR_VALUE' | kubeseal \
  --raw \
  --scope strict \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  --name homepage-secrets \
  --namespace homepage \
  --from-file=/dev/stdin
```

This prints a single encrypted blob to stdout.

**2. Edit `sealedsecret.yaml`** and add/replace only that one key under `encryptedData`:

```yaml
spec:
  encryptedData:
    HOMEPAGE_VAR_YOUR_KEY: <paste-blob-from-step-1>
    # ...all other keys unchanged...
```

**3. Apply just the SealedSecret:**

```bash
kubectl apply -f sealedsecret.yaml
```

The sealed-secrets controller decrypts in-cluster and merges the new key into the underlying `Secret` — other keys are untouched since their ciphertext didn't change.

**4. Verify it decrypted correctly:**

```bash
kubectl get secret homepage-secrets -n homepage -o jsonpath='{.data.HOMEPAGE_VAR_YOUR_KEY}' | base64 -d
```

**5. Reference it in `configmap.yaml`** (if new) as `"{{HOMEPAGE_VAR_YOUR_KEY}}"`, apply the ConfigMap, and restart:

```bash
kubectl apply -f configmap.yaml
kubectl rollout restart deployment/homepage -n homepage
```

**6. Clean up** any plaintext value left in shell history:

```bash
history -d $(history | grep 'echo -n' | tail -1 | awk '{print $1}')
```

> ⚠️ `--scope strict` (the default) binds the encrypted blob to the exact `--name`/`--namespace` pair. If these don't match `metadata.name`/`metadata.namespace` in `sealedsecret.yaml` exactly, the controller will fail to unseal that key.

### Removing a secret

Delete the corresponding line from `encryptedData` in `sealedsecret.yaml`, remove the `{{HOMEPAGE_VAR_*}}` reference from `configmap.yaml`, then:

```bash
kubectl apply -f sealedsecret.yaml
kubectl apply -f configmap.yaml
kubectl rollout restart deployment/homepage -n homepage
```

Note: applying a `SealedSecret` with a key removed does **not** automatically remove that key from the underlying `Secret` object in all sealed-secrets controller versions/configurations. If the stale key needs to be fully gone (not just unreferenced), delete and recreate the underlying `Secret` by deleting the `SealedSecret` first:

```bash
kubectl delete sealedsecret homepage-secrets -n homepage
kubectl apply -f sealedsecret.yaml
```

### Rotating every secret at once (rare — full recreation)

Only use this if you want to regenerate every key in one pass (e.g. periodic full credential rotation, or recovering from a leaked sealing key). This briefly requires having every plaintext value on hand at once — prefer the single-key method above whenever only one or two values need to change.

```bash
kubectl delete sealedsecret homepage-secrets -n homepage

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
  --from-literal=HOMEPAGE_VAR_MIKROTIK_CAPAX_PASSWORD=YOUR_VALUE \
  --from-literal=HOMEPAGE_VAR_PIHOLE_KEY=YOUR_VALUE \
  --from-literal=HOMEPAGE_VAR_NGINX_PASSWORD=YOUR_VALUE \
  --from-literal=HOMEPAGE_VAR_TECHNITIUM_KEY=YOUR_VALUE \
  --dry-run=client -o yaml | \
  kubeseal \
    --controller-name=sealed-secrets-controller \
    --controller-namespace=kube-system \
    --format yaml > sealedsecret.yaml

kubectl apply -f sealedsecret.yaml
kubectl rollout restart deployment/homepage -n homepage
```

## Widget credentials — least-privilege pattern

For widgets backed by a device/service that supports scoped users (e.g. Mikrotik RouterOS), don't reuse admin credentials for the homepage widget. Create a dedicated read-only account instead:

**On RouterOS** (example, adjust per firmware — verify valid policy flags first with `/user group print detail`):

```
/user group add name=homepage-readonly policy=read,api,rest-api,!local,!telnet,!ssh,!ftp,!reboot,!write,!policy,!test,!winbox,!password,!web,!sniff,!sensitive,!romon
/user add name=homepage-widget password=<strong-password> group=homepage-readonly
```

Use `homepage-widget` as the `username` in the relevant `services.yaml` entry, and seal `<strong-password>` as the corresponding `HOMEPAGE_VAR_*` key using the single-key method above.

Also restrict the relevant network service (e.g. `www` for the Mikrotik REST API used by the `mikrotik` widget type) to only the subnet homepage's pods/nodes run on:

```
/ip service set www address=<cluster-subnet>/24
```

Disable services that aren't in use rather than leaving them open (e.g. the binary RouterOS `api` service on port 8728 isn't needed by homepage's `mikrotik` widget, which uses the REST API over `www` instead).

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