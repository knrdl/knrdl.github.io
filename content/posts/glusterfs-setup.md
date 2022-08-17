---
title: "GlusterFs Setup"
date: "2022-04-11"
tags: ["glusterfs", "storage"]
lang: "en"
---

GlusterFs is a distributed storage layer based on
the [rsync algorithm](https://www.andrew.cmu.edu/course/15-749/READINGS/required/cas/tridgell96.pdf)

> GlusterFs works great for semi-static files, but not for databases like sqlite or postgres!
> {.danger }

# Setup

For best performance the traffic between nodes stays unencrypted. Such a glusterfs should only be operated in a dedicated, trustworthy network.

The following setup will create a storage cluster on 3 nodes (node1, node2, node3) as NAS RAID1 (mirroring)

## 1. On each node:

```bash
sudo apt install --no-install-recommends glusterfs-server rpcbind
sudo systemctl start glusterd.service
sudo systemctl enable glusterd.service
sudo mkdir -p /media/storage0/gluster  # where cluster data should be stored
```

## 2. On primary node:

```bash
sudo gluster peer probe node1  # replace node1 with real hostname
sudo gluster peer probe node2
sudo gluster peer probe node3
sudo gluster pool list
sudo gluster volume create vol0 replica 3 node1:/media/storage0/gluster node2:/media/storage0/gluster node3:/media/storage0/gluster force
# force is only necessary if /media/storage0/gluster is on at least one node mounted on the root partition (and not an additional partition / drive)
sudo gluster volume info
sudo gluster volume start vol0
sudo gluster volume set vol0 auth.allow 127.0.0.1  
# allow connections only from localhost (each gluster-node will mount their local storage, access from other hosts in network is prevented)
sudo gluster volume info
```

## 3. On each node:

```bash
sudo mkdir -p /media/gluster0  # where gluster gets mounted
sudo chown -R 1000:1000 /media/gluster0
sudo bash -c 'echo "127.0.0.1:/vol0 /media/gluster0 glusterfs defaults,_netdev 0 0" >> /etc/fstab'
sudo mount -a
```

## 4. Test:

```bash
# one one node: 
echo "hello world" > /media/gluster0/testfile
# on another node:
cat /media/gluster0/testfile
rm /media/gluster0/testfile
```

> Never write a file/dir directly to `/media/storage0/gluster`. GlusterFS won't be able to detect the changes, and they will not be synchronised.
> {.warning }

## 5. On each node:

Add the ip addresses of all nodes to the `/etc/hosts` files to prevent cluster split on DNS outages:

```bash
192.168.123.2 node1
192.168.123.3 node2
192.168.123.4 node3
```
