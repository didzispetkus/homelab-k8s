# cert-manager

Manages TLS certificate issuance for the cluster using [cert-manager](https://cert-manager.io/) with Cloudflare DNS-01 challenge via Let's Encrypt.

## Files

| File | Description |
|------|-------------|
| `cloudflare-api-token-sealedsecret.yaml` | Sealed Secret containing the Cloudflare API token |
| `cloudflare-clusterissuer.yaml` | ClusterIssuer configured for Let's Encrypt + Cloudflare DNS-01 |

## How it works

```
Let's Encrypt (ACME)
        ↑
ClusterIssuer (cloudflare-clusterissuer)
        ↓
Cloudflare DNS-01 challenge
        ↓
cloudflare-api-token-secret  ←  SealedSecret (decrypted by sealed-secrets-controller)
```

1. cert-manager requests a certificate from Let's Encrypt
2. Let's Encrypt issues a DNS-01 challenge
3. cert-manager uses the Cloudflare API token to create a DNS TXT record to satisfy the challenge
4. Let's Encrypt verifies the record and issues the certificate

## Secrets management

Secrets are managed with [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets). The `SealedSecret` manifest is safe to commit — it is encrypted with the cluster's public key and can only be decrypted by the `sealed-secrets-controller` running in the cluster.

The controller automatically creates the real `cloudflare-api-token-secret` Kubernetes Secret at runtime.

### Re-sealing the secret (e.g. after token rotation)

```bash
kubectl create secret generic cloudflare-api-token-secret \
  --namespace cert-manager \
  --from-literal=api-token=NEW_TOKEN_HERE \
  --dry-run=client -o yaml | \
  kubeseal \
    --controller-name=sealed-secrets-controller \
    --controller-namespace=kube-system \
    --format yaml > cloudflare-api-token-sealedsecret.yaml
```

## Dependencies

- [cert-manager](https://cert-manager.io/) installed in the cluster
- [sealed-secrets-controller](https://github.com/bitnami-labs/sealed-secrets) running in `kube-system`
- A Cloudflare API token with `Zone:DNS:Edit` permissions for the target domain

## Upgrading cert-manager

Check the currently installed version:
```bash
kubectl get pods -n cert-manager -o jsonpath='{.items[0].spec.containers[0].image}'
```

Find the latest available version:
```bash
curl -s https://api.github.com/repos/cert-manager/cert-manager/releases/latest | grep tag_name
```

Before upgrading, review the release notes for any breaking changes:
- https://cert-manager.io/docs/releases/

Apply the new version:
```bash
# Replace vX.Y.Z with the target version
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/vX.Y.Z/cert-manager.yaml
```

Verify the rollout completed:
```bash
kubectl rollout status deployment/cert-manager -n cert-manager
kubectl rollout status deployment/cert-manager-webhook -n cert-manager
kubectl rollout status deployment/cert-manager-cainjector -n cert-manager
```

Confirm the new version is running:
```bash
kubectl get pods -n cert-manager -o jsonpath='{.items[0].spec.containers[0].image}'
```

> ⚠️ When jumping multiple minor versions, upgrade one minor version at a time and check release notes for each step.

## ClusterIssuer

| Field | Value |
|-------|-------|
| Name | `cloudflare-clusterissuer` |
| ACME Server | Let's Encrypt Production |
| Challenge Type | DNS-01 via Cloudflare |
| Account Key Secret | `cloudflare-clusterissuer-account-key` |

To use this issuer in a `Certificate` or `Ingress`, reference it as:

```yaml
annotations:
  cert-manager.io/cluster-issuer: cloudflare-clusterissuer
```