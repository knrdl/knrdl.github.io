---
title: "Conceptualize Docker Networks to be minimal"
date: "2022-09-10"
tags: ["docker"]
lang: "en"
---

# Introduction

There are three kinds of private ip ranges:

| Class | CIDR           | Last IP         | IPs        | Typical Usage         |
|-------|----------------|-----------------|------------|-----------------------|
| A     | 10.0.0.0/8     | 10.255.255.255  | 16,777,216 | Big Company Network   |
| B     | 172.16.0.0/12  | 172.31.255.255  | 1,048,576  | ***Docker Network!*** |
| C     | 192.168.0.0/16 | 192.168.255.255 | 65,536     | Home Network          |

So there are about a million IPs available for docker containers in docker networks. However, docker per default splits them up as /24 CIDRs. Therefore, every docker network can include up to
<math>
<msup>
<mi>2</mi>
<mrow>
(<mn>32</mn><mo>-</mo><mn>24</mn>)
</mrow>
</msup>
<mo>-</mo>
<mn>2</mn>
<mo>=</mo>
<mn>254</mn>
</math>
Container-IPs. The total number of docker networks is limited to
<math>
<msup>
<mi>2</mi>
<mrow>
(<mn>24</mn><mo>-</mo><mn>12</mn>)
</mrow>
</msup>
<mo>=</mo>
<mn>4096</mn>
</math>.

While these defaults are okay, it's possible to run out of address spaces, **especially if private Class B addresses are used elsewhere** (by other applications/routing). Using more docker networks with fewer containers per network has two benefits: Security and Safety.

# Security & Safety

1. Scenario with three containers: gateway, appserver, database.

   It's easy to put all three of them in a single docker network. But this would allow the gateway to access the database, which is not required. Therefore, 2 docker networks should be utilized.

2. Scenario with three containers: gateway, appserver1, appserver2.

   Using a single network to connect the appservers to the gateway, allows appserver1 to talk to appserver2. This might not be desired, if app1 and app2 are two distinct applications.

The security gain is a better separation of trust levels, following the principle of least privilege. Anyway, an advanced attacker can still try OSI-Layer 2 ("MAC-Level") sniffing attacks.

The bigger gain is the safety of the container environment. The impact of configuration mistakes is limited. The architecture is clearer: Confusion about what service the hostname "app" belongs to is hopefully prevented.

The cost of this approach is that there might be more docker networks needed than the 4096 possible ones. A config change allows to create more networks.

# Configuration

In `/etc/docker/daemon.json` add:

```json
{
  "default-address-pools": [
    {
      "base": "172.16.0.0/12",
      "size": 27
    }
  ]
}
```

This will use the complete Class B private namespace (`172.16.0.0/12`). The CIDR per docker network is /27.

Max docker networks per host:
<math>
<msup>
<mi>2</mi>
<mrow>
(<mn>27</mn><mo>-</mo><mn>12</mn>)
</mrow>
</msup>
<mo>=</mo>
<mn>32,768</mn>
</math>

Max containers per docker network: 
<math>
<msup>
<mi>2</mi>
<mrow>
(<mn>32</mn><mo>-</mo><mn>27</mn>)
</mrow>
</msup>
<mo>-</mo>
<mn>2</mn>
<mo>=</mo>
<mn>30</mn>
</math>
