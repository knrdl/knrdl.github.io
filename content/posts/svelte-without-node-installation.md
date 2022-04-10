---
title: "Svelte without node.js installation"
date: "2022-04-10"
tags: ['docker', 'svelte', 'nodejs']
lang: "en"
hackerNewsId: ''
---

Docker to the rescue!

## Setup project

```bash
sudo docker run -it --rm --user "$UID:$UID" -v "$PWD:$PWD" -w "$PWD" node:alpine npm init
```

## Development

```bash
sudo docker run -it --rm --user "$UID:$UID" -v "$PWD:$PWD" -w "$PWD" -p8080:8080 -p35729:35729 --env HOST=0.0.0.0 node:alpine npm run dev
```

Open [localhost:8080](http://localhost:8080)

> Port 35729 is live reload websocket (optional)

## Custom `.bashrc` shortcut

```bash
alias svelte-npm='sudo docker run -it --rm --user "$UID:$UID" -v "$PWD:$PWD" -w "$PWD" -p8080:8080 -p35729:35729 --env HOST=0.0.0.0 node:alpine npm'
```

Usage: `$ svelte-npm run dev`

### More general approach

```bash
alias docker-dir='sudo docker run -it --rm --user "$UID:$UID" -v "$PWD:$PWD" -w "$PWD" --env HOST=0.0.0.0'
```
