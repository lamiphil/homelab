# Homelab

GitOps-powered home Kubernetes cluster built with k3s, Flux CD, SOPS/age, MetalLB, and UniFi networking.

## Highlights

- **Network edge**: UniFi UDM, WireGuard VPN, Cloudflare DDNS
- **Main VLANs**:
  - home/client VLAN: `192.168.0.0/24`
  - homelab VLAN: `192.168.42.0/24`
  - WireGuard VPN: `192.168.69.0/24`
- **Cluster nodes**:
  - `magos` — single control plane and schedulable worker
  - `minipc` — worker for Immich and File Browser
  - `datavault` — worker and storage node
- **Applications**: Authentik, Homepage, Immich, File Browser, Praxis, Grafana, Prometheus, Alertmanager
- **Service exposure**: Traefik at `192.168.42.60`, MetalLB pool `192.168.42.60-192.168.42.100`

`ubuntuserver`, `pi`, and `archlinux` were retired on 2026-11-15.
