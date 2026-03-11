# zigbee2mqtt

Zigbee to MQTT bridge connecting Zigbee devices to Home Assistant via Mosquitto.

## Files

| File | Description |
|------|-------------|
| `namespace.yaml` | Dedicated namespace |
| `certificate.yaml` | TLS certificate via cert-manager |
| `serviceaccount.yaml` | Dedicated service account |
| `pvc.yaml` | 1Gi Longhorn persistent volume for zigbee2mqtt data |
| `service.yaml` | ClusterIP service on port 8080 |
| `deployment.yaml` | zigbee2mqtt deployment |
| `ingress.yaml` | nginx ingress with TLS |
| `mqtt-secret-sealedsecret.yaml` | Sealed Secret for MQTT password |

## Accessing

https://zigbee2mqtt.petkus.id.lv

## Dependencies

- cert-manager with `cloudflare-clusterissuer` ClusterIssuer
- nginx ingress controller
- Longhorn storage
- Mosquitto MQTT broker running in `mosquitto` namespace
- Sealed Secrets controller in `kube-system`

## Deployment

### Step 1: Generate the MQTT SealedSecret

```bash
kubectl create secret generic zigbee2mqtt-mqtt-secret \
  --namespace zigbee2mqtt \
  --from-literal=mqtt-password=YOUR_MQTT_PASSWORD \
  --dry-run=client -o yaml | \
  kubeseal \
    --controller-name=sealed-secrets-controller \
    --controller-namespace=kube-system \
    --format yaml > mqtt-secret-sealedsecret.yaml
```

> The password must match the one stored in mosquitto's `passwd` SealedSecret.
> You can verify the mosquitto password is correct by testing directly:
> ```bash
> kubectl exec -it -n mosquitto deployment/mosquitto -- \
>   mosquitto_pub -h localhost -p 1883 -u mqttuser -P YOUR_PASSWORD -t test -m hello
> ```

### Step 2: Apply all manifests

```bash
kubectl create namespace zigbee2mqtt
kubectl apply -f .
```

### Step 3: Verify

```bash
kubectl rollout status deployment/zigbee2mqtt -n zigbee2mqtt
kubectl get pods -n zigbee2mqtt
kubectl logs -n zigbee2mqtt deployment/zigbee2mqtt -f
```

A successful startup looks like:
```
info: z2m: Connecting to MQTT server at mqtt://mosquitto-service.mosquitto.svc.cluster.local:1883
info: z2m: Connected to MQTT server
```

## Upgrading

Update the image version in `deployment.yaml` and the version label in all files:

```yaml
image: koenkk/zigbee2mqtt:NEW_VERSION
app.kubernetes.io/version: "NEW_VERSION"
```

Then apply:
```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/zigbee2mqtt -n zigbee2mqtt
```

## Configuration

zigbee2mqtt is configured via environment variables in `deployment.yaml`. Key settings:

| Variable | Value | Description |
|----------|-------|-------------|
| `ZIGBEE2MQTT_CONFIG_SERIAL_PORT` | `tcp://192.168.100.2:6638` | Zigbee coordinator address |
| `ZIGBEE2MQTT_CONFIG_SERIAL_ADAPTER` | `zstack` | Coordinator type |
| `ZIGBEE2MQTT_CONFIG_MQTT_SERVER` | `mqtt://mosquitto-service.mosquitto.svc.cluster.local:1883` | MQTT broker |
| `ZIGBEE2MQTT_CONFIG_MQTT_USER` | `mqttuser` | MQTT username |
| `ZIGBEE2MQTT_CONFIG_MQTT_PASSWORD` | from SealedSecret | MQTT password |
| `ZIGBEE2MQTT_CONFIG_ADVANCED_CHANNEL` | `25` | Zigbee channel |
| `ZIGBEE2MQTT_CONFIG_FRONTEND_PACKAGE` | `zigbee2mqtt-windfront` | Custom frontend |

## Troubleshooting

### MQTT authorization error
```
MQTT failed to connect, exiting... (Connection refused: Not authorized)
```
The MQTT password in `zigbee2mqtt-mqtt-secret` doesn't match mosquitto's `passwd` file. Re-seal with the correct password:
```bash
kubectl create secret generic zigbee2mqtt-mqtt-secret \
  --namespace zigbee2mqtt \
  --from-literal=mqtt-password=CORRECT_PASSWORD \
  --dry-run=client -o yaml | \
  kubeseal \
    --controller-name=sealed-secrets-controller \
    --controller-namespace=kube-system \
    --format yaml > mqtt-secret-sealedsecret.yaml

kubectl delete sealedsecret zigbee2mqtt-mqtt-secret -n zigbee2mqtt
kubectl apply -f mqtt-secret-sealedsecret.yaml
kubectl rollout restart deployment/zigbee2mqtt -n zigbee2mqtt
```

### Verify the stored password
```bash
kubectl get secret zigbee2mqtt-mqtt-secret -n zigbee2mqtt \
  -o jsonpath='{.data.mqtt-password}' | base64 -d
```

## Notes

- MQTT password is stored as a SealedSecret — safe to commit to git
- `strategy: Recreate` is required since the PVC is ReadWriteOnce
- `replicas: 1` — zigbee2mqtt does not support multi-replica deployments
- Zigbee coordinator is accessed over TCP (`tcp://192.168.100.2:6638`) — no USB passthrough needed
- `NET_ADMIN` and `NET_RAW` capabilities are not needed since the coordinator is TCP-based
- Probes use `tcpSocket` on port 8080 to avoid authentication issues with HTTP probes