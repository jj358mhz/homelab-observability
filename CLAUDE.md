# CLAUDE.md — homelab-observability

## Overview

**Purpose:** Shared source of truth for containerized observability stacks
deployed across Docker-based hosts in the homelab fleet via Portainer GitOps.

Currently ships two stacks:
- **`node-exporter/`** — host-level metrics (CPU, memory, disk, network, thermals,
  SMART data via textfile collector) on port `9100`
- **`cadvisor/`** — per-container metrics (CPU, memory, network, filesystem
  usage per container) on port `8082`

Non-goals:
- **Not** a general "everything monitoring" repo. Prometheus and Grafana live in
  [`homelab-utility`](https://github.com/jj358mhz/homelab-utility) alongside the
  rest of the utility Pi's stacks
- **Not** for bare-metal hosts (Piholes, BFD). Those run the Debian
  `prometheus-node-exporter` package directly, managed by the
  [`ansible-homelab`](https://github.com/jj358mhz/ansible-homelab) `common` role
- **Not** for the Scanner-side Loki+Grafana stack (that's a separate island
  scoped to the public-facing bayareascanner.com box)

---

## 🏗️ Architecture

**Deployment model:** Portainer on utility (`192.168.1.248`) pulls this repo
periodically (GitOps polling, 5 min interval) and redeploys the corresponding
stacks on each Docker host via its Edge Agent. Each Docker host runs both
stacks as independent Portainer stacks.

**Prometheus** (also on utility) scrapes each host's `:9100` (node-exporter)
and `:8082` (cadvisor) endpoints across the LAN. Scrape targets are
statically listed in `homelab-utility/monitoring/prometheus/config/prometheus.yml`.

**Grafana** (also on utility) reads Prometheus and renders the `Pi Host Health`
dashboard, which uses a `$host` variable selector to switch between fleet
members.

## Fleet deployment status

| Host                | Model      | Storage      | node-exporter | cAdvisor |
|---------------------|------------|--------------|---------------|----------|
| raspberrypi-adsb    | Pi 4B      | SATA SSD     | deployed      | deployed |
| raspberrypi-ntp     | Pi 5       | NVMe         | deployed      | deployed |
| raspberrypi-scanner | Pi 5       | NVMe, public | deployed      | deployed |

Bare-metal hosts (Piholes, BFD, utility) don't run these stacks — they use
the Debian `prometheus-node-exporter` package via the `common` ansible role.

---

## 🚀 Adding a new Docker host to the fleet

1. **Baseline the host** via ansible:
```bash
   cd ~/ansible-homelab
   ansible-playbook playbooks/baseline.yml --limit <ip>
```
   Set `common_mask_debian_node_exporter: true` in `host_vars/<ip>.yml` so
   the Debian exporter package (pulled in as a dependency of collectors)
   gets masked and doesn't fight for port 9100.

2. **Deploy the two stacks** in Portainer:
   - Environment: the new host's endpoint
   - Repository: `https://github.com/jj358mhz/homelab-observability`
   - Compose paths: `node-exporter/docker-compose.yml` and
     `cadvisor/docker-compose.yml`
   - GitOps updates: **ON**, 5 min polling
   - Environment variables: override `HOST_BIND_ADDR` to the LAN IP if the
     host is public-facing (see Security notes)

3. **Open the firewall** (only if cross-VLAN):
   - UniFi zone-based policy: utility → host, TCP/9100 and TCP/8082
   - Or a `ufw` rule on the host itself if host-level firewall is active

4. **Add to Prometheus** in `homelab-utility/monitoring/prometheus/config/prometheus.yml`:
```yaml
   - targets: ['<ip>:9100']
     labels:
       host: <hostname>
```
   Same for the cadvisor job on `:8082`. Commit + push, Portainer's GitOps
   redeploys Prometheus.

5. **Verify in Grafana** — Host dropdown should now include the new host,
   Docker Containers row should populate.

---

## ✅ Design principles

- **One stack, one job** — node-exporter and cAdvisor are separate stacks
  so they can be added, removed, or updated independently per host
- **Parametrized bind addresses** — `HOST_BIND_ADDR` env var (defaults to
  `0.0.0.0`) lets public-facing hosts restrict published ports to LAN
  interfaces without repo forks
- **GitOps is the source of truth** — never edit compose files on hosts
  directly. Edit in this repo, push, Portainer redeploys
- **Textfile collector requires ansible prereqs** — node-exporter reads
  `smartmon.prom` and `smartmon-nvme.prom` from `/var/lib/prometheus/node-exporter/`,
  populated by the ansible common role's smartmon timers (SATA + NVMe)
- **Consistent naming** — container names, hostnames, and stack names all
  match (`node-exporter`, `cadvisor`) for easy correlation in logs

---

## ⚠️ Notable details

### The Debian `prometheus-node-exporter` trap

When the ansible common role installs `prometheus-node-exporter-collectors`
(needed for the smartmon shell script), Debian auto-installs
`prometheus-node-exporter` as a hard dependency, auto-starts it, and enables
it on boot. On Docker hosts running the containerized exporter, this creates
a port 9100 conflict.

Mitigation: set `common_mask_debian_node_exporter: true` in the host's
`host_vars/<ip>.yml`. The role will stop and mask the Debian service.

### Public-facing hosts (Scanner)

Scanner runs bayareascanner.com. The node-exporter and cadvisor stacks are
deployed with `HOST_BIND_ADDR=192.168.70.200` (the LAN IP) so the published
ports never bind to the public interface.

Scanner also runs `ufw` with a strict allowlist (22, 80, 443, 3000). The
:9100 and :8082 ports work across VLANs because Docker's iptables rules
bypass ufw for published ports. A UniFi zone-based firewall policy allows
utility → scanner on those specific ports.

### NVMe SMART on Pi 5

Pi 5 kernel boot params include `cgroup_disable=memory` and other firmware
tuning that would seem to break memory monitoring — but Pi 5 uses cgroup v2
where that flag is a no-op. cAdvisor reads memory correctly via the unified
hierarchy.

The stock Debian smartmon collector script only handles SATA drives. A
custom NVMe collector (`smartmon-nvme.sh`) deployed by the ansible common
role writes NVMe SMART data in the same `smartmon_*` metric format, so
Grafana panels work uniformly across drive types.

---

## 🔄 Updating a stack

```bash
# Edit the compose file
$EDITOR node-exporter/docker-compose.yml

# Commit and push — Portainer picks it up on next poll
git add node-exporter/docker-compose.yml
git commit -m "node-exporter: <what changed>"
git push
```

Portainer GitOps polls every 5 min by default. To skip the wait, force a
redeploy in the Portainer UI on each host's stack.

---

## 📚 Related repos

- [`ansible-homelab`](https://github.com/jj358mhz/ansible-homelab) — Fleet
  baseline: installs smartmontools, collectors, smartmon-nvme script and
  timer, and masks the Debian exporter on Docker hosts (`common` role)
- [`homelab-utility`](https://github.com/jj358mhz/homelab-utility) —
  Prometheus config (`monitoring/prometheus/config/prometheus.yml`),
  Grafana stack, and Portainer itself
- [`adsb`](https://github.com/jj358mhz/adsb) — ADS-B receiver stack;
  formerly hosted its own monitoring compose file, now migrated here

---

*Last updated: 2026-08-12*
