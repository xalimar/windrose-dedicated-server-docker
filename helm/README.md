# Windrose Dedicated Server — Helm Chart

Helm chart for running a Windrose dedicated server on Kubernetes.

The chart deploys a single-replica `Deployment`, two `PersistentVolumeClaims` (game data and Steam home), a `NodePort` `Service`, and optionally a `Secret` or `SealedSecret` for the server password and a `CronJob` for daily restarts.

---

## Requirements

| Component | Minimum |
|-----------|---------|
| Kubernetes | 1.26+ |
| Helm | 3.x |
| Persistent storage | A `StorageClass` that supports `ReadWriteOnce` |
| Sealed Secrets operator | Only if `config.password.sealed: true` |

---

## Quick start

```bash
helm install windrose ./helm \
  --namespace windrose \
  --create-namespace \
  --set config.serverName="My Server"
```

Or with a values file:

```bash
helm install windrose ./helm \
  --namespace windrose \
  --create-namespace \
  -f my-values.yaml
```

---

## ArgoCD (multiSource)

Point the chart source at this repo and supply your values file from a separate repo:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: windrose-dedicated
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: https://github.com/UberDudePL/windrose-dedicated-server-docker
      targetRevision: v1.4.0
      path: helm
      helm:
        valueFiles:
          - $values/windrose/values.yaml
    - repoURL: https://github.com/yourorg/your-gitops-repo
      targetRevision: HEAD
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: windrose
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

`$values` resolves to the root of the second source repo. Place your `values.yaml` at the path referenced in `valueFiles`.

---

## Configuration

All values have defaults. Override only what you need.

### `image`

| Key | Default | Description |
|-----|---------|-------------|
| `image.repository` | `ghcr.io/uberdudepl/windrose-dedicated-server-docker` | Container image repository |
| `image.tag` | `latest` | Image tag — pin to a release tag in production |

### `volumes`

| Key | Default | Description |
|-----|---------|-------------|
| `volumes.storageClassName` | `""` | StorageClass name. Empty uses the cluster default |
| `volumes.dataPVC.size` | `10Gi` | Size of the `/data` PVC (game saves, server config) |
| `volumes.steamHomePVC.size` | `5Gi` | Size of the `/home/steam` PVC (Wine prefix, SteamCMD cache) |

### `config`

| Key | Default | Description |
|-----|---------|-------------|
| `config.serverName` | `""` | Server name shown to players |
| `config.serverNote` | `""` | Optional server description |
| `config.maxPlayers` | `4` | Maximum concurrent players |
| `config.port` | `7777` | Game UDP port |
| `config.queryPort` | `7778` | Steam query UDP port |
| `config.multihome` | `0.0.0.0` | Network interface to bind |
| `config.p2pProxyAddress` | `127.0.0.1` | P2P proxy address |
| `config.inviteCode` | `""` | Invite code override. Leave empty to auto-generate |
| `config.updateOnStart` | `true` | Run SteamCMD update on every container start |
| `config.generateSettings` | `true` | Auto-generate `ServerDescription.json` from env vars |
| `config.puid` | `1000` | UID the server process runs as |
| `config.pgid` | `1000` | GID the server process runs as |

### `config.password`

| Key | Default | Description |
|-----|---------|-------------|
| `config.password.sealed` | `false` | Set to `true` to render a `SealedSecret` instead of a plain `Secret` |
| `config.password.value` | `""` | Plaintext password (used when `sealed: false`). Leave empty for no password |
| `config.password.sealedValue` | `""` | Kubeseal-encrypted value (used when `sealed: true`) |

To generate a sealed value:
```bash
kubeseal --raw --namespace windrose --name windrose-server-password --from-file /dev/stdin <<< "yourpassword"
```

### `config.restart`

Daily restart via a `CronJob` that deletes the pod and lets the `Deployment` recreate it.

| Key | Default | Description |
|-----|---------|-------------|
| `config.restart.enabled` | `false` | Enable the daily restart CronJob |
| `config.restart.timezone` | `UTC` | Timezone for the schedule (e.g. `America/New_York`) |
| `config.restart.hour` | `06` | Hour of restart in 24-hour format |
| `config.restart.minute` | `00` | Minute of restart |
| `config.restart.gracefulTimeout` | `30` | Seconds to wait before the pod is terminated |

### `resources`

| Key | Default | Description |
|-----|---------|-------------|
| `resources.requests.memory` | `4G` | Memory the scheduler reserves on the node (not a cap) |
| `resources.requests.cpu` | `1.0` | CPU the scheduler reserves on the node (not a cap) |

`requests` controls pod placement only — the container can use more if the node has capacity. To enforce a hard cap, add `resources.limits.memory` and `resources.limits.cpu` in your values.

---

## Persistence

| Mount path | PVC | Holds |
|------------|-----|-------|
| `/data` | `<release>-data` | Game saves, `ServerDescription.json`, world files |
| `/home/steam` | `<release>-steam-home` | Wine prefix, SteamCMD installation, Steam cache |

> **Note:** PVCs are not deleted when the Helm release is uninstalled. Delete them manually if you want to reset all data.

---

## Ports

The `Service` type is `NodePort`. Kubernetes assigns a random high port (30000–32767) on each cluster node that forwards to the container port. Connect to `<node-ip>:<assigned-nodePort>`.

| Name | Container port | Protocol |
|------|---------------|----------|
| game | `7777` | UDP |
| query | `7778` | UDP |

To use fixed node ports, set `service.nodePorts.game` and `service.nodePorts.query` in your values (not yet wired in the chart — add `nodePort:` to the Service template if needed).

> Players join via **Invite Code**, not a direct IP. The invite code is written to `/data/R5/ServerDescription.json` after the first successful start.
