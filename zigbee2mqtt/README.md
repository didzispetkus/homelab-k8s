# zigbee2mqtt

Zigbee to MQTT bridge connecting Zigbee devices to Home Assistant via Mosquitto.

## Files

| File | Description |
|------|-------------|
| `namespace.yaml` | Dedicated namespace |
| `serviceaccount.yaml` | Dedicated service account |
| `pvc.yaml` | 1Gi Longhorn persistent volume for zigbee2mqtt data |
| `service.yaml` | ClusterIP service on port 8080 |
| `deployment.yaml` | zigbee2mqtt deployment |
| `httproute.yaml` | Traefik Gateway API HTTPRoute |
| `mqtt-secret-sealedsecret.yaml` | Sealed Secret for MQTT password and Zigbee network key |

## Accessing

https://zigbee2mqtt.petkus.id.lv

## Dependencies

- Traefik Gateway API controller in `traefik` namespace
- Longhorn storage
- Mosquitto MQTT broker running in `mosquitto` namespace
- Sealed Secrets controller in `kube-system`

## Deployment

### Step 1: Generate the MQTT SealedSecret

The SealedSecret contains two keys — `mqtt-password` and `network-key`:

```bash
kubectl create secret generic zigbee2mqtt-mqtt-secret \
  --namespace zigbee2mqtt \
  --from-literal=mqtt-password=YOUR_MQTT_PASSWORD \
  --from-literal=network-key='[42,148,140,178,107,50,113,139,235,234,62,37,103,249,20,52]' \
  --dry-run=client -o yaml | \
  kubeseal \
    --controller-name=sealed-secrets-controller \
    --controller-namespace=kube-system \
    --format yaml > mqtt-secret-sealedsecret.yaml
```

> The MQTT password must match the one stored in mosquitto's `passwd` SealedSecret.
> You can verify it is correct by testing directly:
> ```bash
> kubectl exec -it -n mosquitto deployment/mosquitto -- \
>   mosquitto_pub -h localhost -p 1883 -u mqttuser -P YOUR_PASSWORD -t test -m hello
> ```

> The network key must be a 16-byte **decimal** array — zigbee2mqtt does not accept hex notation.
> The PAN ID (`ZIGBEE2MQTT_CONFIG_ADVANCED_PAN_ID`) is a plain env var in `deployment.yaml`
> and must also be decimal.

### Step 2: Apply all manifests

```bash
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
| `ZIGBEE2MQTT_CONFIG_MQTT_PASSWORD` | from SealedSecret (`mqtt-password`) | MQTT password |
| `ZIGBEE2MQTT_CONFIG_ADVANCED_NETWORK_KEY` | from SealedSecret (`network-key`) | Zigbee network key |
| `ZIGBEE2MQTT_CONFIG_ADVANCED_PAN_ID` | `24037` | Zigbee PAN ID (decimal) |
| `ZIGBEE2MQTT_CONFIG_ADVANCED_CHANNEL` | `25` | Zigbee channel |
| `ZIGBEE2MQTT_CONFIG_FRONTEND_PACKAGE` | `zigbee2mqtt-windfront` | Custom frontend |
| `ZIGBEE2MQTT_CONFIG_PERMIT_JOIN` | `false` | Disable joining after initial setup |

## Pairing New Devices

Joining is disabled by default. To pair a new device, temporarily enable it via MQTT:

```bash
kubectl exec -it -n mosquitto deployment/mosquitto -- \
  mosquitto_pub -h localhost -p 1883 -u mqttuser -P YOUR_PASSWORD \
  -t 'zigbee2mqtt/bridge/request/permit_join' \
  -m '{"value": true, "time": 254}'
```

Once paired, disable joining again:
```bash
kubectl exec -it -n mosquitto deployment/mosquitto -- \
  mosquitto_pub -h localhost -p 1883 -u mqttuser -P YOUR_PASSWORD \
  -t 'zigbee2mqtt/bridge/request/permit_join' \
  -m '{"value": false}'
```

## Troubleshooting

### MQTT authorization error
```
MQTT failed to connect, exiting... (Connection refused: Not authorized)
```
The MQTT password in `zigbee2mqtt-mqtt-secret` doesn't match mosquitto's `passwd` file.
Re-seal both keys together with the correct password:
```bash
kubectl create secret generic zigbee2mqtt-mqtt-secret \
  --namespace zigbee2mqtt \
  --from-literal=mqtt-password=CORRECT_PASSWORD \
  --from-literal=network-key='[42,148,140,178,107,50,113,139,235,234,62,37,103,249,20,52]' \
  --dry-run=client -o yaml | \
  kubeseal \
    --controller-name=sealed-secrets-controller \
    --controller-namespace=kube-system \
    --format yaml > mqtt-secret-sealedsecret.yaml

kubectl delete sealedsecret zigbee2mqtt-mqtt-secret -n zigbee2mqtt
kubectl apply -f mqtt-secret-sealedsecret.yaml
kubectl rollout restart deployment/zigbee2mqtt -n zigbee2mqtt
```

### Network key / PAN ID mismatch
```
Configuration is not consistent with adapter state/backup!
```
The network key or PAN ID doesn't match what the coordinator was initialized with.
Either update the secret to match the coordinator's values, or delete the coordinator
backup to force re-commissioning (all paired devices will need to be re-paired):
```bash
kubectl exec -it -n zigbee2mqtt deployment/zigbee2mqtt -- \
  rm /data/coordinator_backup.json
kubectl delete pod -n zigbee2mqtt -l app.kubernetes.io/name=zigbee2mqtt
```

### Verify stored secrets
```bash
# MQTT password
kubectl get secret zigbee2mqtt-mqtt-secret -n zigbee2mqtt \
  -o jsonpath='{.data.mqtt-password}' | base64 -d

# Network key
kubectl get secret zigbee2mqtt-mqtt-secret -n zigbee2mqtt \
  -o jsonpath='{.data.network-key}' | base64 -d
```

## Notes

- Both `mqtt-password` and `network-key` are stored in a single SealedSecret — safe to commit to git
- TLS is handled by the wildcard cert (`*.petkus.id.lv`) in the `traefik` namespace — no per-app certificate needed
- `strategy: Recreate` is required since the PVC is ReadWriteOnce
- `replicas: 1` — zigbee2mqtt does not support multi-replica deployments
- Zigbee coordinator is accessed over TCP (`tcp://192.168.100.2:6638`) — no USB passthrough needed
- `NET_ADMIN` and `NET_RAW` capabilities are not needed since the coordinator is TCP-based
- Probes use `tcpSocket` on port 8080 to avoid authentication issues with HTTP probes
- Network key and PAN ID must be in decimal format — zigbee2mqtt does not accept hex values