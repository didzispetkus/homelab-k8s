# Gateway API Migration

Migration from ingress-nginx to Kubernetes Gateway API using Traefik as the
Gateway controller.

## Why

ingress-nginx is being retired in March 2026 with no further security patches.
Gateway API is the official Kubernetes successor to the Ingress API.

## Architecture

```
Browser
  └─► Traefik LoadBalancer (192.168.20.110)
        └─► Gateway (main-gateway, traefik namespace)
              └─► HTTPRoute (per app namespace)
                    └─► App Service
```

TLS is terminated at Traefik using a wildcard cert (`*.petkus.id.lv`) issued
by cert-manager via Cloudflare DNS-01.

---

## Directory Structure

```
gateway-api/
├── README.md
└── traefik/
    ├── values.yaml        # Helm values for Traefik install
    └── certificate.yaml   # Wildcard TLS cert issued into traefik namespace

cert-manager/
└── gateway-api-patch.yaml # Adds ExperimentalGatewayAPISupport feature gate

home-assistant/
├── middleware.yaml        # Traefik buffering middleware for 50Mi upload limit
└── httproute.yaml

homepage/
└── httproute.yaml

longhorn/
└── httproute.yaml

lubelog/
└── httproute.yaml

technitium/
└── httproute.yaml

zigbee2mqtt/
└── httproute.yaml
```

Each app's `httproute.yaml` lives alongside its other manifests in its own folder.
Per-app `certificate.yaml` files have been removed — TLS is handled centrally by
the wildcard cert in the `traefik` namespace.

---

## Prerequisites

- k3s cluster with MetalLB configured (pool: `192.168.20.101–192.168.20.200`)
- cert-manager v1.20.0 with `cloudflare-clusterissuer` ClusterIssuer working
- Helm 3 installed locally
- IP `192.168.20.110` free in MetalLB pool — verify with:
  ```bash
  kubectl get svc -A | grep LoadBalancer
  ```

---

## Step 1: Install Traefik via Helm

Traefik's Helm chart bundles and manages Gateway API CRDs automatically —
no need to install them manually.

Add the Traefik Helm repository:
```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

Install Traefik into its own namespace:
```bash
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --values gateway-api/traefik/values.yaml
```

Verify Traefik is running cleanly:
```bash
kubectl get pods -n traefik
kubectl logs -n traefik deployment/traefik --tail=20
```

Verify Traefik got its LoadBalancer IP from MetalLB:
```bash
kubectl get svc -n traefik
```

Expected:
```
NAME      TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)
traefik   LoadBalancer   10.43.x.x    192.168.20.110   80:xxx/TCP,443:xxx/TCP
```

Verify GatewayClass was created and accepted:
```bash
kubectl get gatewayclass
```

Expected:
```
NAME      CONTROLLER                      ACCEPTED
traefik   traefik.io/gateway-controller   True
```

Verify the Gateway was created:
```bash
kubectl get gateway -n traefik
```

Expected:
```
NAME           CLASS     ADDRESS          PROGRAMMED
main-gateway   traefik   192.168.20.110   True
```

Check which Gateway API version Traefik installed:
```bash
kubectl get crd gateways.gateway.networking.k8s.io \
  -o jsonpath='{.metadata.annotations.gateway\.networking\.k8s\.io/bundle-version}'
```

---

## Step 2: Enable cert-manager Gateway API support

cert-manager requires a feature gate to issue TLS certificates for Gateway
resources. The flag is stored in a patch file tracked in git so it survives
cert-manager upgrades and reinstalls.

Verify the current args on your running cert-manager match the patch file:
```bash
kubectl get deployment cert-manager -n cert-manager \
  -o jsonpath='{.spec.template.spec.containers[0].args}' | tr ',' '\n'
```

Expected output (must match `cert-manager/gateway-api-patch.yaml`):
```
["--v=2"
"--cluster-resource-namespace=$(POD_NAMESPACE)"
"--leader-election-namespace=kube-system"
"--acme-http01-solver-image=quay.io/jetstack/cert-manager-acmesolver:v1.20.0"
"--max-concurrent-challenges=60"]
```

Apply the patch using `kubectl patch --patch-file` (not `kubectl apply`):
```bash
kubectl patch deployment cert-manager \
  -n cert-manager \
  --patch-file cert-manager/gateway-api-patch.yaml
