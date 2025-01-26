---
title: "The poor man's WAF"
date: "2025-01-26"
tags: [caddy, ipfilter, waf]
lang: "en"
---

Every internet connected server is subject to automated scans and probes from various parties. On web servers they typically try to find exploitable installations by calling e.g. `///wordpress/wp-admin/setup-config.php?step=1`. Another annoyance are web scrapers farming content for AI bot training and often enough ignoring therefore the robots.txt. These requests form a constant background noise which we don't have to listen to. If you run a small home server only for friends and family you can configure the internet-facing reverse proxy to just drop these requests. I will use the excellent [Caddy](https://caddyserver.com/) reverse proxy to build a poor man's <abbr title="Web Application Firewall">WAF</abbr>.

# IP filter

Some companies publish theirs IP address ranges used by their products, e.g. [AWS](https://ip-ranges.amazonaws.com/ip-ranges.json), [Cloudflare](https://api.cloudflare.com/client/v4/ips), [GitHub](https://api.github.com/meta), [Googlebot](https://developers.google.com/search/apis/ipranges/googlebot.json), [Google Cloud](https://www.gstatic.com/ipranges/cloud.json) or OpenAI [1](https://openai.com/searchbot.json) [2](https://openai.com/chatgpt-user.json) [3](https://openai.com/gptbot.json). However, we only need to allow connections from residential or mobile IP addresses so we can block requests from these companies at all. Instead of relying on them to publish relevant IP ranges, we can block all their IPs. This in turn also includes cloud instances which might be used by AI companies for their scraping.

Every big internet company acts as an AS (autonomous system) and has an ASN (autonomous system number) assigned. Every AS manages their assigned IP ranges. So we can block IP ranges based on their AS. The assignment database can be grabbed from [here](https://ip.guide/bulk/networks.csv) among other places. This python script extracts the IP ranges for relevant companies:

```python
import csv, fnmatch
from ipaddress import ip_network, IPv4Network, IPv6Network, collapse_addresses

block_filters = {
    'DigitalOcean*',
    'Amazon.com*',
    'Google*',
    'Cloudflare*',
    'Facebook*', 
    'Microsoft*', 
    'GitHub*', 
    'Twitter*', 
    'Hetzner*', 
    'IBM*', 
    'Akamai*', 
    'Fly.io*', 
    'Apple Inc.', 
    'SAP *', 
    'Oracle Corporation', 
    'Yahoo Inc.', 
    'Yahoo-UK Limited', 
    'Opera Norway AS',  # Opera VPN
    'Clouvider *',
    'PacketHub *'  # NordVPN
}

output: set[IPv4Network | IPv6Network] = set()

# https://ip.guide/bulk/networks.csv
with open('networks.csv') as f:
    reader = csv.reader(f)
    next(reader, None)  # skip the headers
    for network, asn, organization, country in reader:
        if any((fnmatch.fnmatchcase(organization, bf) for bf in block_filters)):
            output.add(ip_network(network))

with open('blocklist.txt', 'w') as f:         
    f.write(
        ' '.join(
            [n.with_prefixlen for n in sorted(collapse_addresses([n for n in output if n.version == 4]))] +
            [n.with_prefixlen for n in sorted(collapse_addresses([n for n in output if n.version == 6]))]
        )
    )

```
There are round about 54&thinsp;000 matching IP ranges in the database. By using `collapse_addresses` these are unified into less than 11&thinsp;000 IP ranges which can be added to the Caddy config:

```ini
@ipfilter remote_ip 1.0.0.0/24 1.1.1.0/24 1.44.96.0/24 ...

handle @ipfilter {
    abort  # the request will be dropped by closing the connection
}
```

# User Agent filter

UA filtering is kind of snake oil as real attackers and stealthy crawlers won't bring their own UA. But at least we can release Internet Explorer users from their suffering:

```ini
@uafilter `
    header_regexp('User-Agent', 'MSIE') ||
    header_regexp('User-Agent', 'Trident')
`
handle @uafilter {
    abort
}
```

You should never block UAs of tools like `curl` because at some point you might need them for debugging.


# Path filter

As long as you don't run a PHP server (which you shouldn't) there is no need in answering PHP related requests:

```ini
@pathfilter `
    path_regexp('(?i)\\.php$')
`
handle @pathfilter {
    abort
}
```

# Method filter

Modern web applications and REST APIs use a standard set of HTTP methods. Requests with deviant methods can be blocked by returning HTTP status code 405.

```ini
@methodfilter `
    !method('GET', 'HEAD', 'OPTIONS', 'POST', 'PUT', 'PATCH', 'DELETE')
`
handle @methodfilter {
    respond 405 {
        body "method not allowed"
	}
}
```

> Keep in mind that some protocols built on top of HTTP might use different methods. Especially CardDAV, CalDAV and WebDAV bring in a bunch of custom HTTP methods!
> {.warning }