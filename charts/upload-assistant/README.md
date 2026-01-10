# upload-assistant (Web UI)

Helm chart for running Upload-Assistant's Web UI (Docker GUI) on Kubernetes.

## Prerequisites

- Kubernetes cluster
- Helm 3
- A valid Upload-Assistant `config.py` (see https://github.com/Audionut/Upload-Assistant/wiki/Configuration)

## TL;DR

1. Pick a namespace (example: `upload-assistant`):

   ```bash
   kubectl create namespace upload-assistant --dry-run=client -o yaml | kubectl apply -f -
   ```

2. Create a Secret containing your `config.py` (recommended):

   ```bash
   kubectl -n upload-assistant create secret generic upload-assistant-config \
     --from-file=config.py=/path/to/config.py
   ```

3. Install/upgrade the chart:

   ```bash
   helm upgrade --install upload-assistant ./charts/upload-assistant \
     --namespace upload-assistant --create-namespace \
     -f my-values.yaml
   ```

## Accessing the Web UI

- `ClusterIP` (default): `kubectl -n upload-assistant port-forward deploy/upload-assistant 5000:5000`
- `LoadBalancer`: `kubectl -n upload-assistant get svc upload-assistant -w`
- `Ingress`: set `ingress.enabled=true` and configure `ingress.hosts`

If you expose the Web UI beyond localhost/LAN (especially with `LoadBalancer` or `Ingress`), strongly consider setting:

- `UA_WEBUI_USERNAME` and `UA_WEBUI_PASSWORD`
- `UA_WEBUI_CORS_ORIGINS` (only if serving the UI from a different origin/domain)

## Uninstalling the chart

```bash
helm uninstall upload-assistant -n upload-assistant
kubectl delete namespace upload-assistant
```

## Configuration

All values (and defaults) are documented in `values.yaml`. A fuller working example is also available at the repo root: `upload-assistant-values.yaml`.

### Providing `config.py`

Upload-Assistant reads configuration from `/Upload-Assistant/data/config.py`.

Recommended approach (default):

- Create a Secret named `upload-assistant-config` containing a `config.py` key.
- Keep `config.enabled=true` and `config.existingSecret=upload-assistant-config`.
  - The Secret/ConfigMap must exist in the same namespace as the Helm release.

Alternatives:

- Provide an existing ConfigMap via `config.existingConfigMap`
- Inline config (chart-managed Secret/ConfigMap) via `config.kind` + `config.configPy` (convenient for quick testing; avoid committing real credentials)

#### Inline config details

To have the chart create the Secret/ConfigMap for you:

- Set `config.existingSecret=""` and `config.existingConfigMap=""`
- Set `config.kind` to `secret` or `configMap`
- Put the contents of your `config.py` file into `config.configPy` (use a YAML block scalar like `|-` and keep indentation)
- If you don’t need `config.py` to be writable, set `config.copyToVolume=false` to mount it read-only at `config.mountPath`

The chart creates a resource named `<release fullname>-config` containing the key `config.key` (defaults to `config.py`).

Example:

```yaml
config:
  enabled: true
  existingSecret: ""
  existingConfigMap: ""
  kind: secret # secret | configMap
  key: config.py
  copyToVolume: true
  overwrite: true
  configPy: |-
    config = {
        "DEFAULT": {
            "tmdb_api": "<your-tmdb-api-key>",
        }
    }
```

If `config.copyToVolume=true`, the chart copies the Secret/ConfigMap file into the writable `persistence.uaData` volume on startup so it can be persisted/edited. Use `config.overwrite` to control whether it is replaced on every start.

### Persistence

- `persistence.uaData`: backs `/Upload-Assistant/data` (must be writable; this is where `config.py` lives)
- `persistence.tmp`: backs `/Upload-Assistant/tmp` (recommended)
- `persistence.files`: optional mount (often NFS) so the UI can browse your downloads/media (typical mountPath: `/data`)
- `persistence.torrentStorage`: optional mount for torrent client state (e.g. qBittorrent `BT_backup`) used for torrent re-use

If you use the Web UI file browser, set `persistence.files.mountPath` to match the `local_path` you configure in `config.py`.

```yaml
persistence:
  files:
    enabled: true
    type: nfs
    mountPath: /data
    nfs:
      server: <nfs-server-ip>
      path: /path/to/share

  uaData:
    enabled: true
    type: pvc
    storageClassName: <your-storage-class>
    size: 1Gi

  tmp:
    enabled: true
    type: emptyDir
```

### Service exposure and Web UI auth

By default, the chart exposes Upload-Assistant using a `ClusterIP` Service, which can be accessed through `kubectl port-forward -n upload-assistant deployment/upload-assistant 5000:5000`, then opening your browser to `http://localhost:5000`.

If willing to expose upload-assistant (with possible security risks), you can change the Service type to `LoadBalancer` or enable Ingress.

```yaml
service:
  type: LoadBalancer
```

or enable Ingress:

```yaml
# Optional ingress
ingress:
  enabled: true
  className: "traefik" # nginx, traefik, etc.
  annotations: {}
  hosts:
    - host: upload.example.com
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []
```

You should also set `UA_WEBUI_USERNAME` and `UA_WEBUI_PASSWORD` to protect access to the Web UI if continuing beyond localhost/LAN.

```yaml
env:
  # Needed if exposed using Ingress or LoadBalancer (optional but strongly recommended).
  UA_WEBUI_USERNAME: admin
  UA_WEBUI_PASSWORD: change-me
  # Optional: only needed if you serve the UI from a different origin/domain.
  UA_WEBUI_CORS_ORIGINS: https://your-ui-host

# Or, keep credentials in a Secret and reference it via envFrom:
# envFrom:
#   - secretRef:
#       name: upload-assistant-webui-auth
```

> `env` is a map and is merged on top of the chart defaults.

### VPN sidecar (Gluetun)

To route Upload-Assistant through a VPN, enable the Gluetun sidecar. All containers in a Pod share the same network namespace, so Upload-Assistant traffic will automatically use the tunnel once Gluetun is up.

If you want to prevent any outbound traffic before the VPN is ready, override the main container `command`/`args` to wait for `tun0` (OpenVPN) or `wg0` (WireGuard).

#### Example for ProtonVPN (OpenVPN)

First, create a Secret containing your OpenVPN credentials:
```bash
kubectl -n upload-assistant create secret generic protonvpn \
  --from-literal=OPENVPN_USER='your-username' \
  --from-literal=OPENVPN_PASSWORD='your-password'
```

Add the following to your `values.yaml`:

```yaml
command: ["/bin/sh", "-lc"]
args:
  - |
    echo "Waiting for VPN (tun0)…"
    until [ -d /sys/class/net/tun0 ] || grep -q "tun0" /proc/net/dev; do sleep 1; done
    exec /bin/bash -lc 'source /venv/bin/activate && python /Upload-Assistant/web_ui/server.py'

vpn:
  enabled: true
  type: gluetun
  gluetun:
    tunDevice:
      enabled: true
    # `vpn.gluetun.env` supports either a list (env-style) or a simple map, and supports `valueFrom`.
    env:
      VPN_SERVICE_PROVIDER: protonvpn
      VPN_TYPE: openvpn
      FIREWALL_OUTBOUND_SUBNETS: 10.42.0.0/16,10.43.0.0/16,192.168.0.0/16
      FIREWALL_INPUT_PORTS: "5000"
      OPENVPN_USER:
        valueFrom:
          secretKeyRef:
            name: protonvpn
            key: OPENVPN_USER
      OPENVPN_PASSWORD:
        valueFrom:
          secretKeyRef:
            name: protonvpn
            key: OPENVPN_PASSWORD
```

## Parameters

| Key | Description | Default |
| --- | --- | --- |
| `image.repository` | Image repository | `ghcr.io/audionut/upload-assistant` |
| `image.tag` | Image tag (defaults to chart `appVersion`) | `""` |
| `service.type` | Service type | `ClusterIP` |
| `service.port` | Service port | `5000` |
| `ingress.enabled` | Enable ingress | `false` |
| `config.enabled` | Enable config management | `true` |
| `config.existingSecret` | Existing Secret name containing `config.key` | `upload-assistant-config` |
| `config.existingConfigMap` | Existing ConfigMap name containing `config.key` | `""` |
| `config.kind` | Create a `secret` or `configMap` when not using `existingSecret`/`existingConfigMap` | `secret` |
| `config.key` | Key name in the Secret/ConfigMap | `config.py` |
| `config.mountPath` | Mount/copy destination path | `/Upload-Assistant/data/config.py` |
| `config.copyToVolume` | Copy config into `persistence.uaData` (writable) | `true` |
| `config.overwrite` | Overwrite destination on every start | `true` |
| `config.configPy` | Inline `config.py` contents (used when creating Secret/ConfigMap) | `""` |
| `persistence.uaData.enabled` | Enable `/Upload-Assistant/data` volume | `true` |
| `persistence.uaData.type` | Storage type for `/Upload-Assistant/data` | `pvc` |
| `persistence.tmp.enabled` | Enable `/Upload-Assistant/tmp` volume | `true` |
| `persistence.tmp.type` | Storage type for `/Upload-Assistant/tmp` | `emptyDir` |
| `persistence.files.enabled` | Enable optional files volume (UI browser) | `false` |
| `persistence.files.type` | Storage type for files volume | `nfs` |
| `env` | Main container env vars (map) | `{ENABLE_WEB_UI: "true", UA_WEBUI_HOST: "0.0.0.0", UA_WEBUI_PORT: "5000", UA_BROWSE_ROOTS: "/data,/Upload-Assistant"}` |
| `envFrom` | Main container envFrom | `[]` |
| `vpn.enabled` | Enable VPN sidecar | `false` |
| `vpn.type` | VPN sidecar type | `gluetun` |
| `vpn.gluetun.image.repository` | Gluetun image repository | `qmcgaw/gluetun` |
| `vpn.gluetun.image.tag` | Gluetun image tag | `v3.41.0` |
| `vpn.gluetun.tunDevice.enabled` | Mount host `/dev/net/tun` into Gluetun | `false` |
| `vpn.gluetun.env` | Gluetun env (list or map; supports `valueFrom`) | `null` |
