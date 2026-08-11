# homelab-observability

Observability stack components deployed to fleet hosts via Portainer GitOps.

## Stacks

### `node-exporter/`

Prometheus node-exporter for Docker-based hosts. Publishes host metrics
(CPU, memory, disk, network, thermals) plus SMART data via the textfile
collector on `:9100/metrics`.

**Prerequisites:** The `common` role in
[ansible-homelab](https://github.com/jj358mhz/ansible-homelab) must be
applied first. It installs `smartmontools` and
`prometheus-node-exporter-collectors`, and enables the smartmon timer
that writes `smartmon.prom` to `/var/lib/prometheus/node-exporter/`.

### `cadvisor/`

Google cAdvisor for per-container metrics (CPU, memory, network, filesystem
usage per container) on `:8082/metrics`. Deployed alongside node-exporter
on hosts where container-level visibility is wanted.

**Prerequisites:** Docker running on the host. No `common` role setup required.

## Deployment status

**node-exporter:**
- `raspberrypi-ntp` (192.168.123.123) — Pi 5, NVMe — deployed
- `raspberrypi-scanner` (192.168.70.200) — Pi 5, NVMe, public-facing — deployed
- `raspberrypi-adsb` (192.168.1.249) — Pi 4B, SATA SSD — migration pending (currently in `git/adsb/monitoring/`)

**cadvisor:**
- `raspberrypi-ntp` (192.168.123.123) — Pi 5, NVMe — pending
- `raspberrypi-adsb` (192.168.1.249) — Pi 4B, SATA SSD — migration pending (currently in `git/adsb/monitoring/`)
- `raspberrypi-scanner` (192.168.70.200) — Pi 5, NVMe, public-facing — deployed

## Security notes

- On public-facing hosts, override `HOST_BIND_ADDR` to the internal LAN IP
  (e.g. `192.168.70.200`) via a Portainer stack env var. Prevents published
  ports from being reachable via the public interface.
- Never install the Debian `prometheus-node-exporter` package on hosts
  running the node-exporter stack — port 9100 conflict. The `common` role's
  `common_mask_debian_node_exporter: true` flag prevents this automatically
  on container-exporter hosts.
