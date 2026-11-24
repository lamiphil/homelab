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
├── filebrowser/                 # Browser-based file manager for /srv/media/Documents
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

Immich, Grafana, Prometheus, Alertmanager, File Browser, and Praxis have no dedicated IP: Traefik serves them at `https://<service>.lamiphil.dev`. UDM private DNS maps those names to `192.168.42.60`.

## SSO (Authentik)

- Authentik runs at `https://auth.lamiphil.dev`. One break-glass administrator account is used only for recovery; day-to-day work uses a separate account that belongs to `homelab-users`.
- ForwardAuth: Proxy Provider `homelab-forward-auth` (domain level, cookie domain `lamiphil.dev`), application `Homelab`, served by the embedded outpost. Access is bound to `homelab-users`.
- Traefik Middleware `authentik-forward-auth` (one copy per namespace, in `authentik/forward-auth-middlewares.yaml`) protects Prometheus, Alertmanager, File Browser, and Praxis.
- Homepage is intentionally not protected. Never put the middleware on Authentik's own Ingress.
- A protected service must not keep a LoadBalancer IP: it would bypass Authentik.
- Grafana and Immich use native OIDC (Authentik applications `grafana` and `immich`, RS256 signing key). Grafana's client credentials are in `monitoring/kube-prometheus-stack/grafana-oidc-secret.sops.yaml`; Immich's OAuth settings live in Immich's admin UI. Both keep their local login as a break-glass path.
- File Browser and Praxis trust the username forwarded by the middleware so a single Authentik login opens them without a second prompt:
  - File Browser is set to `auth.method=proxy` with header `X-authentik-username`. The user is `lamiphil` with admin scope `.`. The initContainer `configure-proxy-auth` on the Deployment rewrites `auth.method` and the user name idempotently on every pod start (BoltDB is locked while the server runs).
  - Praxis trusts the header when the env var `PRAXIS_TRUSTED_HEADER=X-authentik-username` is set (from v0.1.3 onwards). The password login stays reachable at `/login` for port-forward break-glass.
