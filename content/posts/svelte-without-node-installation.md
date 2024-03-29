---
title: "Svelte without node.js installation"
date: "2022-04-10"
tags: ['docker', 'svelte', 'nodejs', podman]
lang: "en"
hackerNewsId: ''
---

Docker (or podman) to the rescue!

Each first line shown is the Docker command. And the second line is the corresponding command for Podman.

# Setup project

```bash
sudo docker run --pull always -it --rm --user "$UID:$UID" -v "$PWD:$PWD" -w "$PWD" node:alpine npm init
podman run --pull always -it --rm -v "$PWD:$PWD" -w "$PWD" node:alpine npm init
```

# Development

Svelte has replaced rollup with vite as build tool of choice.

## Vite (new)

```bash
sudo docker run --pull always -it --rm --user "$UID:$UID" -v "$PWD:$PWD" -w "$PWD" -p5173:5173 node:alpine npm run dev -- --host
podman run --pull always -it --rm -v "$PWD:$PWD" -w "$PWD" -p5173:5173 node:alpine npm run dev -- --host
```

Open [localhost:5173](http://localhost:5173)

To omit the `-- --host` in the command above, add to your `vite.config.js`:
```javascript
const config = {
    server: {
        host: '0.0.0.0'
    }
}
```

## Rollup (old)

```bash
sudo docker run --pull always -it --rm --user "$UID:$UID" -v "$PWD:$PWD" -w "$PWD" -p8080:8080 -p35729:35729 --env HOST=0.0.0.0 node:alpine npm run dev
podman run --pull always -it --rm -v "$PWD:$PWD" -w "$PWD" -p8080:8080 -p35729:35729 --env HOST=0.0.0.0 node:alpine npm run dev
```

Open [localhost:8080](http://localhost:8080)

> Port 8080 is for the web server (http). Everything is handled by this port, except hot reloading (optional feature). Therefore, port 35729 is additionally used (websocket).
> {.info }

# Custom `.bashrc` shortcut

```bash
alias svelte-npm='sudo docker run --pull always -it --rm --user "$UID:$UID" -v "$PWD:$PWD" -w "$PWD" -p5173:5173 --env HOST=0.0.0.0 node:alpine npm'
alias svelte-npm='podman run --pull always -it --rm -v "$PWD:$PWD" -w "$PWD" -p5173:5173 --env HOST=0.0.0.0 node:alpine npm'
```

Usage: `$ svelte-npm run dev`

## More general approach

```bash
alias docker-dir='sudo docker run --pull always -it --rm --user "$UID:$UID" -v "$PWD:$PWD" -w "$PWD" --env HOST=0.0.0.0'
alias podman-dir='podman run --pull always -it --rm -v "$PWD:$PWD" -w "$PWD" --env HOST=0.0.0.0'
```

Example: `$ docker-dir node:alpine -v`
