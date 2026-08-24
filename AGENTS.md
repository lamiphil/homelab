# Homelab -- GitOps Kubernetes Cluster

**Repo**: `git@github.com:lamiphil/homelab.git` (branch: `main`)
**Stack**: k3s + Flux CD v2.7.3 + SOPS/age + MetalLB

## Architecture

Everything is declarative and Git-driven. Flux CD watches this repo and reconciles all manifests under `clusters/homelab/` every 10 minutes. No manual `kubectl apply` -- the repo IS the cluster state.

```
GitHub repo --> Flux GitRepository (polls every 1m)
            --> Flux Kustomization (reconciles every 10m)
            --> All manifests applied to k3s cluster
            --> SOPS secrets decrypted via age key
```

## Directory Structure

```
clusters/homelab/
├── drawio/                # Self-hosted draw.io / diagrams.net (LoadBalancer at drawio.homelab)
├── filebrowser/           # Browser-based file manager for /srv/media (LoadBalancer at filebrowser.homelab)
├── flux-system/           # Flux CD bootstrap (auto-generated, do not manually edit gotk-components.yaml)
├── headlamp/              # Kubernetes web dashboard (Ingress at headlamp.homelab)
├── homepage/              # Landing page / service dashboard (gethomepage)
├── immich/                # Self-hosted Google Photos (v2.0.0, PostgreSQL + pgvecto-rs)
├── metallb-system/        # Bare-metal LoadBalancer (L2 mode, IPs 192.168.42.60-100)
├── monitoring/
│   ├── kube-prometheus-stack/  # Prometheus + Grafana + Alertmanager
│   └── loki/                   # Loki (logs) + Alloy (collector DaemonSet)
├── nvidia-device-plugin/  # GPU passthrough for Jellyfin transcoding
├── pihole/                # DNS ad blocker
└── streaming/             # Full media automation stack
    ├── flaresolverr/      # CAPTCHA bypass proxy (internal only)
    ├── jellyfin/          # Media player (NVIDIA GPU transcoding)
    ├── prowlarr/          # Indexer manager
    ├── qbittorrent/       # Torrent client (behind Gluetun VPN sidecar)
    ├── qbittorrent-direct/# Torrent client (no VPN, for debugging)
    ├── qui/               # qBittorrent web UI (autobrr)
    ├── radarr/            # Movie manager
    ├── seer/              # Jellyseerr (media request portal)
    └── sonarr/            # TV show manager
```

## IP Address Map (MetalLB L2)

| IP | Service |
|----|---------|
| 192.168.42.60-100 | MetalLB pool range |
| .60 | Traefik |
| .61 | Homepage |
| .62 | Pi-hole DNS |
| .63 | Pi-hole Web UI |
| .70 | Grafana |
| .71 | Prometheus |
| .72 | Alertmanager |
| .80 | Immich |
| .90 | Jellyfin |
| .91 | Jellyseerr |
| .92 | Prowlarr |
| .93 | qUI |
| .94 | Radarr |
| .95 | Sonarr |
| .96 | File Browser |
| .97 | draw.io |

## Network Setup

- **Primary homelab VLAN/subnet**: `192.168.42.0/24`.
- **MetalLB mode**: L2 advertisement using `default-pool` (`192.168.42.60-192.168.42.100`).
- **Traefik entrypoint**: k3s-managed Traefik is pinned to `192.168.42.60` via `HelmChartConfig`.
- **Local DNS**: Pi-hole is exposed at `192.168.42.62` for DNS and `192.168.42.63` for the web UI. `.homelab` hostnames are used throughout Homepage and Ingresses; DNS records are expected to live outside this repo, likely in Pi-hole/local DNS.
- **Kubernetes network CIDRs**: pod CIDR appears to be `10.42.0.0/16`, service CIDR `10.43.0.0/16` (referenced in the Gluetun firewall allowlist).
- **LAN access from VPN sidecars**: qBittorrent/Gluetun allows outbound access to `10.42.0.0/16`, `10.43.0.0/16`, and `192.168.42.0/24`.
- **External/network tools referenced**: router at `192.168.0.1`, UniFi controller bookmark, Tailscale admin bookmark, ProtonVPN for qBittorrent WireGuard.
- **Exposure pattern**: most services use MetalLB `LoadBalancer` IPs directly; Headlamp uses an Ingress at `headlamp.homelab` through Traefik.

