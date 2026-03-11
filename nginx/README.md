# ingress-nginx

nginx ingress controller for k3s, installed via bare metal manifest. Exposes HTTP/HTTPS traffic to cluster services via MetalLB LoadBalancer.

## Files

| File | Description |
|------|-------------|
| `loadbalancer-service.yaml` | LoadBalancer service exposing ports 80 and 443 via MetalLB |

## Installation

ingress-nginx is installed directly from the official bare metal manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.14.3/deploy/static/provider/baremetal/deploy.yaml
```

Wait for the controller to be ready:

```bash
kubectl rollout status deployment/ingress-nginx-controller -n ingress-nginx
```

Then apply the LoadBalancer service:

```bash
kubectl apply -f loadbalancer-service.yaml
```

Verify MetalLB has assigned an external IP:

```bash
kubectl get service ingress-nginx-controller-loadbalancer -n ingress-nginx
```

## Upgrading

Check the current version:

```bash
kubectl get deploy ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Find the latest version at https://github.com/kubernetes/ingress-nginx/releases, then apply the new manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-vX.Y.Z/deploy/static/provider/baremetal/deploy.yaml
```

Verify the rollout:

```bash
kubectl rollout status deployment/ingress-nginx-controller -n ingress-nginx
kubectl get pods -n ingress-nginx
```

> ⚠️ Always check the [ingress-nginx changelog](https://github.com/kubernetes/ingress-nginx/blob/main/Changelog.md) before upgrading — some versions drop support for older Kubernetes API versions.

## Configuration

### LoadBalancer service

The bare metal installation uses a `NodePort` service by default. The `loadbalancer-service.yaml` adds a separate `LoadBalancer` service so MetalLB assigns a dedicated external IP for ingress traffic.

| Port | Protocol | Description |
|------|----------|-------------|
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |

### Useful global annotations

These annotations can be added per-Ingress resource:

| Annotation | Description |
|------------|-------------|
| `nginx.ingress.kubernetes.io/proxy-body-size` | Max upload size (default 1m) |
| `nginx.ingress.kubernetes.io/ssl-redirect` | Force HTTPS redirect |
| `nginx.ingress.kubernetes.io/proxy-read-timeout` | Read timeout in seconds |
| `nginx.ingress.kubernetes.io/proxy-send-timeout` | Send timeout in seconds |

## Troubleshooting

### Check ingress controller logs

```bash
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller -f
```

### List all ingress resources across cluster

```bash
kubectl get ingress -A
```

### Verify ingress is picking up a resource

```bash
kubectl describe ingress <name> -n <namespace>
```

## Notes

- ingress-nginx manifests are not stored in this repo — the official URL is the source of truth
- The bare metal install uses a `NodePort` service by default — `loadbalancer-service.yaml` adds MetalLB support on top
- All services in the cluster use `ingressClassName: nginx` to route through this controller
- TLS termination is handled by cert-manager certificates referenced in each Ingress resource