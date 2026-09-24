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


## Advanced OpenWrt metrics

The dashboard also supports QoSify, dnscrypt-proxy/X-Wing, Chrony/NTS and banIP metrics through the OpenWrt textfile collector.

Install the collector on OpenWrt:

```sh
wget -O /usr/bin/router-extra-metrics \
  https://raw.githubusercontent.com/thompsonsryan/intunestuff/main/openwrt/router-extra-metrics
chmod +x /usr/bin/router-extra-metrics

wget -O /etc/init.d/router-extra-metrics \
  https://raw.githubusercontent.com/thompsonsryan/intunestuff/main/openwrt/router-extra-metrics.init
chmod +x /etc/init.d/router-extra-metrics

/etc/init.d/router-extra-metrics enable
/etc/init.d/router-extra-metrics restart
sleep 20
```

Verify:

```sh
wget -qO- http://127.0.0.1:9100/metrics | \
grep -E '^(openwrt_qosify_|dnscrypt_proxy_|openwrt_dnscrypt_|openwrt_chrony_|openwrt_banip_)' | \
head -100
```

The collector writes atomically to `/var/prometheus/router-extra.prom` every 15 seconds. It uses only BusyBox shell tools, `jsonfilter`, existing OpenWrt services, and the dnscrypt-proxy local metrics endpoint.

Chrony NTS state is read with `chronyc -N authdata`. QoSify counters are read from `ubus call qosify get_stats`. Detailed banIP packet counts are taken from the existing nftables Prometheus collector.


### CAKE qdisc metrics

The extra collector also parses the WAN CAKE qdisc from `tc -s qdisc show dev eth1` by default. Override the interface with the `ROUTER_EXTRA_CAKE_IFACE` environment variable if needed.

Exported CAKE metrics include configured bandwidth, capacity estimate, total packets/bytes/drops/overlimits/requeues, backlog, memory usage, active queues, and per-diffserv4-tin threshold/delay/backlog/packet/drop/ECN/flow statistics.

The dashboard uses these metrics in the **CAKE / WAN egress** section. CAKE `overlimits` are shaping activity and are not equivalent to packet drops.

For this router the default WAN interface is `eth1`.
