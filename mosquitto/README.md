# mosquitto

Eclipse Mosquitto MQTT broker for Home Assistant and Zigbee2MQTT.

## Files

| File | Description |
|------|-------------|
| `namespace.yaml` | Dedicated namespace |
| `serviceaccount.yaml` | Dedicated service account |
| `configmap.yaml` | Mosquitto configuration (no credentials) |
| `passwd-sealedsecret.yaml` | Sealed Secret containing the hashed passwd file |
| `pvc.yaml` | 5Gi Longhorn persistent volume for MQTT data |
| `service.yaml` | LoadBalancer service on 192.168.20.190 |
| `deployment.yaml` | Mosquitto deployment |

## Accessing

| Protocol | Address |
|----------|---------|
| MQTT | `mqtt://192.168.20.190:1883` |
| WebSockets | `ws://192.168.20.190:9001` |
| In-cluster | `mqtt://mosquitto-service.mosquitto.svc.cluster.local:1883` |

## Dependencies

- Longhorn storage
- Sealed Secrets controller in `kube-system`
- MetalLB or equivalent for LoadBalancer IP assignment

## Deployment

### Step 1: Generate the passwd SealedSecret

```bash
kubectl create secret generic mosquitto-passwd \
  --namespace mosquitto \
  --from-literal=passwd='mqttuser:HASHED_PASSWORD' \
  --dry-run=client -o yaml | \
  kubeseal \
    --controller-name=sealed-secrets-controller \
    --controller-namespace=kube-system \
    --format yaml > passwd-sealedsecret.yaml
```

To generate a new password hash:
```bash
docker run -it eclipse-mosquitto mosquitto_passwd -c /dev/stdout mqttuser
```

### Step 2: Apply all manifests

```bash
kubectl create namespace mosquitto
kubectl apply -f .
```

### Step 3: Verify

```bash
kubectl rollout status deployment/mosquitto -n mosquitto
kubectl get pods -n mosquitto

# Test MQTT connection
mosquitto_pub -h 192.168.20.190 -p 1883 -u mqttuser -P YOUR_PASSWORD -t test -m hello
```

## Upgrading

Update the image version in `deployment.yaml` and the version label in all files:

```yaml
image: eclipse-mosquitto:NEW_VERSION
app.kubernetes.io/version: "NEW_VERSION"
```

Then apply:
```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/mosquitto -n mosquitto
```

## Notes

- `passwd` file is mounted from a Secret (not ConfigMap) so credentials are not stored in plaintext
- `strategy: Recreate` is required since the PVC is ReadWriteOnce
- `nodePort` was removed — not needed when using a static LoadBalancer IP
- LoadBalancer IP `192.168.20.190` is managed by MetalLB
- `fsGroup: 1883` is set at the pod level so Kubernetes pre-owns all mounted volumes as the mosquitto user before startup — without this, the entrypoint tries to `chown` read-only ConfigMap/Secret mounts and crashes
- `runAsUser: 1883` and `runAsGroup: 1883` run the process as the mosquitto user directly
- Dropping `ALL` capabilities is not possible for mosquitto as it needs to drop privileges from root to the mosquitto user during startup