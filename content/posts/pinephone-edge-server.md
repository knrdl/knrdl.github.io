---
title: "PinePhone as edge server in a remote network"
date: "2024-12-20"
tags: [pinephone, ssh, docker, wireguard]
lang: "en"
---

The straight forward way to access a remote network is to setup a VPN connection with the router. If that's not possible, a separate device is required to act as VPN client. A smartphone is particularly suitable for this as it is always on, small, has a low power consumption and makes it easy to configure a wifi connection with the network. The PinePhone is especially suitable for the job as it can host a standard Linux environment which is not messed up like Android. On the PinePhone we need very few tools to work in the remote network:
1. **Wireguard** for the VPN tunnel
2. **SSH** server
3. **Docker** runtime, e.g. to run a time series database to collect sensor stats 

# Setup

1. flash PostmarketOS to a microSD card
2. run the wizard on the phone (to resize the root partition)
3. copy the PostmarketOS img-file to the microSD card
4. flash the img-file to the eMMC via `dd`: https://pine64.org/documentation/PinePhone/Installation/Installation_to_the_eMMC/#from-the-booted-microsd-os
5. remove the sd card and reboot
6. login (user:147147) and connect wifi

## SSH

1. start ssh server: `sudo service sshd start`
2. enable ssh on boot: `sudo rc-update add sshd`
3. connect to the phone: `ssh user@pine64-pinephone`
4. in `/etc/ssh/sshd_config`: set `AllowTcpForwarding yes` to forward ports from the remote network to your local host

## System

1. check updates: `sudo apk update` and `sudo apk upgrade -a`
2. install some tools: `sudo apk add htop curl nano`
3. remove unnecessary apps: `sudo apk del gnome-maps gnome-calculator gnome-software gnome-clocks gnome-calendar gnome-text-editor gnome-contacts gnome-weather  chatty portfolio lollypop firefox-esr evince calls megapixels postmarketos-default-camera postmarketos-welcome loupe flatpak`
4. list remaining packages: `sudo apk list -I`
5. disable unnecessary services: `sudo rc-update del bluetooth` and `sudo rc-update del modemmanager`
6. list enabled services: `sudo rc-update show`

## Wireguard

1. install: `sudo apk add wireguard-tools-wg-quick wireguard-tools-openrc`
2. add wireguard config to `/etc/wireguard/wg0.conf`, e.g.:
```conf
[Interface]
PrivateKey = XXXXXXXXXXXXXXXXXXXXXXXXXXXXX
Address = 192.168.178.204/24
DNS = 192.168.178.1
DNS = fritz.box

[Peer]
PublicKey = XXXXXXXXXXXXXXXXXXXXXXXXXXXXX
PresharedKey = XXXXXXXXXXXXXXXXXXXXXXXXXXXXX
AllowedIPs = 192.168.178.0/24,0.0.0.0/0
Endpoint = XXXXXXXXXXXXXXXXXXXXXXXXXXXXX.myfritz.net:12345
PersistentKeepalive = 25
```
3. fix permissions: `sudo chmod o-r /etc/wireguard/wg0.conf`
4. [autostart](https://wiki.alpinelinux.org/wiki/Configure_a_Wireguard_interface_(wg)):
```
sudo ln -s /etc/init.d/wg-quick /etc/init.d/wg-quick.wg0
sudo rc-update add wg-quick.wg0
sudo service wg-quick.wg0 start
```
5. manual start: `sudo wg-quick up wg0`
6. manual stop: `sudo wg-quick down wg0`

## Docker

1. install: `sudo apk add docker` (alpine sources typically provide an up-to-date version)
2. autostart: `sudo rc-update add docker`
3. start: `sudo service docker start`
4. the PinePhone can execute arm32v7 and also arm64v8 images