- Traefik's `authResponseHeaders` list in `authentik/forward-auth-middlewares.yaml` overwrites every `X-authentik-*` header on the incoming request, so an outside client cannot spoof them. To close the in-cluster spoof path, a `NetworkPolicy` named `allow-traefik-only` in the `filebrowser` and `praxis` namespaces restricts ingress to the Traefik pod in `kube-system`. k3s enforces NetworkPolicies through its built-in kube-router controller. Grafana reads Praxis through a read-only PV mounted from `/srv/praxis`, not over the network, so monitoring is unaffected.
- Break-glass when Authentik is down:
  - File Browser: `kubectl -n flux-system patch kustomization flux-system --type merge -p '{"spec":{"suspend":true}}'`, then `kubectl -n filebrowser scale deploy/filebrowser --replicas=0 && kubectl -n filebrowser wait --for=delete pod -l app.kubernetes.io/name=filebrowser --timeout=60s`. Run a one-off pod to flip the DB back to password auth: `kubectl -n filebrowser run fb-breakglass --rm -i --restart=Never --image=filebrowser/filebrowser:s6 --image-pull-policy=IfNotPresent --overrides='{"spec":{"nodeSelector":{"kubernetes.io/hostname":"minipc"},"containers":[{"name":"fb","image":"filebrowser/filebrowser:s6","command":["sh","-c","filebrowser -d /database/filebrowser.db config set --auth.method=json"],"volumeMounts":[{"name":"db","mountPath":"/database"}]}],"volumes":[{"name":"db","hostPath":{"path":"/srv/filebrowser/database","type":"Directory"}}]}}'`. Scale the Deployment back to 1, `kubectl -n filebrowser port-forward deploy/filebrowser 8080:80`, and sign in as `lamiphil` with the original admin password. Resume Flux (`"suspend":false`); the initContainer flips the DB back to `proxy` on the next pod start.
  - Praxis: `kubectl -n praxis port-forward deploy/praxis 8080:8080` and sign in with the password from `praxis-secret` at `http://localhost:8080/login`. The trusted header is only applied when it arrives in the request, so port-forwarded traffic falls back to the cookie session.

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
| `192.168.42.11:/srv/media` | NFS | File Browser, mounted with `subPath: Documents` so it only shows `/srv/media/Documents` |
| `/srv/media/photos` | hostPath PV (Retain) | Immich library, mounted at `/data`; uploads live in `photos/upload`. Never move it |
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
- Grafana also queries the Cloudflare GraphQL Analytics API through the `yesoreyeram-infinity-datasource` plugin (pinned to `3.11.1`, datasource UID `cloudflare-r2`). The bearer token is read from env var `CLOUDFLARE_API_TOKEN`, loaded by `envFromSecrets` from `grafana-cloudflare-r2` and substituted into `secureJsonData.httpHeaderValue1` at provisioning time.
- Dashboards are ConfigMaps labeled `grafana_dashboard=1`: `Homelab Cluster Health` (UID `homelab-cluster-health`), `Homelab Cluster Resources` (UID `homelab-cluster-resources`), `Praxis — Habitudes` (UID `praxis-habits`), and `Homelab R2 Backups` (UID `homelab-r2-backups`). The chart's bundled dashboards stay disabled (`grafana.defaultDashboardsEnabled=false`).
- `Homelab Cluster Resources` covers CPU, RAM, storage, network I/O, requests vs limits vs actual, pod restarts, and OOMKills. Grafana template variables `node` and `namespace` are multi-select with `All`. Node-exporter metrics are joined to `node_uname_info` via `on(instance) group_left(nodename)` so they filter by `$node`; cAdvisor and kube-state-metrics series already carry `namespace` and `node`. HostPath volumes do not appear in `kubelet_volume_stats_*`; the per-node filesystem panel covers `/srv` (minipc) and `/datavault` (datavault).
- In `Praxis — Habitudes`, an unrecorded day is a gap and breaks streaks. Streaks count from today if today is saved, otherwise from yesterday. Habits appear only on the days they were active. Sleep and coffee have separate averages and trend panels; they never enter habit completion or streak SQL.
- `Homelab R2 Backups` shows storage size, object count, and Class A/B operations per bucket for `praxis-backups`, `authentik-backups`, and `immich-backups`, plus backup freshness per CronJob from `kube_cronjob_status_last_successful_time` (green <28 h, yellow <36 h, red beyond). It also projects the monthly Cloudflare R2 storage bill from the latest `max.payloadSize` snapshot and the standard rate of $0.015 per GB-month, both per bucket (gross) and account-wide (gross plus net after the 10 GB-month free tier; negative values mean the account is still inside the free tier). Operation costs are not shown — Class A at $4.50/M and Class B at $0.36/M are negligible for backup traffic. The dashboard uses Infinity 3.11's backend parser, which does not evaluate JSONata arithmetic, so storage selectors are raw `max.payloadSize` paths (metadata overhead is <0.1%) and all math runs through Grafana `calculateField`/`groupBy`/`reduce` transformations. The Cloudflare account tag is stored as a hidden dashboard constant, not as a secret.

## Secrets Management

- SOPS + age encrypts files matching `*-secret.sops.yaml`.
- Flux decrypts secrets with the `sops-age` Secret in `flux-system`.
- Encrypted application secrets include `authentik/authentik-secret.sops.yaml`, `monitoring/kube-prometheus-stack/grafana-oidc-secret.sops.yaml`, `monitoring/kube-prometheus-stack/grafana-cloudflare-r2-secret.sops.yaml`, `authentik/authentik-smtp-secret.sops.yaml`, `immich/immich-db-secret.sops.yaml`, `immich/immich-backup-r2-secret.sops.yaml`, and `praxis/praxis-secret.sops.yaml`.
- `monitoring/kube-prometheus-stack/grafana-cloudflare-r2-secret.sops.yaml` holds a Cloudflare token scoped to **Account Analytics: Read** (via the "Read analytics & logs" template). It is used only by Grafana's Cloudflare R2 datasource and has no access to R2 bucket contents.
- `authentik/authentik-backup-r2-secret.sops.yaml` contains a token limited to the `authentik-backups` bucket.
- `cert-manager/cloudflare-api-token-secret.sops.yaml` holds a Cloudflare token limited to `Zone:DNS:Edit` and `Zone:Zone:Read` on `lamiphil.dev`.
- `praxis/praxis-secret.sops.yaml` contains the password/session secrets plus the VAPID public/private key pair used for Web Push. The public key is safe to expose to subscribed browsers; the private key must remain encrypted.

