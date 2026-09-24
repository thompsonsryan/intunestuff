# OpenWrt Prometheus / Grafana dashboard

Import `grafana/openwrt-router-dashboard.json` into Grafana and select the Prometheus datasource that scrapes `prometheus-node-exporter-lua`.

## Expected Prometheus job

The dashboard expects a Prometheus scrape job named `openwrt`, for example:

```yaml
scrape_configs:
  - job_name: openwrt
    static_configs:
      - targets:
          - 192.168.1.1:9100
```

The dashboard prompts for the Prometheus datasource during import. Its router variable is populated from `up{job="openwrt"}`.

Defaults:
- WAN interface: `eth1`
- LAN interface: `eth0`
- Refresh: 15 seconds
- Initial time range: 24 hours

Included sections:
- Router overview: uptime, CPU, RAM, temperature, conntrack usage, Unbound cache-hit rate
- Network: WAN/LAN throughput, errors/drops, conntrack entries
- DNS / Unbound: query rate, cache hits/misses, prefetch, recursion latency, expired/stale responses, timeouts/errors, request-list activity
- System: CPU, memory, temperature
- Firewall: nftables named counter packet/byte rates

Required OpenWrt collectors for the full dashboard:
- base prometheus-node-exporter-lua collectors
- unbound
- thermal
- nft-counters

The dashboard still imports if optional nftables counters are not present; those panels will simply be empty.

This dashboard was prepared for Grafana 13.2.x and Prometheus 3.x.

## Important label note

Do not use a target label named `device` on the OpenWrt scrape target. The OpenWrt exporter already uses `device` for interface names such as `eth0`, `eth1`, and `br-lan`. A static `device` label causes Prometheus to preserve the exporter's original interface label as `exported_device`, which breaks the dashboard's interface queries. Use a label such as `router: qotom-openwrt` instead.
