# Homelab

GitOps-powered home Kubernetes cluster built with k3s, Flux CD, SOPS/age, MetalLB, and UniFi networking.

## Highlights

- **Network edge**: UniFi UDM, WireGuard VPN, Cloudflare DDNS
- **Main VLANs**:
  - home/client VLAN: `192.168.0.0/24`
  - homelab VLAN: `192.168.42.0/24`
  - WireGuard VPN: `192.168.69.0/24`
- **Target cluster nodes**:
  - `magos` — control plane and schedulable worker
  - `minipc` — Immich and File Browser
  - `datavault` — worker and storage workloads
- **Applications**: Homepage, Immich, File Browser, Grafana, Prometheus, Alertmanager
- **Service exposure**: Traefik at `192.168.42.60`, MetalLB pool `192.168.42.60-192.168.42.100`

The migration from `ubuntuserver`, `pi`, and `archlinux` is in progress.
