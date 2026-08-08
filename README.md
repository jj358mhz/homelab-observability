# homelab-observability

Observability stack components deployed to fleet hosts via Portainer GitOps.

## Stacks

### `node-exporter/`

Prometheus node-exporter for Docker-based hosts. Publishes host metrics
(CPU, memory, disk, network, thermals) plus SMART data via the textfile
collector to `:9100/metrics`.

**Deployment targets:**
- `raspberrypi-ntp` (192.168.123.123) — Pi 5, NVMe
- `raspberrypi-scanner` (192.168.70.200) — Pi 5, NVMe, public-facing
- `raspberrypi-adsb` (192.168.1.249) — Pi 4B, SATA SSD (migration pending)

**Prerequisites on each host:**
The `common` role in [ansible-homelab](https://github.com/jj358mhz/ansible-homelab)
must be applied first. It installs `smartmontools` and
`prometheus-node-exporter-collectors`, and enables the smartmon timer
that writes `smartmon.prom` to `/var/lib/prometheus/node-exporter/`.

**Security notes:**
- On public-facing hosts, bind the published port to the internal LAN
  interface only (e.g. `192.168.70.200:9100:9100`) to avoid exposing
  system metrics to the internet.
- Never install the Debian `prometheus-node-exporter` package on hosts
  running this stack — port 9100 conflict.
