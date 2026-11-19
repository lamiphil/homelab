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
├── authentik/                   # Identity provider, PostgreSQL, and R2 backups
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
| `192.168.42.80` | Immich |

Prometheus, Alertmanager, File Browser, and Praxis have no dedicated IP: Traefik serves them at `https://<service>.lamiphil.dev`. UDM private DNS maps those names to `192.168.42.60`.

## SSO (Authentik)

- Authentik runs at `https://auth.lamiphil.dev`. The admin account is `akadmin`; daily access uses `lamiphil`, a member of `homelab-users`.
- ForwardAuth: Proxy Provider `homelab-forward-auth` (domain level, cookie domain `lamiphil.dev`), application `Homelab`, served by the embedded outpost. Access is bound to `homelab-users`.
- Traefik Middleware `authentik-forward-auth` (one copy per namespace, in `authentik/forward-auth-middlewares.yaml`) protects Prometheus, Alertmanager, File Browser, and Praxis.
- Homepage is intentionally not protected. Never put the middleware on Authentik's own Ingress.
- A protected service must not keep a LoadBalancer IP: it would bypass Authentik.
- Grafana and Immich use native OIDC (Authentik applications `grafana` and `immich`, RS256 signing key). Grafana's client credentials are in `monitoring/kube-prometheus-stack/grafana-oidc-secret.sops.yaml`; Immich's OAuth settings live in Immich's admin UI. Both keep their local login as a break-glass path.

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
| `datavault` | Worker and storage workloads; Authentik and Praxis | Ethernet |

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
| `datavault:/srv/authentik/data` | hostPath | Authentik uploaded files and generated reports |
| `datavault:/srv/authentik/postgresql/data` | hostPath | Authentik PostgreSQL |

HostPath volumes pin workloads to a node. Back up and move their data before changing node selectors.

## Monitoring

```
Kubernetes metrics --> Prometheus --> Grafana + Alertmanager
Praxis SQLite (read-only) -----------> Grafana
```

- Grafana is pinned to `datavault`. It reads the live Praxis database through the `frser-sqlite-datasource` plugin (pinned to `4.0.6`, datasource UID `praxis-sqlite`, `mode=ro`).
- The read-only PV `praxis-data-grafana` mounts `datavault:/srv/praxis` at `/var/lib/praxis`. Grafana gets supplemental group `65532`, and `/srv/praxis` must be mode `750`.
- Dashboards are ConfigMaps labeled `grafana_dashboard=1`: `Homelab Cluster Health` and `Praxis — Habitudes` (UID `praxis-habits`).
- In `Praxis — Habitudes`, an unrecorded day is a gap and breaks streaks. Streaks count from today if today is saved, otherwise from yesterday. Habits appear only on the days they were active.

## Secrets Management

- SOPS + age encrypts files matching `*-secret.sops.yaml`.
- Flux decrypts secrets with the `sops-age` Secret in `flux-system`.
- Encrypted application secrets include `authentik/authentik-secret.sops.yaml`, `monitoring/kube-prometheus-stack/grafana-oidc-secret.sops.yaml`, `authentik/authentik-smtp-secret.sops.yaml`, `immich-db-secret.sops.yaml`, and `praxis-secret.sops.yaml`.
- `authentik/authentik-backup-r2-secret.sops.yaml` contains a token limited to the `authentik-backups` bucket.
- `cert-manager/cloudflare-api-token-secret.sops.yaml` holds a Cloudflare token limited to `Zone:DNS:Edit` and `Zone:Zone:Read` on `lamiphil.dev`.

## Praxis backups

- CronJob `praxis/praxis-backup` runs nightly at 03:30 America/Toronto. It takes a `sqlite3 .backup` snapshot, checks `PRAGMA integrity_check`, and uploads `praxis-<UTC timestamp>.db` to the Cloudflare R2 bucket `praxis-backups`.
- An R2 lifecycle rule deletes copies after 30 days. The R2 token (Object Read & Write, this bucket only) is in `praxis/praxis-backup-r2-secret.sops.yaml`.
- A failed run fires `KubeJobFailed`. To run a backup now: `kubectl -n praxis create job praxis-backup-manual --from=cronjob/praxis-backup`.

Restore:

1. Download the wanted `praxis-*.db` from the R2 dashboard, then `scp` it to `datavault:/tmp/`.
2. Suspend Flux so that it does not scale Praxis back up: `kubectl -n flux-system patch kustomization flux-system --type merge -p '{"spec":{"suspend":true}}'`.
3. `kubectl -n praxis scale deploy/praxis --replicas=0`.
4. On `datavault`: `sudo mkdir /srv/praxis/old && sudo mv /srv/praxis/praxis.db* /srv/praxis/old/`, then `sudo install -o 65532 -g 65532 -m 644 /tmp/praxis-<ts>.db /srv/praxis/praxis.db`.
5. Resume Flux (`"suspend":false`). Flux scales Praxis back to 1 replica. Check the data, then delete `/srv/praxis/old`.

## Authentik backups

- CronJob `authentik/authentik-backup` runs nightly at 04:00 America/Toronto. It validates and uploads a custom-format PostgreSQL dump plus a compressed copy of `/srv/authentik/data` to the Cloudflare R2 bucket `authentik-backups`.
- An R2 lifecycle rule deletes copies after 30 days. The token has Object Read & Write access to this bucket only.
- To run a backup now: `kubectl -n authentik create job authentik-backup-manual --from=cronjob/authentik-backup`.

Restore:

1. Download a matching `authentik-<ts>.dump` and `authentik-<ts>-data.tar.gz` pair from R2.
2. Suspend Flux and scale `authentik-server` and `authentik-worker` to zero.
3. Copy the dump into the PostgreSQL pod and restore it with `pg_restore --clean --if-exists --no-owner --single-transaction --exit-on-error --username=authentik --dbname=authentik`.
4. Back up the current `/srv/authentik/data`, extract the matching data archive there, and restore UID/GID `1000` ownership.
5. Resume Flux, wait for both Authentik deployments, and test login before deleting the temporary restore files.

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
