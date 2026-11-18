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
├── cert-manager/               # Let's Encrypt via Cloudflare DNS-01
├── praxis/                     # Habit tracker (SQLite on datavault)
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

Praxis has no dedicated IP: Traefik serves it at `https://praxis.lamiphil.dev`. UDM private DNS maps that name to `192.168.42.60`.

MetalLB uses `192.168.42.60-192.168.42.100`.

## Network Setup

- Homelab VLAN: `192.168.42.0/24`.
- WireGuard VPN: `192.168.69.0/24`.
- Traefik is pinned to `192.168.42.60`.
- The UniFi UDM is the local DNS authority for `.homelab` names.
- k3s uses pod CIDR `10.42.0.0/16` and service CIDR `10.43.0.0/16`.

## Node Topology

| Node | Role / workloads | Network |
|------|------------------|---------|
| `magos` | Single k3s server/control-plane and schedulable worker | Wi-Fi |
| `minipc` | Worker; Immich and File Browser | Wi-Fi |
| `datavault` | Worker and storage workloads; Praxis | Ethernet |

`ubuntuserver`, `pi`, and `archlinux` were retired on 2026-11-15. The surviving agents and local kubeconfig use `https://192.168.42.15:6443`. Because `magos` is the only control plane, its downtime makes the Kubernetes API unavailable.

## Storage

| Path | Type | Used by |
|------|------|---------|
| `192.168.42.11:/srv/media` | NFS | File Browser |
| `/srv/media/photos` | hostPath | Immich library |
| `/srv/media/immich-upload` | hostPath | Immich uploads |
| `/srv/immich/postgres/data` | hostPath | Immich PostgreSQL |
| `/srv/filebrowser/{database,config}` | hostPath | File Browser state |
| `datavault:/srv/praxis` | hostPath PV (Retain) | Praxis SQLite; must exist and belong to UID/GID `65532` |

HostPath volumes pin workloads to a node. Back up and move their data before changing node selectors.

## Monitoring

```
Kubernetes metrics --> Prometheus --> Grafana + Alertmanager
```

Monitoring workloads have no retired-node scheduling pins.

## Secrets Management

- SOPS + age encrypts files matching `*-secret.sops.yaml`.
- Flux decrypts secrets with the `sops-age` Secret in `flux-system`.
- Encrypted application secrets: `immich-db-secret.sops.yaml` and `praxis-secret.sops.yaml` (Praxis password and session key).
- `cert-manager/cloudflare-api-token-secret.sops.yaml` holds a Cloudflare token limited to `Zone:DNS:Edit` and `Zone:Zone:Read` on `lamiphil.dev`.

## TLS

- cert-manager (`v1.21.3`) issues Let's Encrypt certificates with the DNS-01 challenge on Cloudflare. Nothing is exposed to the Internet.
- ClusterIssuers: `letsencrypt-staging` (tests) and `letsencrypt-prod`.
- To secure an Ingress, add the annotation `cert-manager.io/cluster-issuer: letsencrypt-prod` and a `tls` block with a `secretName`.
- cert-manager checks the TXT records through `1.1.1.1`, because the UDM answers `lamiphil.dev` names locally.
- A new cert-manager CRD kind cannot be added in the same commit as cert-manager itself: the flat Flux Kustomization rejects unknown kinds.

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
