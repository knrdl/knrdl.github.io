---
title: "Docker Registry Mirror"
date: "2022-09-21"
tags: [docker, kaniko, podman]
lang: "en"
hackerNewsId: ""
---

# Concept

A docker-registry stores docker-images, composed of:
* **metadata**: image names and tags
* **blobs**: actual image contents

The most famous public docker-registry is [DockerHub](https://hub.docker.com). DockerHub applies a rate-limiting for downloading blobs. The fetching of metadata is not sanctioned. Therefore, a local docker-registry mirror can be used to circumvent DockerHub's rate-limiting. This might also reduce the bandwidth usage of your ISP connection.

For metadata retrieval the docker-registry mirror will serve as a simple proxy server to the upstream (e.g. DockerHub). If you retrieve a docker-image via the mirror the blobs are stored locally by the mirror. That way the mirror can serve as a cache for further requests. For example, when multiple servers deploy the same docker-image, only the first request will be a cache-miss. As the metadata records are always fetched from upstream, there is no risk of serving outdated docker-images.

# Setup

## Server

The registry mirror is a simple docker container:

```yaml
version: '3.9'

services:
  registry-mirror:
    image: registry:2
    hostname: registry-mirror
    environment:
      REGISTRY_PROXY_REMOTEURL: https://registry-1.docker.io  # Mirror DockerHub
    networks:
      - reverse_proxy_net  # just an example
    deploy:
      resources:
        reservations:
          memory: 16m
        limits:
          memory: 250m
```

It should always be exposed via HTTPS to the clients, by a tls-terminating reverse-proxy. The registry-mirror runs http on port 5000.

Cached images can be listed via `curl https://registry-mirror.example.org/v2/_catalog`.

## Clients

### Docker

> On the server which should execute the registry-mirror run `docker pull registry:2` first, to prevent the chicken or egg problem.
> {.warning }

To make Docker use the registry mirror, add to `/etc/docker/daemon.json`:

```json
{
  "registry-mirrors": [
    "https://registry-mirror.example.org"
  ]
}
```

Then restart the docker daemon: `sudo systemctl restart docker.service`

### Podman

Add to `$HOME/.config/containers/registries.conf`:

```ini
[[registry.mirror]]
location = "registry-mirror.example.org"
```

### Kaniko

To make Kaniko use the mirror, run it with the flag:

```shell
/kaniko/executor --registry-mirror=registry-mirror.example.org ...
```

# Operations

The registry mirror might be part of the critical path for high availability. Make sure all hosts are provided with their docker-images before shutting it down for maintenance etc.

There is no authentication in place, anybody with access can download arbitrary images. Therefore, the registry mirror should only be exposed to the server's network segment.

There is no storage limit per default and old blobs will not be pruned automatically. An attacker might crash the server by querying too many images. As countermeasure a storage quota should be applied. Also restarting the mirror container from time to time (e.g. patch-day server reboots) helps to reduce the storage usage.