## Praxis tracking

- Praxis `v0.2.1` adds nullable day-level `sleep_minutes` and `coffee_count` fields. Sleep accepts quarter-hour values from 0 to 24 hours and belongs to the wake date; coffee counts caffeinated coffee drinks from 0 to 20. Blank means unknown and differs from zero.
- These fields are measurements, not habits. They do not affect habit completion, streaks, or calendar rules. The former binary `Sommeil` habit was archived with its existing history intact.
- The in-app `Tendances` page shows the last 30 calendar days, averages over recorded values only, a 7–9 hour sleep target, and a maximum of 3 coffees.
- Each browser can opt into one Web Push reminder after 20:00 America/Toronto when either measurement is blank. Subscriptions and once-per-day delivery state live in the Praxis SQLite database and are included in backups.
- The schema migration is additive. `v0.1.3` can open the upgraded database and ignores the new nullable columns/tables. For break-glass rollback, pin the old image and manually reactivate the `Sommeil` habit; do not restore the database unless the new data itself must be discarded.

## Praxis backups

- CronJob `praxis/praxis-backup` runs nightly at 03:30 America/Toronto. It takes a `sqlite3 .backup` snapshot, checks `PRAGMA integrity_check`, and uploads `praxis-<UTC timestamp>.db` to the Cloudflare R2 bucket `praxis-backups`. The snapshot includes habits, sleep/coffee measurements, push subscriptions, and reminder delivery state.
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

## Immich backups

- CronJob `immich/immich-backup` runs nightly at 03:00 America/Toronto on `minipc` (the only node that sees the library hostPath).
- It dumps the Immich PostgreSQL database with `pg_dump --format=custom`, then runs `rclone sync /data/upload r2:immich-backups/originals` and uploads the dump to `r2:immich-backups/db/immich-<UTC>.dump`.
- `rclone sync` uses `--backup-dir r2:immich-backups/trash/<date>` with a timestamp `--suffix`: any file deleted or overwritten on the source is moved under `trash/<date>/` instead of being destroyed. Add an R2 lifecycle rule `expire-trash-30d` on prefix `trash/` with `Delete uploaded objects: 30 days` to get a rolling 30-day recycle bin.
- The `immich-backups` R2 bucket is scoped to a dedicated Object Read & Write token in `immich/immich-backup-r2-secret.sops.yaml`.
- A failed run fires `KubeJobFailed`. To run a backup now: `kubectl -n immich create job immich-backup-manual --from=cronjob/immich-backup`.

Restore:

1. Download the matching `db/immich-<ts>.dump` and the needed originals from R2 (or sync the whole `originals/` prefix back to a scratch directory).
2. Suspend Flux (`kubectl -n flux-system patch kustomization flux-system --type merge -p '{"spec":{"suspend":true}}'`) and scale `deploy/immich-server` and `deploy/immich-machine-learning` to zero.
3. Copy the dump into the PostgreSQL pod and restore it with `pg_restore --clean --if-exists --no-owner --single-transaction --exit-on-error --username=immich --dbname=immich`.
4. On `minipc`, stage the originals under `/srv/media/photos/upload/` preserving ownership (`chown -R 1000:1000`).
5. Resume Flux, wait for Immich to come back, and verify a photo opens and faces/albums are present before removing the scratch copy.

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
