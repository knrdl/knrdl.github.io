---
title: "Docker Swarm: `error creating vxlan interface: file exists`"
date: "2022-04-10"
tags: ['docker-swarm', 'docker', 'devops']
lang: "en"
---

If docker swarm rejects to deploy a service because network interface already exists:

```shell
$ docker service ps stackname_appname --no-trunc

Rejected 34 seconds ago   "network sandbox join failed: subnet sandbox join failed for "10.0.14.0/24": error creating vxlan interface: file exists
```

Then find all problematic interfaces on the host and delete them:

```shell
$ ip -d link show | grep vx | grep DOWN
$ sudo ip link delete vx-001095-owhr8  # for each entry
```

Now redeploying should work!