## Node Topology

| Node | Role / Workloads |
|------|-----------------|
| `ubuntuserver` | Grafana |
| `minipc` | Immich (server, ML, Valkey, PostgreSQL), File Browser |
| GPU node (label: `nvidia.com/gpu.present=true`, taint: `dedicated=transcoding`) | Jellyfin (NVIDIA GPU transcoding) |
| Streaming node (label: `role=streaming`) | qBittorrent, Radarr, Sonarr, Prowlarr, FlareSolverr |

## NFS Storage (server: 192.168.42.11)

| NFS Path | PV | Access | Used By |
|----------|----|--------|---------|
| `/srv/media` | media-pv (2Ti) | ReadOnlyMany | Jellyfin |
| `/srv/media` | filebrowser-media (2Ti) | ReadWriteMany | File Browser |
| `/srv/media/photos` | library-pv (2Ti) | ReadWriteMany | Immich |

HostPath storage: `/srv/streaming/*/config` for app configs, `/srv/media/downloads` for torrent downloads, `/srv/filebrowser/{database,config}` for File Browser app state on `minipc`.

## Media Automation Pipeline

```
User request --> Jellyseerr (:5055)
             --> Radarr (movies, :7878) / Sonarr (TV, :8989)
             --> Prowlarr (indexer search, :9696) --> FlareSolverr (CAPTCHA, :8191)
             --> qBittorrent (download via VPN/Gluetun, :8080)
             --> /srv/media/downloads --> Radarr moves to /Movies, Sonarr to /TV
             --> Jellyfin (playback, :8096, GPU transcoding)
```

## Monitoring Pipeline

```
All pods --> Grafana Alloy (DaemonSet) --> Loki (http://loki:3100) --> Grafana (data source)
K8s metrics --> Prometheus (kube-prometheus-stack) --> Grafana + Alertmanager
```

## Secrets Management

- **SOPS + age** encryption. Rule in `.sops.yaml`: files matching `*-secret.sops.yaml`.
- Age public key: `age1lh9d945vv33slydy9kavezqh34jsqapa3prdvm82twqnem3uxcfq724qwd`
- Flux decrypts in-cluster using `sops-age` Secret in `flux-system` namespace.
- Encrypted secrets: `immich-db-secret.sops.yaml`, `qbittorrent-vpn-wg-secret.sops.yaml`

## Work Log

- `_log.md` at the repository root is the canonical journal for all homelab work.
- Log every investigation, decision, configuration change, live-cluster operation, and other progress or action related to the homelab, including work that does not change files in this repository.
- Add each update to `_log.md` as a concise timestamped entry under the appropriate date heading; keep the newest date sections first and never remove previous entries.
- Never include secrets, credentials, tokens, or decrypted SOPS values in the log.

## Patterns and Conventions

- **Helm vs Raw**: Helm (HelmRelease) for complex upstream charts; raw K8s manifests for simpler single-container apps.
- **OCI Repositories**: Immich and Jellyseerr use OCIRepository (GHCR) instead of traditional HelmRepository.
- **Namespace isolation**: Each service group gets its own namespace.
- **LinuxServer.io images**: The arr stack uses `lscr.io/linuxserver/*` images with `PUID=1000`, `PGID=1000`, `TZ=America/Toronto`.
- **VPN sidecar pattern**: qBittorrent uses Gluetun (WireGuard) as a sidecar container. A parallel `qbittorrent-direct` exists for debugging without VPN.
- **Resource limits**: All deployments have explicit CPU/memory requests and limits.
- **Commit messages**: French, format `COMPONENT - Description` (e.g., `JELLYFIN - Correction StorageClassName manquant`).
