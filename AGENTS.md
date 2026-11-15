# Homelab -- GitOps Kubernetes Cluster

**Repo**: `git@github.com:lamiphil/homelab.git` (branch: `main`)
**Stack**: k3s + Flux CD v2.7.3 + SOPS/age + MetalLB

## Architecture

Everything is declarative and Git-driven. Flux watches `clusters/homelab/`, reconciles every 10 minutes, and prunes resources removed from Git.

```
GitHub repo --> Flux --> Kubernetes resources
```

Do not use `kubectl apply` for persistent changes. The repository is the desired cluster state.

## Directory Structure

```
clusters/homelab/
├── filebrowser/                 # Browser-based file manager for /srv/media
├── flux-system/                # Flux bootstrap; do not edit gotk-components.yaml manually
├── homepage/                   # Landing page and service dashboard
├── immich/                     # Photo management, PostgreSQL, Valkey, and ML
├── metallb-system/             # Bare-metal LoadBalancer in L2 mode
├── monitoring/
│   └── kube-prometheus-stack/  # Prometheus, Grafana, and Alertmanager
└── traefik-helmchartconfig.yaml
```

## Service IPs

| IP | Service |
|----|---------|
| `192.168.42.60` | Traefik |
| `192.168.42.61` | Homepage |
| `192.168.42.70` | Grafana |
| `192.168.42.71` | Prometheus |
| `192.168.42.72` | Alertmanager |
| `192.168.42.80` | Immich |
| `192.168.42.96` | File Browser |

MetalLB uses `192.168.42.60-192.168.42.100`.

## Network Setup

- Homelab VLAN: `192.168.42.0/24`.
- WireGuard VPN: `192.168.69.0/24`.
- Traefik is pinned to `192.168.42.60`.
- The UniFi UDM is the local DNS authority for `.homelab` names.
- k3s uses pod CIDR `10.42.0.0/16` and service CIDR `10.43.0.0/16`.

## Target Node Topology

The node migration is in progress.

| Node | Target role / workloads |
|------|-------------------------|
| `magos` | Single k3s server/control-plane and schedulable worker |
| `minipc` | Worker; Immich and File Browser |
| `datavault` | Worker and storage workloads |

`ubuntuserver`, `pi`, and `archlinux` are being retired. Before retiring `ubuntuserver`, move Grafana because its HelmRelease is still pinned there, remove the control-plane taint from `magos`, and safely remove `ubuntuserver` from embedded etcd.

## Storage

| Path | Type | Used by |
|------|------|---------|
| `192.168.42.11:/srv/media` | NFS | File Browser |
| `/srv/media/photos` | hostPath | Immich library |
| `/srv/media/immich-upload` | hostPath | Immich uploads |
| `/srv/immich/postgres/data` | hostPath | Immich PostgreSQL |
| `/srv/filebrowser/{database,config}` | hostPath | File Browser state |

HostPath volumes pin workloads to a node. Back up and move their data before changing node selectors.

## Monitoring

```
Kubernetes metrics --> Prometheus --> Grafana + Alertmanager
```

Grafana is currently pinned to `ubuntuserver` in `monitoring/kube-prometheus-stack/helmrelease.yaml`.

## Secrets Management

- SOPS + age encrypts files matching `*-secret.sops.yaml`.
- Flux decrypts secrets with the `sops-age` Secret in `flux-system`.
- The remaining encrypted application secret is `immich-db-secret.sops.yaml`.

## Work Log

- `_log.md` is the canonical homelab work journal.
- Log investigations, decisions, configuration changes, and live-cluster operations.
- Add concise timestamped entries under newest-date-first headings.
- Never log credentials, tokens, or decrypted secret values.

## Patterns and Conventions

- Use HelmRelease for complex upstream charts and raw manifests for simple apps.
- Isolate application groups by namespace.
- Give deployments explicit resource requests and limits.
- Prefer pinned image versions over `latest`.
- Commit format: French `COMPONENT - Description`.
