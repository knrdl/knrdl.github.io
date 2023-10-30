---
title: "InfluxDB v2: Single Sign On"
date: "2023-10-30"
tags: [influxdb, devops]
lang: "en"
hackerNewsId: ""
---

The <abbr title="Telegraf, InfluxDB, Chronograf, Kapacitor">TICK</abbr> stack v1 did not enforce authentication. So SSO for the Chronograf WebUI could be handled easily via the reverse proxy. InfluxDB v2 includes a WebUI on its own and therefore replaces Chronograf. As InfluxDB v2 enforces built-in authentication there must be accounts. But InfluxDB v2 has no <abbr title="Single Sign On">SSO</abbr> capabilities. The solution is to create an "All Access API Token" and inject it via the reverse proxy, e.g. Caddy:

```
monitor.example.org {
    reverse_proxy influxdb:8086 {
        header_up Authorization "Token YOUR_TOKEN"
    }
}
```

Now the WebUI does not require built-in authentication anymore and it could be handled by the reverse proxy again.

The other always working, obvious solution is to just use Grafana instead.