```

Verify the flag was added:
```bash
kubectl get deployment cert-manager -n cert-manager \
  -o jsonpath='{.spec.template.spec.containers[0].args}' | tr ',' '\n' | grep feature
```

Expected:
```
"--feature-gates=ExperimentalGatewayAPISupport=true"
```

Wait for cert-manager to restart:
```bash
kubectl rollout status deployment/cert-manager -n cert-manager
```

> **Important:** After any future cert-manager upgrade or reinstall, re-apply
> this patch — upgrades overwrite the Deployment and will drop the feature gate:
> ```bash
> kubectl patch deployment cert-manager \
>   -n cert-manager \
>   --patch-file cert-manager/gateway-api-patch.yaml
> ```

---

## Step 3: Issue wildcard TLS certificate

Apply the Certificate resource so cert-manager issues a wildcard cert
into the traefik namespace (where the Gateway lives):

```bash
kubectl apply -f gateway-api/traefik/certificate.yaml
```

Watch cert-manager issue the certificate:
```bash
kubectl get certificate -n traefik -w
```

Wait until `READY` is `True`:
```
NAME                   READY   SECRET                 AGE
traefik-wildcard-tls   True    traefik-wildcard-tls   2m
```

This takes 1–2 minutes while cert-manager completes the Cloudflare DNS-01
challenge. If it takes longer, check:
```bash
kubectl describe certificate traefik-wildcard-tls -n traefik
kubectl describe challenge -n traefik
```

---

## Step 4: Migrate Homepage

Apply the HTTPRoute from the homepage folder:
```bash
kubectl apply -f homepage/httproute.yaml
```

Verify the service name matches what is in the cluster:
```bash
kubectl get svc -n homepage
```

Update `homepage/httproute.yaml` `backendRefs.name` if the service name differs.

Verify the route is accepted and attached to the Gateway:
```bash
kubectl get httproute -n homepage
kubectl describe httproute homepage -n homepage
```

Look for `Accepted: True` and `ResolvedRefs: True` in the conditions.

Update the Technitium DNS `A` record for `home.petkus.id.lv` from the
ingress-nginx IP to the Traefik IP `192.168.20.110`.

Test in browser: `https://home.petkus.id.lv`

Once confirmed working, delete the old Ingress resource and remove from git:
```bash
kubectl delete ingress homepage -n homepage
git rm homepage/ingress.yaml
git commit -m "chore(homepage): remove ingress-nginx ingress, migrated to Gateway API HTTPRoute"
git push
```

---

## Migrating Remaining Apps

Repeat Step 4 for each remaining app:

1. Create `<app>/httproute.yaml` alongside the app's other manifests
2. `kubectl apply -f <app>/httproute.yaml`
3. Verify `ResolvedRefs: True` and `Accepted: True`
4. Update Technitium DNS record to `192.168.20.110`
5. Test in browser
6. `kubectl delete ingress <name> -n <namespace>`
7. `git rm <app>/ingress.yaml && git commit && git push`

---

## Migration Status

| App | httproute.yaml | DNS Updated | Ingress Deleted |
|-----|---------------|-------------|-----------------|
| Homepage (`home.petkus.id.lv`) | ✅ | ✅ | ✅ |
| Home Assistant (`haas.petkus.id.lv`) | ✅ | ✅ | ✅ |
| Longhorn (`longhorn.petkus.id.lv`) | ✅ | ✅ | ✅ |
| Zigbee2MQTT (`zigbee2mqtt.petkus.id.lv`) | ✅ | ✅ | ✅ |
| Technitium (`technitium.petkus.id.lv`) | ✅ | ✅ | ✅ |
| LubeLogger (`lubelog.petkus.id.lv`) | ✅ | ✅ | ✅ |

✅ Migration complete. ingress-nginx has been removed from the cluster.

---

## Upgrading Traefik

Traefik's Helm chart manages all CRDs including Gateway API — upgrading the
chart upgrades the CRDs automatically:

```bash
helm repo update
helm upgrade traefik traefik/traefik \
  --namespace traefik \
  --values gateway-api/traefik/values.yaml
```

---

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
Look for `ResolvedRefs` and `Accepted` conditions. Common causes:
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