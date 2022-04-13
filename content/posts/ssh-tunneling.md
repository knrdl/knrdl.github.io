---
title: "SSH Tunneling"
date: "2022-04-13"
tags: ["ssh", "sftp", "rsync"]
lang: "en"
---

SSH allows tcp port forwarding between ssh client and ssh server as part of the encrypted ssh connection.

# Server → Client (local)

A socket on the server (source) gets forwarded to the client (target).

```shell
ssh user@host -L target_ip:target_port:source_ip:source_port
```

## Use case 1: Admin software

Server socket with port 9000 will be available on client machine at port 5000.

```shell
ssh user@host -L 127.0.0.1:5000:127.0.0.1:9000
```

Useful if admin software is only accessible on the server (localhost binding) and not exposed to the network.

## Use Case 2: Quick demo

Server socket with port 9000 will be accessible at client network on port 8080 of the client machine.

```shell
ssh user@host -L 8080:127.0.0.1:9000
ssh user@host -L 0.0.0.0:8080:127.0.0.1:9000
```

Useful for a quick demo of a service running in a different network (e.g. cloud) to multiple participants in the same
network (e.g. company).

## Use case 3: Jump Host

Server can reach host with ip addr 192.168.1.15. Service on 192.168.1.15:8080 will be available on client machine at
port 8081.

```shell
ssh user@host -L 127.0.0.1:8081:192.168.1.15:8080
```

Hop to hop forwarding of hosts accessible in the server network(s).

# Client → Server (remote)

A socket on the client (source) gets forwarded to the server (target).

```shell
ssh user@host -R target_port:source_ip:source_port
```

## Use case 1: Microservice development

Service on port 8000 of the client machine will be available on server at port 8081. Unless `GatewayPorts` is set
to `yes` in sshd config (default `no`) the serverside bind will always be localhost (127.0.0.1).

```shell
ssh user@host -R 8081:127.0.0.1:8000
```

Useful to integrate a microservice on developer machine into a foreign environment (e.g. test cluster). Also works with
other source addresses than `127.0.0.1`.

## Use case 2: TLS terminating web proxy as a service

Client starts a http server on local machine on port 8000. That service will be available as https://server.tld

```shell
ssh user@host -R 8001:127.0.0.1:8000
```

Processing order:

1. Webbrowser (User Agent) requests https://server.tld
2. Reaches a reverse proxy on port 443
    * does tls termination
    * and proxying, nginx: `proxy_pass 127.0.0.1:8001;`
3. SSH Tunnel
4. local machine port 8000 http server

# SSH Config

port forwarding can also be specified on the client in `~/.ssh/config`:

```text {hl_lines=["5-6"]}
Host server-01
	HostName	192.168.1.2
	Port		22
	User		clusteradm
	LocalForward	localhost:9001 localhost:9000
	RemoteForward	8001 localhost:8000
	KeepAlive	yes
	IdentitiesOnly	yes
	IdentityFile	~/.ssh/server_01_clusteradm
```

Config format is `(Local|Remote)Forward target source`

connect via `$ ssh server-01`

# Persistent connections

Just use the `autossh` command in place of `ssh`. AutoSSH uses heartbeats to check if the connection is still open and
open another one otherwise automatically and fully transparent.

# Dedicated tunneling server

It's possible to run a standalone ssh server which just allows port forwarding and no remote command execution. Setup:

```shell
mkdir -p /jail
adduser --gecos "" --no-create-home --shell /bin/false --disabled-password --uid 1000 sshtunnel
```

Excerpt from `/etc/ssh/sshd_config`:

```shell
# AllowUsers list all users which should be able to login
AllowUsers sshtunnel
Match User sshtunnel
  PermitTTY no
  Banner none
  X11Forwarding no
  AllowAgentForwarding no
  # AllowTcpForwarding: yes (= local+remote), local, remote, no
  AllowTcpForwarding local
  GatewayPorts no
  PermitTunnel no
  ChrootDirectory /jail
  ForceCommand /bin/false
  PermitOpen 127.0.0.1:8730
```

Start sshd in foreground: `$ sshd -D -e`

Connect client: `$ ssh -N -L 127.0.0.1:8730:127.0.0.1:8730 sshtunnel@localhost`

Flag `-N` prevents the spawning of a shell (which would result in a connection abortion otherwise).

# Rsync tunneling (also sftp)

`rsync` can copy (sync) files between hosts. Therefore, it can use ssh as a transfer protocol. But that implies running
the rsync command on the server:

```shell
rsync -av -e ssh user@host:/backup/ /datadir/
```

That won't work with a tunneling only ssh server. Luckily rsync can also be operated with a standalone server, so the
rsync protocol can be tunneled via ssh.

## Server Setup

```shell
adduser --gecos "" --no-create-home --shell /bin/false --disabled-password --uid 1001 rsyncbackup
sudo mkdir -p /jail/backup
sudo chown -R root:root /jail  # user cannot have write permission to chroot dir
sudo chown -R rsyncbackup:rsyncbackup /jail/backup
sudo chmod -R 755 /jail/backup
```

Excerpt from `/etc/ssh/sshd_config`:

```shell
# allow sftp connections
Subsystem sftp internal-sftp
# AllowUsers list all users which should be able to login
AllowUsers rsyncbackup
Match User rsyncbackup
  PermitTTY no
  Banner none
  X11Forwarding no
  AllowAgentForwarding no
  # AllowTcpForwarding: yes (= local+remote), local, remote, no
  AllowTcpForwarding local
  GatewayPorts no
  PermitTunnel no
  ChrootDirectory /jail
  ForceCommand internal-sftp
  PermitOpen 127.0.0.1:8730
```

In addition to rsync this config also allows file browsing via sftp in the `/jail` dir.

Rsync server config in `/etc/rsyncd.conf`:

```shell
use chroot = true
hosts allow = 127.0.0.1/32
port = 8730
timeout = 300
max connections = 2
reverse lookup = no


log file = /dev/stdout
log format = %h %o %f %l %b
pid file = /var/run/rsyncd.pid

[backup_sink]
comment = Backup
path = /jail
read only = no
list = yes
uid = rsyncbackup
gid = rsyncbackup
```

Start rsync server: `$ rsync --daemon --config /etc/rsyncd.conf`

## Client Setup

Record in `.ssh/config`:

```shell
 Host				backup-conn
     HostName		192.168.1.32
     Port			2201
     User			rsyncbackup
     KeepAlive		yes
     LocalForward	127.0.0.1:8730 127.0.0.1:8730
```

Run a rsync job:

```bash
ssh -N backup-conn &  # no tty possible, either use pubkey-auth or use sshpass to submit the password
pid=$!
sleep 3
time rsync bigfile rsync://localhost:8730/backup_sink/backup
kill $pid
```

Check that `/jail/backup/bigfile` has been created on the server.

