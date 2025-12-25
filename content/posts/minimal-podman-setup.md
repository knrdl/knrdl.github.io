---
title: "Minimal setup for a rootless Podman host"
date: "2025-12-25"
tags: [podman, docker, ubuntu, debian]
lang: "en"
---

# Install Podman

```shell
sudo apt install podman podman-compose
```

Compose can be called as `podman-compose` with autocomplete.

## Allow binding to privileged ports (<1024)

```shell
echo "net.ipv4.ip_unprivileged_port_start=1" > /etc/sysctl.d/100-podman.conf
/usr/sbin/sysctl --system  # apply setting
```

## Start containers after reboot without user login

```shell
loginctl enable-linger $USER
```

## Pull images from Docker Hub without explicit `docker.io` prefix

Add to `~/.config/containers/registries.conf`:

```
unqualified-search-registries = ["docker.io"]
```

# Useful shortcuts

Add to `~/.bashrc`:

```shell
alias deploy='podman-compose --podman-run-args=--replace up -d --pull --build --remove-orphans'
alias undeploy='podman-compose down --remove-orphans'
alias logs='podman-compose logs --follow --names'
alias stack='podman-compose'
```

Test in a new terminal or run `source ~/.bashrc`



# Housekeeping

/etc/crontab:

```
0    1  * * *	root	     apt-get -y update && apt-get autoremove --purge -y
```

User-specific `crontab -e`:

```
0   3  *   *   *     /home/container/update_all.sh
0   5  *   *   *     podman system prune --force
*/3 *  *   *   *     podman ps --filter health=unhealthy --format "podman restart {{.ID}}" | bash
```

## Stack updater script `/home/container/update_all.sh`:

```shell
#!/bin/bash

cd /home/container

for d in */; do
    cd "$d"
    echo "$d"
    podman-compose --podman-run-args=--replace up -d --pull --build --remove-orphans
    cd ..
done
```

## Auto-Updater

Alternatively, Podman's built-in auto-updater can be enabled with:

```shell
sudo systemctl enable --now podman-auto-update.timer
podman auto-update --dry-run  # test
```

Containers needs to be marked as updatable via label.
However, it might not be sufficient to rebuild compose stacks that use `build:` instead of `image:` for services.

# Test

```shell
podman run -it --rm -p80:80 nginx:alpine
```

## Portainer

```shell
systemctl --user enable --now podman.socket
podman run -it --rm -p 9443:9443 --name portainer -v /run/user/1000/podman/podman.sock:/var/run/docker.sock portainer/portainer-ce
```