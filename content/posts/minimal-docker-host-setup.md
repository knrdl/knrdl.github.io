---
title: "Minimal setup for a docker host"
date: "2022-11-01"
tags: [docker, devops, ubuntu]
lang: "en"
---

# Install docker

Comes with Docker Compose as `$ docker compose`

```shell
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
     $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Check docker is running: `$ docker version`

# Enable swap

If not enabled yet

```shell
fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
sudo swapon --show
```

Add to `/etc/fstab`:

```shell
/swapfile swap swap defaults 0 0
```

Optional reboot to check if it worked

# Enable fail2ban

Necessary if ssh is accessible over the internet

```shell
sudo apt install fail2ban
sudo systemctl enable --now fail2ban
```

Statistics of failed attempts:

```shell
awk '($(NF-1) = /Ban/){print $NF}' /var/log/fail2ban.log | sort | uniq -c | sort -n
```

try to log in with wrong passwords to check banning is working!

# Useful shortcuts

Add to `~/.bashrc`:

```
alias dc='docker compose'
alias dclf='docker compose logs --follow'
alias dcup='docker compose up --detach --remove-orphans --build'
```

Test in new terminal or run `source ~/.bashrc`

# Docker Hardening

Disable inter container communication (ICC) via `/etc/docker/daemon.json`, add:

```json
{
  "icc": false
}
```

Might also want to justify address pool, see [other post](./minimal-docker-networks).

Don't forget to apply changes: `$ sudo systemctl restart docker.service`

# Docker Housekeeping

/etc/crontab:

```
0    1  * * *	root	apt-get -y update
0    3  * * *   root    /apps/update_all.sh
0    5  * * *   root    docker system prune --force
*/3  *  * * *   root    /apps/restart_unhealthy.sh
```

Cronjob commands in detail:

### 1. `apt-get -y update`

Will install safe package updates.
> Be aware that upgrades (`apt-get upgrade`) must be still done manually from time to time.
> {.warning }

### 2. `/apps/update_all.sh`

Update all container images (might not be desired):

```shell
#!/bin/bash

cd /apps

for d in */; do
    cd "$d"
    echo "$d"
    docker compose pull  # pull new images
    docker compose build --pull  # pull new base images and build new images
    docker compose up -d --remove-orphans  # start new containers from new images
    cd ..
done
```

### 3. `docker system prune --force`

Free disk space by cleaning up old docker entities, mostly stopped containers and images.

> `docker system prune --force` might take some time. do not run container updates meanwhile as it could confuse docker (problems with port allocations)
> {.warning }

### 4. `/apps/restart_unhealthy.sh`

Docker detects failing health checks but will happily ignore those. Docker Swarm (or Kubernetes) on the other hand will restart unhealthy containers. That missing feature can be corrected with an [extra container](https://github.com/willfarrell/docker-autoheal) or a really simple script:

```shell
#!/bin/bash
docker ps --filter health=unhealthy --format "docker restart {{.ID}}" | bash
```

# General advices

* Add a `mem_limit` for every container, don't forget it!
* If a container has no need to connect to the world (internet or local network) then make all attached Docker Networks "internal".
* If addressing a container in the network via it's name fails then set the `hostname` explicitly.
* If volumes map to directories on the host (bind mounts) then create the folders before starting the containers. Otherwise docker will create the folders as user root which often causes permission problems.
