---
title: "Wireguard: Client/Server - Setup"
date: "2022-09-27"
tags: [wireguard, vpn]
lang: "en"
---

# Approach

Wireguard is a modern VPN protocol. Per default all nodes are treated equally, named peers. Let's build a point-to-point connection:
* Client: Wireguard running, **no** port opened
* Server: Wireguard running, **one** port opened

Wireguard uses UDP packets only. If an incoming request is invalid (wrong encryption key) then it will be dropped silently. Thereby additional security measures like fail2ban are not necessary.

# Setup

## 1. Installation

```shell
sudo apt install wireguard
```

A docker based installation is [possible](https://docs.linuxserver.io/images/docker-wireguard). But as wireguard requires a separate kernel module, there are no isolation benefits.

## 2. Generate keys

```shell
# Client:
wg genkey > client.key
wg pubkey < client.key > client.pub

# Server:
$ wg genkey > server.key
$ wg pubkey < server.key > server.pub
```

## 3. Config files

On the client in `/etc/wireguard/wg0.conf`:
```ini
[Interface]
PrivateKey = XXXXX  # client.key
Address = 192.168.7.2/32
ListenPort = 51822

[Peer]
PublicKey = XXXXX  # server.pub
Endpoint = fqdn:51821  # public fqdn (domain) or static ip addr of the server
AllowedIPs = 192.168.7.1/32
PersistentKeepalive = 25
```

On the server in `/etc/wireguard/wg0.conf`:
```ini
[Interface]
PrivateKey = XXXXX  # server.key
Address = 192.168.7.1/32
ListenPort = 51821

[Peer]
PublicKey = XXXXX  # client.pub
AllowedIPs = 192.168.7.2/32
PersistentKeepalive = 25
```

Wireguard won't send any messages per default if there is no traffic. If both peers provide an `Endpoint` in the config (Server - Server Setup), this works well. But in the presented setup keepalives are required for the client to contact the server. `PersistentKeepalive = 25` will send a ping every 25sec through the tunnel.


## 4. Up and running

`wg-quick` is a convenient wrapper around the `wireguard` and `ip` commands. It allows to configure wireguard tunnels in a declarative fashion. Config files are stored as `/etc/wireguard/IFNAME.conf`. 

Setup autostart on both hosts and check logs after startup:
```shell
sudo systemctl enable --now wg-quick@wg0.service
sudo journalctl -u wg-quick@wg0.service
```

## 5. Test connection

```shell
ping 192.168.7.2  # on the server
ping 192.168.7.1  # on the client
```

## 6. Restart a tunnel

```shell
wg-quick down wg0
wg-quick up wg0
```

# Usage

```shell
# Server: bind a port to the wireguard tunnel address
docker run -it --rm -p 192.168.7.1:80:80 nginx:alpine
# Client: query a connection
curl 192.168.7.1:80
```

# Limitations

This wireguard setup is fairly simple. A tunneled client/server relation is realized. 

However, there is no routing of container traffic through the tunnel in place. This would require additional network config on the server host.  
