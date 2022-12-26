---
title: "Ubuntu: configure custom dns resolver"
date: "2022-12-22"
tags: [ubuntu, netplan, dns]
lang: "en"
---

DNS provisioning mostly works out of the box. But once for a cloud instance the DNS resolver had terrible response times. So I added another resolver.

Ubuntu uses `netplan` as abstraction layer for network config.

`sudo nano /etc/netplan/01-netcfg.yaml`

```yaml
...
nameservers:
  search: [ invalid ]
  addresses:
    - 127.0.0.1  # the new resolver
    - 123.123.123.1  # provided dns resolver 
    - 223.123.123.2  # provided dns resolver
```

For a containerized DNS resolver see [here](https://github.com/knrdl/unbound-dns-server).

> DNS resolvers should NOT be exposed to public networks (dns amplification attacks etc.)! So make sure to bind the dns server socket to localhost or the local network.
> {.warning }

Finally, run: `sudo netplan apply`

Check which DNS resolver is currently in use: `sudo resolvectl status eth0` (filter for network interface `eth0` is optional)
