# Gateway API / Traefik

Traefik is the Gateway API controller for the cluster. All HTTPS traffic is routed through a single Traefik LoadBalancer and dispatched to apps via per-namespace HTTPRoutes.

## Architecture

```
Browser
  └─► Traefik LoadBalancer (192.168.20.110)
        └─► Gateway (main-gateway, traefik namespace)
              └─► HTTPRoute (per app namespace)
                    └─► App Service
```

TLS is terminated at Traefik using a wildcard cert (`*.petkus.id.lv`) issued by cert-manager via Cloudflare DNS-01. All apps share this cert — no per-app certificates are needed.

## Files

| File | Description |
|------|-------------|
| `traefik/values.yaml` | Helm values for Traefik install |
| `traefik/certificate.yaml` | Wildcard TLS cert issued into `traefik` namespace |

Each app's `httproute.yaml` lives alongside its other manifests in its own folder.

## Dependencies

- k3s cluster with MetalLB configured (pool: `192.168.20.101–192.168.20.200`)
- cert-manager v1.20.0 with `cloudflare-clusterissuer` ClusterIssuer working
- Helm 3 installed locally
- IP `192.168.20.110` reserved in MetalLB pool

## Installing Traefik

Add the Helm repository:
```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

Install Traefik — the chart bundles and manages Gateway API CRDs automatically:
```bash
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --values gateway-api/traefik/values.yaml
```

Apply the wildcard certificate:
```bash
kubectl apply -f gateway-api/traefik/certificate.yaml
```

Watch cert-manager issue it (takes 1–2 minutes for the Cloudflare DNS-01 challenge):
```bash
kubectl get certificate -n traefik -w
```

Expected:
```
NAME                   READY   SECRET                 AGE
traefik-wildcard-tls   True    traefik-wildcard-tls   2m
```

## cert-manager Gateway API feature gate

cert-manager requires a feature gate to issue certificates for Gateway resources. This must be applied after every cert-manager upgrade as upgrades overwrite the Deployment:

```bash
kubectl patch deployment cert-manager \
  -n cert-manager \
  --patch-file cert-manager/gateway-api-patch.yaml
```

Verify the flag is present:
```bash
kubectl get deployment cert-manager -n cert-manager \
  -o jsonpath='{.spec.template.spec.containers[0].args}' | tr ',' '\n' | grep feature
```

Expected:
```
"--feature-gates=ExperimentalGatewayAPISupport=true"
```

## Upgrading Traefik

```bash
helm repo update
helm upgrade traefik traefik/traefik \
  --namespace traefik \
  --values gateway-api/traefik/values.yaml
```

After upgrading, re-apply the cert-manager feature gate patch (see above).

## Accessing the Traefik dashboard

The dashboard runs on port 9000 inside the pod but is not exposed externally. Access it via port-forward from WSL:

```bash
kubectl port-forward -n traefik deployment/traefik 9000:9000 --address 0.0.0.0
```

Then open in your Windows browser: http://localhost:9000/dashboard/

> The trailing slash is required.

## Verifying the setup

```bash
# Traefik pod is running
kubectl get pods -n traefik

# LoadBalancer IP assigned
kubectl get svc -n traefik

# GatewayClass accepted
kubectl get gatewayclass

# Gateway programmed
kubectl get gateway -n traefik

# All HTTPRoutes accepted
kubectl get httproute -A
```

Expected GatewayClass:
```
NAME      CONTROLLER                      ACCEPTED
traefik   traefik.io/gateway-controller   True
```

Expected Gateway:
```
NAME           CLASS     ADDRESS          PROGRAMMED
main-gateway   traefik   192.168.20.110   True
```

## Adding a new app

1. Create `httproute.yaml` in the app's directory using the standard pattern:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: <app>
  namespace: <namespace>
spec:
  parentRefs:
    - name: main-gateway
      namespace: traefik
      sectionName: websecure
  hostnames:
    - "<hostname>.petkus.id.lv"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: <service-name>
          port: <port>
```

2. Apply it and verify:
```bash
kubectl apply -f <app>/httproute.yaml
kubectl describe httproute <name> -n <namespace>
```

Look for `Accepted: True` and `ResolvedRefs: True` in the conditions.

3. Add a DNS `A` record in Technitium pointing `<hostname>.petkus.id.lv` → `192.168.20.110`.

## Troubleshooting

**GatewayClass not accepted:**
```bash
kubectl describe gatewayclass traefik
```
Traefik pod must be running and healthy first.

**HTTPRoute not attached:**
```bash
kubectl describe httproute <name> -n <namespace>
```
Common causes:
- Gateway name or namespace wrong in `parentRefs`
- `sectionName` does not match listener name in `values.yaml` (`websecure`)
- Service name or port wrong in `backendRefs` — verify with `kubectl get svc -n <namespace>`

**Certificate not issuing:**
```bash
kubectl describe certificate traefik-wildcard-tls -n traefik
kubectl describe certificaterequest -n traefik
kubectl describe challenge -n traefik
```

**Traefik not routing traffic:**
```bash
kubectl logs -n traefik deployment/traefik -f
```

**Check Gateway status:**
```bash
kubectl describe gateway main-gateway -n traefik
```